<!-- **Note:** When your KEP is complete, all of these comment blocks should be
removed.

To get started with this template:

- [ ] **Pick a hosting SIG.** Make sure that the problem space is something the
  SIG is interested in taking up. KEPs should not be checked in without a
  sponsoring SIG.
- [ ] **Create an issue in kubernetes/enhancements** When filing an enhancement
  tracking issue, please make sure to complete all fields in that template. One
  of the fields asks for a link to the KEP. You can leave that blank until this
  KEP is filed, and then go back to the enhancement and add the link.
- [ ] **Make a copy of this template directory.** Copy this template into the
  owning SIG's directory and name it `NNNN-short-descriptive-title`, where
  `NNNN` is the issue number (with no leading-zero padding) assigned to your
  enhancement above.
- [ ] **Fill out as much of the kep.yaml file as you can.** At minimum, you
  should fill in the "Title", "Authors", "Owning-sig", "Status", and
  date-related fields.
- [ ] **Fill out this file as best you can.** At minimum, you should fill in the
  "Summary" and "Motivation" sections. These should be easy if you've
  preflighted the idea of the KEP with the appropriate SIG(s).
- [ ] **Create a PR for this KEP.** Assign it to people in the SIG who are
  sponsoring this process.
- [ ] **Merge early and iterate.** Avoid getting hung up on specific details and
  instead aim to get the goals of the KEP clarified and merged quickly. The best
  way to do this is to just start with the high-level sections and fill out
  details incrementally in subsequent PRs.

Just because a KEP is merged does not mean it is complete or approved. Any KEP
marked as `provisional` is a working document and subject to change. You can
denote sections that are under active debate as follows:

``` <<[UNRESOLVED optional short context or usernames ]>> Stuff that is being
argued. <<[/UNRESOLVED]>> ```

When editing KEPS, aim for tightly-scoped, single-topic PRs to keep discussions
focused. If you disagree with what is already in a document, open a new PR with
suggested changes.

One KEP corresponds to one "feature" or "enhancement" for its whole lifecycle.
You do not need a new KEP to move from beta to GA, for example. If new details
emerge that belong in the KEP, edit the KEP. Once a feature has become
"implemented", major changes should get new KEPs.

The canonical place for the latest set of instructions (and the likely source of
this file) is [here](/keps/NNNN-kep-template/README.md).

**Note:** Any PRs to move a KEP to `implementable`, or significant changes once
it is marked `implementable`, must be approved by each of the KEP approvers. If
none of those approvers are still appropriate, then changes to that list should
be approved by the remaining approvers and/or the owning SIG (or SIG
Architecture for cross-cutting KEPs). -->
# KEP-2896: OpenAPI V3

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
    - [Transparency in the OpenAPI](#transparency-in-the-openapi)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
    - [Future Work](#future-work)
    - [OpenAPI V2 plan](#openapi-v2-plan)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Paths](#paths)
  - [Controllers](#controllers)
    - [OpenAPI Builder](#openapi-builder)
    - [Proto Models &amp; ETags](#proto-models--etags)
    - [Aggregator](#aggregator)
  - [OpenAPI](#openapi)
  - [Version Skew](#version-skew)
  - [OpenAPI V3 Proto](#openapi-v3-proto)
  - [Clients](#clients)
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
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone /
release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in
      [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and
      SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests for meet requirements for [Conformance
        Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [ ] (R) Graduation criteria is in place
  - [ ] (R) [all GA
        Endpoints](https://github.com/kubernetes/community/pull/1806) must be
        hit by [Conformance
        Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for
      publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to
      mailing list discussions/SIG meetings, relevant PRs/issues, release notes

<!-- **Note:** This checklist is iterative and should be reviewed and updated
every time this enhancement is being considered for a milestone. -->

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

This KEP proposes a new endpoint to publish the [OpenAPI v3
specification](https://swagger.io/specification/) for Kubernetes types. This
solves our problem of stripping fields when publishing the OpenAPI v2 and
improves the transparency and accuracy of the published OpenAPI.


## Motivation

Kubernetes resources and types can be described through their OpenAPI
definitions, currently served as OpenAPI v2 at the $cluster/openapi/v2 cluster
endpoint. With the introduction of CRDs, users provide an OpenAPI v3 spec to
describe the resource. Since kubernetes only publishes OpenAPI v2, these
definitions are converted into an OpenAPI v2 definition before being aggregated
and served at the /openapi/v2 endpoint. OpenAPI v3 is more expressive than v2,
and the conversion results in the loss of some information. Some of the type
definitions are just transformed into “accept anything” definitions because of
our lack of ability to perform a better conversion and limitations in kubectl.
Some efforts at improving the OpenAPI (eg: [enum
support](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/2887-openapi-enum-types))
are targeted to bo bundled with OpenAPI v3.

Additionally, since kubernetes aggregate the OpenAPI v2 spec for all specs into
one endpoint, whenever a spec is modified the entire spec is updated and all
clients will need to redownload the spec. Depending on the size and number of
CRDs, this is not an inexpensive operation.

### Goals

- Support publishing and aggregating OpenAPI v3 for all kubernetes types
- Published v3 spec should be a lossless representation of the data and no
  fields should be stripped for both built-in types and CRDs
- Instead of serving the entire OpenAPI spec at one endpoint, separate the spec
  by group-version and only serve the resources required by each group-version

#### Transparency in the OpenAPI

There are a couple of important fields part of the CRD structural schema that
are stripped when we publish v2: `oneOf`, `anyOf`, `nullable`, `default`. We
would like to keep all fields and publish v3 specs without any information loss
for both built-in types and CRDs. Refer to
[discussion](https://github.com/kubernetes/kube-openapi/blob/master/pkg/handler/handler.go#L105)
for more information around exposing defaults.

### Non-Goals

<!-- What is out of scope for this KEP? Listing non-goals helps to focus
discussion and make progress. -->

- Accommodating new OpenAPI v3 fields outside the Schema object that do not
  affect validity of the spec. These are nice to have fields that can be added
  later.
- Consumers of the OpenAPI (eg: Server Side Apply, client-go, kubectl, etc) will
  eventually need to be updated to consume v3. This is outside the scope of this
  KEP.


## Proposal

<!-- This is where we get down to the specifics of what the proposal actually
is. This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation. What is the desired outcome and how do we measure success?. The
"Design Details" section below is for the real nitty-gritty. -->

The proposal is to publish OpenAPI v3 schemas at the
`/openapi/v3/apis/{group}/{version}` endpoint for all resources. Instead of
aggregating everything into one schema, we will publish separate schemas per
resource group-version. The aggregated schema will not be published at the
openapi/v3 endpoint and the endpoint will instead be used for discovery
purposes. For users who still need to retrieve the entire OpenAPI spec (not
recommended), we will offer a client side utility to aggregate all the
endpoints.

### Notes/Constraints/Caveats (Optional)

#### Future Work

After OpenAPI v3 is implemented, we can start working on updating clients to
consume OpenAPI v3. A separate KEP will be published for that.

#### OpenAPI V2 plan

As OpenAPI V3 becomes more and more available and easily accessible,
clients should move to OpenAPI V3 in favor of OpenAPI V2. This should
result in a steady decline in the number of requests sent to the
OpenAPI V2 endpoint. In the short term, we will make OpenAPI V2 make
computations more lazily, deferring the aggregation and marshaling
process to be on demand rather than at a set interval. This is not
directly part of OpenAPI V3's GA plan, but is something we will be
monitoring closely once OpenAPI V3 is GA.

### Risks and Mitigations

Aggregation for OpenAPI v2 already consumes a non-negligible amount of
resources, naively adding in an aggregation for v3 will double the workload when
the OpenAPI needs to be re-aggregated. Splitting by group-version partially
mitigates this problem because when a resource is changed, only a small subset
of groups will need to be reaggregated compared to the current behavior of
re-aggregating everything.

## Design Details

<!-- This section should contain enough information that the specifics of your
change are understandable. This may include API specs (though not always
required) or even code snippets. If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them. -->

### Paths

The root `/openapi/v3` endpoint will contain the list of paths (groups)
available and serve as a discovery endpoint. Clients can then choose the
group(s) to fetch and send additional requests.

/openapi/v3

```json
{
   "Paths" : {
      "api": "/openapi/v3/api?etag=tag",
      "api/v1": "/openapi/v3/api/v1?etag=tag",
      "apis": "/openapi/v3/apis?etag=tag",
      "apis/admissionregistration.k8s.io/v1": "/openapi/v3/apis/admissionregistration.k8s.io/v1?etag=tag",
      "apis/apiextensions.k8s.io/v1": "/openapi/v3/apis/apiextensions.k8s.io/v1?etag=tag",
      "apis/apps/v1": "/openapi/v3/apis/apps/v1?etag=tag",
      ...
   }
}
```

Based on the provided group, clients can then request `openapi/v3/apis/apps/v1`,
`/openapi/v3/apis/networking.k8s.io/v1` and etc. These leaf node specs are self
contained OpenAPI v3 specs and include all referenced types.

The discovery document has the format of a map with the key being the
group-version and value representing the URL of the OpenAPI for the
particular group version. Note that the URL can be constructed by the
client by prepending the group-version name with the `openapi/v3`
prefix. The URL listed here provides a special etag query parameter to
denote the latest etag for the OpenAPI spec for the particular
group-version. The concept of using and changing the query parameter
when a new version of the spec is available is a pattern used
frequently in browser caching known as cache busting. All OpenAPI spec
requests with the `?etag` query parameter will return a response with
`Cache Control: immutable`. This allows clients to cache the OpenAPI
spec when the etag is not changed. The `max-age` will also be set to a
large value that is equivalent to publishing a spec that never
expires.

Support for caching is built into the httpcache library used in
client-go, and no change is needed on the client side to support this
mechanism other than passing the additional query parameter. Passing
the etag as the query parameter allows clients to check the etag in
the root discovery document. Clients can avoid sending an additional
request to fetch an ETag from specific group-versions, and the root
document itself can provide information on group-version changes and
updates. If there is a race and the client passes in an outdated etag
value, the server will send a 301 to redirect the client to the URL
with the latest etag value.

To ensure that the client cache does not grow in an unbounded manner,
the client-go disk cache will be replaced with a cmopactable disk
cache which performs occasional cleanup of old cached files. Refer to
[PR](https://github.com/kubernetes/kubernetes/pull/109701) for more
details.

### Controllers

#### OpenAPI Builder

The OpenAPI builder for CRDs converts the v3 validation schema into a v2 schema.
We will keep the conversion to the v2 schema for backwards compatibility, but
also generate the v3 schema to publish at the new v3 endpoint.

#### Proto Models & ETags

Kubernetes publishes the openapi schema in both JSON and Protobuf. v3 schemas
will include the same mechanisms as v2, publishing a protobuf model of the
schema, as well as the corresponding etags for caching. In order to publish the
schema based on groups, the CRDs will be grouped by group and published only at
the endpoint for their specific group.

#### Aggregator

The aggregator has a mapping of all the APIServices and refreshes the aggregated
spec on an interval. APIService already publish by group-version so their
behavior is unchanged. Because OpenAPI V3 is published by
group-version, the fully aggregated spec is not needed and thus
aggregation can be skipped. No spec downloading will be done by the
aggregator. The aggregator in OpenAPI V3 will act as a proxy
rather than aggregator, proxing group-version requests to downstream
API servers.

### OpenAPI

Swagger 2.0 and OpenAPI 3.0 have a couple of incompatible changes that are not
currently supported by kube-openapi. Definitions are moved to components and the
schema for Paths is slightly modified. A new struct and builder will be created
in kube-openapi to represent the v3 schema to be imported by the necessary
consumers.

CRD structural schemas will be published as is without any value stripping for
v3. Built-in types will generate the v3 OpenAPI definition, and either keep all
fields or strip incompatible fields when building the swagger spec for v3 and v2
respectively.

The OpenAPI handler will have an additional map to handle the publishing of v3.

```go
// Group -> OpenAPIv3 mapping
var v3Schemas map[string]*OpenAPIv3

type OpenAPIv3 struct {
  rwMutex sync.RWMutex
  lastModified time.Time

  spec3Bytes []byte
  spec3Pb []byte
  spec3PbGz []byte

  spec3BytesETag string
  spec3PbETag string
  spec3PbGzETag string
}
```

### Version Skew

There is a potential version skew between old aggregated apiservers and a new
kube-apiserver. All new aggregated apiservers will publish v3 which will be
discovered and published by the aggregator for v3. Old aggregated apiservers
will not publish v3, and the aggregator cannot discover the v3 schema for the
corresponding aggregated apiservers. To make the v2 to v3 transition process
smoother when a version skew exists, the aggregator will download the v2 schema
from aggregated apiservers if they have not upgraded to v3. This will provide
clients with an option to only use the v3 endpoint and they can immediately drop
support for v2. The drawback is that v2 is lossy and converting it to
v3 will provide a lossy v3 schema. This problem will be fixed when aggregated
apiservers upgrade to publishing v3.

### OpenAPI V3 Proto

Kubernetes relies on a "bug" (relaxed constraint) in the gnostic library for OpenAPI v2 where a `$ref` and `description` can coexist in the same object. See [Issue](https://github.com/kubernetes/kubernetes/issues/106387) for more details. This is disallowed per JSON Schema Draft 4 which is the schema version OpenAPI v2 follows. The `$ref` and `description` coexistence is important to Kubernetes because kubectl explain uses it for providing documentation for reference fields.

For instance, a PodSpec object could have the properties:

```
"affinity": {
  "$ref": "#/definitions/io.k8s.api.core.v1.Affinity",
  "description": "If specified, the pod's scheduling constraints"
},
```

kubectl explain uses the description to provide documentation for struct fields that are represented as references in the OpenAPI schema. The description here describes a field/struct's role in the PodSpec object rather than the struct itself. The gnostic library for OpenAPI 3.0 currently disallows the relaxed constraint and removes the description field when the OpenAPI spec is passed through proto. We will work with the gnostic team to update the library to support the same constraint relaxation as in OpenAPI 2.0 to fix the OpenAPI 3.0 protobuf.

Another workaround to this problem could be to wrap reference structures with `allOf`.

Eg:

```
"affinity": {
  "allOf": [{
    "$ref": "#/definitions/io.k8s.api.core.v1.Affinity",
  }],
  "description": "If specified, the pod's scheduling constraints"
},
```

This solves the immediate protobuf issue but adds complexity to the OpenAPI schema.

The final alternative is to upgrade to OpenAPI 3.1 where the new JSON Schema version it is based off of supports fields alongside a `$ref`. However, OpenAPI does not follow semvar and 3.1 is a major upgrade over 3.0 and introduces various backwards incompatible changes. Furthermore, library support is currently lacking (gnostic) and doesn't fully support OpenAPI 3.1. One important backwards incompatible change is the removal of the nullable field and replacing it by changing the type field from a single string to an array of strings.

### Clients

Updating all OpenAPI V2 clients to use OpenAPI V3 is outside the scope
of this KEP and part of future work. However, some key clients will be
updated to use OpenAPI V3 as part of GA. One example is [OpenAPI V3
for kubectl
explain](https://github.com/kubernetes/enhancements/blob/master/keps/sig-cli/3515-kubectl-explain-openapiv3/README.md).
Server Side Apply will also be updated to directly use the OpenAPI V3
schema rather than OpenAPI V2. Finally, the query param verifier in
kubectl will be updated to use OpenAPI V3.

### Test Plan

<!--
**Note:** *Not required until targeted at a release.*
The goal is to ensure that we don't accept encements with inadequate testing.
All code is expected to have adequate tests (eventually with coverage
expectations). Please adhere to the [Kubernetes testing guidelines][testing-guidelines]
when drafting this test plan.
[testing-guidelines]: https://git.k8s.io/community/contributors/devel/sig-testing/testing.md
-->

[x] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this encement.

##### Prerequisite testing updates

<!--
Based on reviewers feedback describe what additional tests need to be added prior
implementing this encement to ensure the encements have also solid foundations.
-->

##### Unit tests

<!--
In principle every added code should have complete unit test coverage, so providing
the exact set of tests will not bring additional value.
However, if complete unit test coverage is not possible, explain the reason of it
together with explanation why this is acceptable.
-->

<!--
Additionally, for Alpha try to enumerate the core package you will be touching
to implement this encement and provide the current unit coverage for those
in the form of:
- <package>: <date> - <current test coverage>
The data can be easily read from:
https://testgrid.k8s.io/sig-testing-canaries#ci-kubernetes-coverage-unit
This can inform certain test coverage improvements that we want to do before
extending the production code to implement this encement.
-->

This feature is primarily implemented in kube-openapi and unit tests, validation, benchmarks and fuzzing are added there. Some packages in k/k will be modified to capture the changes in kube-openapi and unit tests accompany them.

`k8s.io/apiextensions-apiserver/pkg/controller/openapi/builder`
`k8s.io/kube-aggregator/pkg/controllers/openapiv3/aggregator`

##### Integration tests

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the encement.
For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html
-->

Tests in the following directory:

- `test/integration/apiserver/openapi/...`: https://storage.googleapis.com/k8s-triage/index.html?test=TestOpenAPIV3

##### e2e tests

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the encement.
For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html
We expect no non-infra related flakes in the last month as a GA graduation criteria.
-->

Tests will be added to ensure that OpenAPI is present for all group versions.

### Graduation Criteria

#### Alpha

- Feature implemented behind a feature flag
- Initial e2e tests completed and enabled

#### Beta

- Native types are updated to capture capabilities introduced with v3
  - Incorrect OpenAPI polymorphic types (IntOrString, Quantity) are updated to use `anyOf` in OpenAPI V3
- Definition names of native resources are updated to omit their package paths
- Parameters are reused as components
- Aggregated API servers are queried for their v2 endpoint and converted to
  publish v3 if they do not directly publish v3
- Heuristics are used for the OpenAPI v2 to v3 conversion to maximize
  correctness of published spec
- Aggregation for OpenAPI v3 will serve as a proxy to downstream OpenAPI paths

#### GA

- OpenAPI V3 uses the optimized JSON marshaler for increased performance. See [link](https://github.com/kubernetes/kube-openapi/pull/319) for the benefits with OpenAPI V2.
- [Updated OpenAPI V3 Client Interface](https://docs.google.com/document/d/1HmqJH-yyK8WyU8V1y5z-RyMLq9-Xe8JfTTuaNkm9GZ0)
- Direct conversion from OpenAPI V3 to SMD schema for Server Side Apply
- HTTP Disk Cache is replaced with Compactable Disk Cache
- Conformance tests

### Upgrade / Downgrade Strategy

<!-- If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade, in order to maintain previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade, in order to make use of the enhancement?
  -->

- Enabling/disabling the feature gate
- For upgrading, aggregated apiserver images must also be updated to serve
  OpenAPI v3 in order for the aggregator to pick up the spec and publish OpenAPI
  v3. OpenAPI v2 will continue to work regardless of version skew.
- For downgrading, aggregated apiservers do not need to be downgraded as
  everything will revert to using OpenAPI v2 and the OpenAPI v3 endpoint will be
  untouched.

### Version Skew Strategy

<!-- If applicable, how will the component handle version skew with other
components? What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- Does this enhancement involve coordinating behavior in the control plane and
  in the kubelet? How does an n-2 kubelet without this feature available behave
  when this feature is used?
- Will any other components on the node change? For example, changes to CSI, CRI
  or CNI may require updating that component before the kubelet. -->

There is a possible skew between kube-apiserver and old aggregated apiservers.
The affected aggregated apiservers will not publish v3 and but the
kube-apiserver will attempt to query the v2 endpoint and convert it to v3. This
is a lossy conversion, but the v3 will be a complete representation of all the
cluster resources.

## Production Readiness Review Questionnaire

<!--

Production readiness reviews are intended to ensure that features merging into
Kubernetes are observable, scalable and supportable; can be safely operated in
production environments, and can be disabled or rolled back in the event they
cause increased failures in production. See more in the PRR KEP at
https://git.k8s.io/enhancements/keps/sig-architecture/1194-prod-readiness.

The production readiness review questionnaire must be completed and approved for
the KEP to move to `implementable` status and be included in the release.

In some cases, the questions below should also have answers in `kep.yaml`. This
is to enable automation to verify the presence of the review, and to reduce
review burden and latency.

The KEP must have a approver from the
[`prod-readiness-approvers`](http://git.k8s.io/enhancements/OWNERS_ALIASES)
team. Please reach out on the
[#prod-readiness](https://kubernetes.slack.com/archives/CPNHUMN74) channel if
you need any help or guidance. -->

### Feature Enablement and Rollback

<!-- This section must be completed when targeting alpha to a release. -->

###### How can this feature be enabled / disabled in a live cluster?

<!-- Pick one of these and delete the rest. -->

- [x] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: OpenAPIV3
  - Components depending on the feature gate: kube-apiserver
- [ ] Other
  - Describe the mechanism:
  - Will enabling / disabling the feature require downtime of the control plane?
  - Will enabling / disabling the feature require downtime or reprovisioning of
    a node?

###### Does enabling the feature change any default behavior?

<!-- Any change of default behavior may be surprising to users or break existing
automations, so be extremely careful here. -->

No. A new `openapi/v3` endpoint is added but no existing behavior is changed.
All api resources will have both their openapi v2 and v3 published.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

<!-- Describe the consequences on existing workloads (e.g., if this is a runtime
feature, can it break the existing applications?).

NOTE: Also set `disable-supported` to `true` or `false` in `kep.yaml`. -->

Yes, the feature may be disabled by reverting the feature flag.

###### What happens if we reenable the feature if it was previously rolled back?

The feature does not depend on state, and can be disabled/enabled at will.

###### Are there any tests for feature enablement/disablement?

<!-- The e2e framework does not currently support enabling or disabling feature
gates. However, unit tests in each component dealing with managing data, created
with and without the feature, are necessary. At the very least, think about
conversion tests if API types are being modified. -->

n/a.

### Rollout, Upgrade and Rollback Planning

<!-- This section must be completed when targeting beta to a release. -->

###### How can a rollout or rollback fail? Can it impact already running workloads?

<!-- Try to be as paranoid as possible - e.g., what if some components will
restart mid-rollout?

Be sure to consider highly-available clusters, where, for example, feature flags
will be enabled on some API servers and not others during the rollout.
Similarly, consider large clusters and how enablement/disablement will rollout
across nodes. -->

Version skew is discussed in a section above during a rolling control plane upgrade.

###### What specific metrics should inform a rollback?

<!-- What signals should users be paying attention to when the feature is young
that might indicate a serious problem? -->

Non 200 responses from the `openapi/v3` endpoint could indicate a problem. A long response time from the apiserver for OpenAPI requests could also be an indicator of a problem.

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

<!-- Describe manual testing that was done and the outcomes. Longer term, we may
want to require automated upgrade/rollback tests, but we are missing a bunch of
machinery and tooling and can't do that now. -->

n/a

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

<!-- Even if applying deprecation policies, they may still surprise some users.
-->

No.

### Monitoring Requirements

<!-- This section must be completed when targeting beta to a release. -->

###### How can an operator determine if the feature is in use by workloads?

The OpenAPI path `/openapi/v3` is populated. On the metrics side, an OpenAPI V3 specific metric is `crd_openapi_v3_aggregation_duration_seconds`, and should emit data if the feature is enabled.

###### How can someone using this feature know that it is working for their instance?

<!-- For instance, if this is a pod-related feature, it should be possible to
determine if the feature is functioning properly for each individual pod. Pick
one more of these and delete the rest. Please describe all items visible to end
users below with sufficient detail so that they can verify correct enablement
and operation of this feature. Recall that end users cannot usually observe
component logs or access metrics. -->

The `openapi/v3` endpoint will be populated with the list of groups if OpenAPI V3 is enabled.

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

<!-- This is your opportunity to define what "normal" quality of service looks
like for a feature.

It's impossible to provide comprehensive guidance, but at the very high level
(needs more precise definitions) those may be things like:
  - per-day percentage of API calls finishing with 5XX errors <= 1%
  - 99% percentile over day of absolute value from (job creation time minus
    expected job creation time) for cron job <= 10%
  - 99.9% of /health requests per day finish with 200 code

These goals will help you determine what you need to measure (SLIs) in the next
question. -->

This feature should not affect the SLO of any components.

OpenAPI v3 aggregation comes on-top of v2 aggregation => there is additional load. But v3 aggregation is cheap:

- OpenAPI v3 aggregation is much cheaper as every group-version is aggregated independently.
- APIServices (aggregated apiservers) coming and going (due to availability changes) do not lead to aggregation because kube-apiserver only acts as a proxy
- CRD aggregation is structurally trivial (no unification of definition names, which is quadratic in schema sizes) and hence linear in the number of CRDs per group-version.
- CRDs do not "come and go" as aggregated APIs, but it's a one-time per CRD schema change and API server startup operation.

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

<!-- Pick one more of these and delete the rest. -->

A new metric will be added in CRD controller to measure the time taken to aggregate CRD OpenAPI specs.

 - [X] Metrics
  - Metric name: `crd_openapi_v3_aggregation_duration_seconds`
  - Components exposing the metric: kube-apiserver

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

<!-- Describe the metrics themselves and the reasons why they weren't added
(e.g., cost, implementation difficulties, etc.). -->

Not at the moment.

### Dependencies

<!-- This section must be completed when targeting beta to a release. -->

###### Does this feature depend on any specific services running in the cluster?

OpenAPI V3 aggregates from apiservers provided by APIService.

  - APIService
    - OpenAPI V3 fetches the `openapi/v3` endpoint and specs from aggregated API
      - Impact of its outage on the feature: If an APIService is unavailable, the OpenAPI spec of the corresponding APIService will be unavailable but other OpenAPI specs will be unaffected. The resource usage will be better than with OpenAPI V2 because no aggregation is needed when APIServices become unavailable and available.
      - Impact of its degraded performance or high-error rates on the feature: Same as above.

<!-- Think about both cluster-level services (e.g. metrics-server) as well as
node-level agents (e.g. specific version of CRI). Focus on external or optional
services that are needed. For example, if this feature depends on a cloud
provider API, or upon an external software-defined storage or network control
plane.

For each of these, fill in the following—thinking about running existing user
workloads and creating new ones, as well as about cluster-level services (e.g.
DNS):
  - [Dependency name]
    - Usage description:
      - Impact of its outage on the feature:
      - Impact of its degraded performance or high-error rates on the feature:
-->

### Scalability

<!-- For alpha, this section is encouraged: reviewers should consider these
questions and attempt to answer them.

For beta, this section is required: reviewers must answer these questions.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field. -->

###### Will enabling / using this feature result in any new API calls?

<!-- Describe them, providing:
  - API call type (e.g. PATCH pods)
  - estimated throughput
  - originating component(s) (e.g. Kubelet, Feature-X-controller) Focusing
mostly on:
  - components listing and/or watching resources they didn't before
  - API calls that may be triggered by changes of some Kubernetes resources
    (e.g. update of object X triggers new updates of object Y)
  - periodic API calls to reconcile state (e.g. periodic fetching state,
    heartbeats, leader election, etc.) -->

Yes. Get on the `/openapi/v3` endpoint as well as
`/openapi/v3/{group}/{version}` for each API group provided by Kubernetes.

###### Will enabling / using this feature result in introducing new API types?

<!-- Describe them, providing:
  - API type
  - Supported number of objects per cluster
  - Supported number of objects per namespace (for namespace-scoped objects) -->

No.

###### Will enabling / using this feature result in any new calls to the cloud provider?

<!-- Describe them, providing:
  - Which API(s):
  - Estimated increase: -->

No.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

<!-- Describe them, providing:
  - API type(s):
  - Estimated increase in size: (e.g., new annotation of size 32B)
  - Estimated amount of new objects: (e.g., new Object X for every existing Pod)
-->

No.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

<!-- Look at the [existing SLIs/SLOs].

Think about adding additional work or introducing new steps in between (e.g.
need to do X to start a container), etc. Please describe the details.

[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos -->

No

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

<!-- Things to keep in mind include: additional in-memory state, additional
non-trivial computations, excessive access to disks (including increased log
volume), significant amount of data sent and/or received over network, etc. This
through this both in small and large cases, again with respect to the [supported
limits].

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md -->

No. One thing to note is that aggregation for OpenAPI v2 consumes quite a bit of
memory. OpenAPI v3 will avoid aggregating the entire spec and only aggregate (if
necessary) per group/version, decreasing the runtime and memory usage to a
negligible amount.

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

This feature strictly operates on the control plane and has no effect on node resources.

### Troubleshooting

<!-- This section must be completed when targeting beta to a release.

The Troubleshooting section currently serves the `Playbook` role. We may
consider splitting it into a dedicated `Playbook` document (potentially with
some monitoring details). For now, we leave it here. -->

###### How does this feature react if the API server and/or etcd is unavailable?

The feature is part of the API server and will not function if it is unavailable. It does not depend on the availability of etcd.

###### What are other known failure modes?

<!-- For each of them, fill in the following information by copying the below
template:
  - [Failure mode brief description]
    - Detection: How can it be detected via metrics? Stated another way: how can
      an operator troubleshoot without logging into a master or worker node?
    - Mitigations: What can be done to stop the bleeding, especially for already
      running user workloads?
    - Diagnostics: What are the useful log messages and their required logging
      levels that could help debug the issue? Not required until feature
      graduated to beta.
    - Testing: Are there any tests for failure mode? If not, describe why. -->

      - Failure in endpoint
    - Detection: How can it be detected via metrics? Stated another way: how can
      an operator troubleshoot without logging into a master or worker node?

      Lack of 200 status in the OpenAPI endpoint. High latency in apiserver responses

    - Mitigations: What can be done to stop the bleeding, especially for already
      running user workloads?

      OpenAPI V3 can be rolled back and disabled via the feature flag

    - Diagnostics: What are the useful log messages and their required logging
      levels that could help debug the issue? Not required until feature
      graduated to beta.

      Warning and Error logging messages having openapi/swagger/spec keywords.

    - Testing: Are there any tests for failure mode? If not, describe why.

      Tests will be added for failure conditions.


###### What steps should be taken if SLOs are not being met to determine the problem?

## Implementation History

<!-- Major milestones in the lifecycle of a KEP should be tracked in this
section. Major milestones might include:
- the `Summary` and `Motivation` sections being merged, signaling SIG acceptance
- the `Proposal` section being merged, signaling agreement on a proposed design
- the date implementation started
- the first Kubernetes release where an initial version of the KEP was available
- the version of Kubernetes where the KEP graduated to general availability
- when the KEP was retired or superseded -->

## Drawbacks

<!-- Why should this KEP _not_ be implemented? -->

## Alternatives

<!-- What other approaches did you consider, and why did you rule them out?
These do not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable. -->
