# KEP-3333: Retroactive default StorageClass assignment

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories (Optional)](#user-stories-optional)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
    - [Story 3 (current behavior)](#story-3-current-behavior)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
    - [Behavior change](#behavior-change)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [GA](#ga)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
  - [New annotation](#new-annotation)
  - [Bind PVCs with <code>nil</code> will wait for default SC](#bind-pvcs-with--will-wait-for-default-sc)
- [Infrastructure Needed (Optional)](#infrastructure-needed-optional)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [X] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [X] (R) KEP approvers have approved the KEP status as `implementable`
- [X] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
    - [X] e2e Tests for all Beta API Operations (endpoints)
    - [X] (R) Ensure GA e2e tests for meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
    - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [X] (R) Graduation criteria is in place
    - [X] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
- [X] (R) Production readiness review completed
- [X] (R) Production readiness review approved
- [X] "Implementation History" section is up-to-date for milestone
- [X] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [X] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

We intend to change behaviour of default storage class assignment to be
retroactive for existing *unbound* persistent volume claims without any storage
class assigned. This changes the existing Kubernetes behaviour slightly, which
is further described in sections below.

## Motivation

When user needs to provision a storage they create a PVC to request a volume. 
A control loop looks for any new PVCs and based on current state of the 
cluster the volume will be provided using one of the following methods:

* Static provisioning - PVC did not specify any storage class and there is
  already an existing PV that can be bound to it.
* Dynamic provisioning - there is no existing PV that could be bound but PVC 
  did specify a storage class or there is exactly one storage class in the 
  cluster marked as default.

Considering the "normal" operation described above there are additional
cases that can be problematic:

1. It’s hard to mark a different SC as the default one. Cluster admin can
   choose between two bad solutions:

    1. Option 1: Cluster has two default SCs for a short time, i.e. admin marks the new
       default SC as default and then marks the old default SC as non-default. When
       there are two default SCs in a cluster, the PVC admission plugin refuses to
       accept new PVCs with `pvc.spec.storageClassName = nil`. Hence, cluster users
       may get errors when creating PVCs at the wrong time. They must know it’s a
       transient error and manually retry later.

    2. Option 2: Cluster has no default SC for a short time, i.e. admin marks the old
       default SC as non-default and then marks the new default SC as default.
       Since there is no default SC for some time, PVCs with
       `pvc.spec.storageClassName = nil` created during this time will not get any
       SC and are Pending forever. Users must be smart enough to delete the PVC and
       re-create it.

2. When users want to change the default SC parameters, they must delete the SC
   and re-create it, Kubernetes API does not allow change in the SC. So there is
   no default SC for some time and second case above applies here too.
   Re-creating the storage class to change parameters can be useful in cases
   there is a quota set for the SC, and since the quota is coupled with SC name
   users can not use Option 1 because second SC would have a different name
   and so the existing quota would not apply to it.

3. Defined ordering during cluster installation. Kubernetes cluster installation
   tools must be currently smart enough to create a default SC before starting
   anything that may create PVCs that need it. If such a tool supports multiple
   cloud providers, storage backends and addons that require storage (such an image
   registry), it may be quite complicated to do the ordering right.

### Goals

* Loosen ordering requirements between PVCs and default SC creation.

### Non-Goals

* Introduce new API for default SCs.

## Proposal

Change PV controller behavior so it will assign a default SC to unbound PVCs
with `pvc.spec.storageClassName=nil` after any storage class is marked as
default.

### User Stories (Optional)

#### Story 1

Admin needs to change the default SC from SC1 to SC2
1. The admin marks the current default SC1 as non-default.
2. Another user creates PVC requesting a default SC, by leaving
   `pvc.spec.storageClassName=nil`. The default SC does not exist at
   this point, therefore the admission plugin leaves the PVC untouched with
   `pvc.spec.storageClassName=nil`.
3. The admin marks SC2 as default.
4. PV controller, when reconciling the PVC, updates
   `pvc.spec.storageClassName=nil` to the new SC2.
5. PV controller uses the new SC2 when binding / provisioning the PVC.

#### Story 2

An installation tool wants to install Kubernetes with a CSI driver providing
a default SC and an application that wants to use it (such as image registry).

1. The installer creates PVC for the image registry first, requesting the
   default storage class by leaving `pvc.spec.storageClassName=nil`.
2. The installer creates a default SC.
3. PV controller, when reconciling the PVC, updates
   `pvc.spec.storageClassName=nil` to the new default SC.
4. PV controller uses the new default SC when binding / provisioning the PVC.


#### Story 3 (current behavior)

User wants to provision a volume and there is one default storage class set by
admin.

1. Admin creates a default storage class.
2. Another user creates PVC requesting a default SC, by leaving
   `pvc.spec.storageClassName=nil`. Since the default SC already exists
   the admission plugin changes the `nil` to a name of the default storage 
   class.

### Notes/Constraints/Caveats (Optional)

#### Behavior change

Currently, when a default SC is not available at PVC creation time, a PVC
requesting a default SC (`pvc.spec.storageClassName=nil`) will keep
`pvc.spec.storageClassName=nil` forever. Such PVC can be bound only to PV with
`pvc.spec.storageClassName=""`.

With the new behavior, the PV controller will keep binding PVCs with
`pvc.spec.storageClassName=nil` to PVs with `pv.spec.storageClassName=""` until
a default SC exists. After admin creates default storage class, the `nil` value
of the PVC will change to this SC name and the PVC will no longer bind to PVs
with `pv.spec.storageClassName=""`.

### Risks and Mitigations

Risk: Users depend on existing Kubernetes behavior.

Mitigation:

* Document the behavior change with a release note.

* Suggest users that the only guaranteed way to bind PVs with 
  `storageClassName=""` is to create PVCs with `storageClassName=""` and not
  `storageClassName=nil`.

## Design Details

This KEP requires changes in:
* kube-controller-manager / its PV controller:
    * `pvc.spec.storageClassName=nil` in PVC must be reconciled to a current
      default SC, if it exists, or after it's created.

* kube-apiserver / PVC update validations:
    * PVC update from  `pvc.spec.storageClassName=nil` to
      `pvc.spec.storageClassName=<storage class name>` is now forbidden, and it
      must be allowed for this KEP to work.
      * Validation will still reject changing `pvc.spec.storageClassName` from any
        other value than `nil`.

### Test Plan

[x] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

Kubernetes already has a good coverage of PV/PVC binding tests in
[test/e2e/storage/persistent_volumes.go](https://github.com/kubernetes/kubernetes/blob/ccfac6d3200f63656575819e7b5976c12c3019a6/test/e2e/storage/persistent_volumes.go)

There are no e2e tests that cover behavior of the default SC presence / absence,
they will be added only with the new behavior.

##### Unit tests

All changes should be only in the packages below, which have enough unit test
coverage, we will add only unit tests for the new or changed code.

- `pkg/controller/volume/persistentvolume`: 2022-06-03 - 79%
- `pkg/apis/core/validation/`: 2022-06-03 - 82%

##### Integration tests

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.

For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html
-->

- test/integration/volume/persistent_volumes_test.go: Has few cases for PV/PVC
  binding.

Tests were extended to include default SC and how it will be applied to
existing PVCs. We use integration tests almost like e2e tests of the new
behavior to be sure that change of the default SC won't affect other tests
running in parallel.

Test: 
https://github.com/kubernetes/kubernetes/blob/dfc9bf0953360201618ad52308ccb34fd8076dff/test/integration/volume/persistent_volumes_test.go#L1043

Testgrid:
https://testgrid.k8s.io/sig-release-master-blocking#integration-master&width=5

##### e2e tests

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.

For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html

We expect no non-infra related flakes in the last month as a GA graduation criteria.
-->

There are no tests that would check how the default SC works, because it would
need to create/delete default SCs, which could affect other tests running in
parallel that may expect that a default SC exists (typically StatefulSet e2e
tests).

We added a `[Disruptive]` `[Serial]` test with the new behavior as
"smoke" test of the new behavior, still, most of the tests will be integration
ones.

Test:
https://github.com/kubernetes/kubernetes/blob/91a9ce28ac2486c50222aeeec1f76e664155d769/test/e2e/storage/pvc_storageclass.go#L62

Test Grid:
https://testgrid.k8s.io/sig-network-gce#gci-gce-alpha-features.

### Graduation Criteria

#### Alpha

- Feature implemented behind a feature flag.
- Initial integration tests completed and enabled.
- Implement and enable e2e tests, visible in Testgrid and linked in KEP.

#### Beta

- Scalability tests with existing framework ([perf-tests](https://github.com/kubernetes/perf-tests/tree/81c96c34e3c1f11c5fe91744ef7ce7bf44c2fe5c/clusterloader2/testing/experimental/storage/pod-startup/volume-types/persistentvolume)).
- Allowing time for feedback (at least 2 releases between beta and GA).
- No conformance tests, since we don't test StorageClasses there.
- Manually test version skew between the API server and KCM. See the expected
  behavior below in Version Skew Strategy.
- Manually test upgrade->downgrade->upgrade path.

#### GA

- No users complaining about the new behavior.

### Upgrade / Downgrade Strategy

No change in cluster upgrade / downgrade process.

But all PVCs with `pvc.spec.storageClassName=nil` that exist in a cluster prior
to update and are not bound will be updated with default storage class name if
there is one.

### Version Skew Strategy

This feature is implemented only in the API server and KCM and controlled by
`RetroactiveDefaultStorageClass` feature gate. Following cases may happen:

| API server | KCM | Behavior                                                                                                                         |
|------------|-----|----------------------------------------------------------------------------------------------------------------------------------|
| off | off | Existing Kubernetes behavior.                                                                                                    |
| on | off| Existing Kubernetes behavior, only users can change `pvc.spec.storageClassName=nil` to a SC name.                                |
| off | on | PV controller may try to change `pvc.spec.storageClassName=nil` to a new default SC name, which will fail on the API server. (*) |
| on | on | New behavior.                                                                                                                    |

*) For this case, we strongly suggest that the feature is enabled in the API
server first and then in KCM. Similarly, the feature should be disabled in KCM
first and then in the API server. This follows generic Kubernetes version skew
support.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- [X] Feature gate (also fill in values in `kep.yaml`)
    - Feature gate name: RetroactiveDefaultStorageClass
    - Components depending on the feature gate: kube-apiserver, 
      kube-controller-manager
- [ ] Other
    - Describe the mechanism:
    - Will enabling / disabling the feature require downtime of the control
      plane?
    - Will enabling / disabling the feature require downtime or reprovisioning
      of a node?

###### Does enabling the feature change any default behavior?

Yes. See "Behavior change" section above for details.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes. It has to be disabled in a reverse order of enabling the feature - 
first disable the feature in KCM then in API server. See "Version Skew 
Strategy" section above for more details.

###### What happens if we reenable the feature if it was previously rolled back?

No issues are expected. The case is exactly the same as when the feature is 
enabled for the first time.

###### Are there any tests for feature enablement/disablement?

There is no new field that needs to be handled in a special way. The feature 
gate just enables/disables a code path in PV controller which is already covered
by existing unit tests.

Validation test:
https://github.com/kubernetes/kubernetes/blob/42458952616406922ea59e6d0b65c35c94444172/pkg/apis/core/validation/validation_test.go#L2291

PV controller test:
https://github.com/kubernetes/kubernetes/blob/42458952616406922ea59e6d0b65c35c94444172/pkg/controller/volume/persistentvolume/pv_controller_test.go#L753

### Rollout, Upgrade and Rollback Planning

###### How can a rollout or rollback fail? Can it impact already running workloads?

In case the feature is not enabled during rollout due to a failure there
should be no impact and the behavior of StorageClass assignment will not change.
If the feature is rolled out partially, so it's enabled only on some API
servers, there will be no impact on running workloads because the request will
be  processed as if the feature is disabled - that means nothing happens and PVC
will not be changed.

###### What specific metrics should inform a rollback?

A KCM metric for failure counts called `retroactive_storageclass_errors_total`
will indicate a problem with the feature.

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

Upgrade and rollback will be tested when the feature gate will change to beta.

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

No.

### Monitoring Requirements

###### How can an operator determine if the feature is in use by workloads?

A counter metric will be present in KCM metric endpoint, it will show total
count of successful and failed retroactive StorageClass assignments.

###### How can someone using this feature know that it is working for their instance?

By inspecting a `retroactive_storageclass_total` metric value. If the counter
is increasing while letting PVCs being updated retroactively with a default 
StorageClass the feature is enabled. And at the same time if
`retroactive_storageclass_errors_total` counter does not increase the feature
works as expected.

<!--
For instance, if this is a pod-related feature, it should be possible to determine if the feature is functioning properly
for each individual pod.
Pick one more of these and delete the rest.
Please describe all items visible to end users below with sufficient detail so that they can verify correct enablement
and operation of this feature.
Recall that end users cannot usually observe component logs or access metrics.
-->

- [ ] Events
    - Event Reason:
- [X] API .spec
    - Condition name:
    - Other field: `pvc.spec.storageClassName` changing from nil to current default StorageClass name after the default is set
- [ ] Other (treat as last resort)
    - Details: metric

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

<!--
This is your opportunity to define what "normal" quality of service looks like
for a feature.

It's impossible to provide comprehensive guidance, but at the very
high level (needs more precise definitions) those may be things like:
  - per-day percentage of API calls finishing with 5XX errors <= 1%
  - 99% percentile over day of absolute value from (job creation time minus expected
    job creation time) for cron job <= 10%
  - 99.9% of /health requests per day finish with 200 code

These goals will help you determine what you need to measure (SLIs) in the next
question.
-->

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

- [X] Metrics
    - Metric name: `retroactive_storageclass_total` and `retroactive_storageclass_errors_total`
    - [Optional] Aggregation method:
    - Components exposing the metric: KCM

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

A metric which would include PersistentVolumeClaim name or StorageClass name 
as a label to help users debug possible issues. Such metric would have
potentially unbounded cardinality, which is a hard blocker for adding it.

<!--
Describe the metrics themselves and the reasons why they weren't added (e.g., cost,
implementation difficulties, etc.).
-->

### Dependencies

<!--
This section must be completed when targeting beta to a release.
-->

###### Does this feature depend on any specific services running in the cluster?

No.

### Scalability

<!--
For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them.

For beta, this section is required: reviewers must answer these questions.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

###### Will enabling / using this feature result in any new API calls?

Yes.

- API call type: PATCH PVC
- estimated throughput: low, only once for PVCs that have
  `pvc.spec.storageClassName=nil`
- originating component(s): kube-controller-manager

###### Will enabling / using this feature result in introducing new API types?

No.

###### Will enabling / using this feature result in any new calls to the cloud provider?

No.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

No.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

For WIP SLI ["Startup latency of schedulable stateful pods"](https://github.com/kubernetes/community/blob/master/sig-scalability/slos/pod_startup_latency.md#definition)
there can be one extra API call in case there is no default SC while creating a
PVC.

With current behavior the PVC state would be stuck in `Pending` state so the 
pod would not be scheduled at all.

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

PV controller already has all the informers it will need for this change to 
be implemented.

### Troubleshooting

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.

The Troubleshooting section currently serves the `Playbook` role. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now, we leave it here.
-->

###### How does this feature react if the API server and/or etcd is unavailable?

API requests are performed by Persistent volume controller periodically. If 
some requests fail the PVC will not get updated with current default storage
class as it would under normal operation and a metric with error count is
increased. PV controller will attempt the PVC update again in the next
periodic sync.

###### What are other known failure modes?

None.

<!--
For each of them, fill in the following information by copying the below template:
  - [Failure mode brief description]
    - Detection: How can it be detected via metrics? Stated another way:
      how can an operator troubleshoot without logging into a master or worker node?
    - Mitigations: What can be done to stop the bleeding, especially for already
      running user workloads?
    - Diagnostics: What are the useful log messages and their required logging
      levels that could help debug the issue?
      Not required until feature graduated to beta.
    - Testing: Are there any tests for failure mode? If not, describe why.
-->

###### What steps should be taken if SLOs are not being met to determine the problem?

The only case that is considered a failure is when a PVC fails to update due
to API server error so users should investigate API server logs to determine
the problem.

## Implementation History

<!--
Major milestones in the lifecycle of a KEP should be tracked in this section.
Major milestones might include:
- the `Summary` and `Motivation` sections being merged, signaling SIG acceptance
- the `Proposal` section being merged, signaling agreement on a proposed design
- the date implementation started
- the first Kubernetes release where an initial version of the KEP was available
- the version of Kubernetes where the KEP graduated to general availability
- when the KEP was retired or superseded
-->
- 1.25: initial version
- 1.26: beta

## Drawbacks

See "Behavior change" section above.

## Alternatives

### New annotation

Keep the existing behavior of `storageclass.kubernetes.io/is-default-class`
annotation as it is, but add a new SC annotation, say
`storageclass.kubernetes.io/default-is-retroactive`, that  will retroactively
mark all existing PVCs with `storageClassName: nil` to the new default SC.

This way, we don't break existing Kubernetes behavior, trading it for a bigger
complexity on user side - they need to use two annotations instead of one.

We assume that more users will want the new behavior introduced in this KEP
than users that want to keep the existing behavior.

### Bind PVCs with `nil` will wait for default SC

Today, if there is no default SC, a PVC with `pvc.spec.storageClassName=nil`
will be bound to PV with `pv.spec.storageClassName=""`.

We could simplify this by changing the behavior so that PVCs with
`pvc.spec.storageClassName=nil` would always bind to default SC or would get SC
assigned retroactively, and still keep part of the existing behavior by binding
PVs with `pv.spec.storageClassName=""` only to PVCs
with `pvc.spec.storageClassName=""`.

This change would affect small group of users but might be still too breaking
for users who rely on these semantics.

## Infrastructure Needed (Optional)

Not needed.
