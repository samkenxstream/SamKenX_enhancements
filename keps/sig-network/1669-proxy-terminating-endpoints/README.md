# KEP-1669: Proxy Terminating Endpoints

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Example: only some endpoints terminating when traffic policy is &quot;Cluster&quot;](#example-only-some-endpoints-terminating-when-traffic-policy-is-cluster)
  - [Example: only some endpoints terminating on a node when traffic policy is &quot;Local&quot;](#example-only-some-endpoints-terminating-on-a-node-when-traffic-policy-is-local)
  - [Example: all endpoints terminating and traffic policy is &quot;Cluster&quot;](#example-all-endpoints-terminating-and-traffic-policy-is-cluster)
  - [Example: all endpoints terminating on a node when traffic policy is &quot;Local&quot;](#example-all-endpoints-terminating-on-a-node-when-traffic-policy-is-local)
  - [Handling terminating endpoints that are not ready.](#handling-terminating-endpoints-that-are-not-ready)
  - [User Stories (optional)](#user-stories-optional)
    - [Story 1](#story-1)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Additions to EndpointSlice](#additions-to-endpointslice)
  - [kube-proxy](#kube-proxy)
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

- [X] Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [X] KEP approvers have approved the KEP status as `implementable`
- [X] Design details are appropriately documented
- [X] Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [X] Graduation criteria is in place
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

This KEP proposes some enhancements to kube-proxy to handle terminating endpoints in an effort to improve the traffic engineering capabilities and overall relability of Kubernetes.
These changes will depend on recent changes to the EndpointSlice API as part of KEP-1672 to include terminating pods in the EndpointSlice API.

## Motivation

Historically, Kubernetes has ignored terminating Pods from both the Endpoints and EndpointSlice API. KEP-1672 recently introduced an API change in EndpointSlice
where terminating endpoints are now included in EndpointSlice, with the addition of two new endpoint conditions "Serving" and "Terminating". Even though the EndpointSlice
API now includes terminating endpoints, kube-proxy strictly forwards traffic to only "Ready" pods that are not terminating. There are several scenarios where not handling
terminating endpoints can lead to traffic loss. It's worth diving into one specific scenario described in [this issue](https://github.com/kubernetes/kubernetes/issues/85643):

When using Service Type=LoadBalancer w/ externalTrafficPolicy=Local, the availability of node backend is determined by the healthCheckNodePort served by kube-proxy.
Kube-proxy returns a "200 OK" http response on this endpoint if there is a local ready endpoint for a Service, otherwise it returns 500 http response signalling to the load balancer that the node should be removed
from the backend pool. Upon performing a rolling update of a Deployment, there can be a small window of time where old pods on a node are terminating (hence not "Ready") but the load balancer
has not probed kube-proxy's healthCheckNodePort yet. In this event, there is traffic loss because the load balancer is routing traffic to a node where the proxy rules will blackhole
the traffic due to a lack of local endpoints. The likihood  of this traffic loss is impacted by two factors: the number of local endpoints on the node and the interval between health checks
from the load balancer. The worse case scenario is a node with 1 local endpoint and a load balancer with a long health check interval.

Currently there are several workarounds that users can leverage:
* Use Kubernetes scheduling/deployment features such that a node would never only have terminating endpoints. For example, always scheduling two pods on a node and only allowing 1 pod to update at a time.
* Reducing the load balancer health check interval to very small time windows. This may not alwyays be possible based on the load balancer implementation.
* Use a preStop hook in the Pod to delay the time between a Pod terminating and the process receiving SIGTERM.

While some of these solutions help, there's more that Kubernetes can do to handle this complexity for users.

### Goals

* Reduce potential traffic loss from kube-proxy that occurs on rolling updates because traffic is sent to Pods that are terminating.

### Non-Goals

* Changing the behavior of how pods terminate.
* Handling terminating endpoints for other consumers of the EndpointSlice API, such as ingress controllers or external load balancers.

## Proposal

This KEP proposes that if all endpoints for a given Service scoped to its traffic policy are terminating (i.e. pod.DeletionTimestamp != nil), then all traffic should be sent to
terminating Pods that are still Ready. Note that the EndpointSlice API introduced a new condition called "Serving" which is semantically equivalent to "Ready" except that the Ready condition
must always be "False" for terminating pods for compatibility reasons. For consumers of the EndpointSLice API that want to route traffic strictly based on a Pod's readiness ignoring
it's terminating state, they should be reading the Serving condition going forward. Below are some examples to help illustrate the proposed behavior:

### Example: only some endpoints terminating when traffic policy is "Cluster"

When the traffic policy is "Cluster" and some endpoints are terminating, all traffic should be routed to the ready endpoints that are not terminating..

### Example: only some endpoints terminating on a node when traffic policy is "Local"

When the traffic policy is "Local" and some endpoints are terminating within a single node, traffic should be routed to ready endpoints on that node that are not terminating.

### Example: all endpoints terminating and traffic policy is "Cluster"

When the traffic policy is "Cluster" and all endpoints are terminating, then traffic should be routed to any terminating endpoint that is ready.

### Example: all endpoints terminating on a node when traffic policy is "Local"

When the traffic policy is "Local" and all endpoints are terminating within a single node, then traffic should be routed to any terminating endpoint that is ready on that node.


### Handling terminating endpoints that are not ready.

It is worth noting that traffic should not be routed to terminating pods if their readiness probe is failing, even if it is the only endpoints remaining. This is to give workloads
the flexibility/control to opt out of this behavior by either exiting immediately or failing the readiness probe when receiving SIGTERM from kubelet. This would also be counter-intuitive
to the current understanding of readiness probes.


### User Stories (optional)

#### Story 1

As a user I would like to do a rolling update of a Deployment fronted by a Service Type=LoadBalancer with ExternalTrafficPolicy=Local.
If a node that has only 1 pod of said deployment goes into the `Terminating` state, all traffic to that node is dropped until either a new pod
comes up or my cloud provider removes the node from the loadbalancer's node pool. Ideally the terminating pod should gracefully handle traffic to this node
until either one of the conditions are satisfied.

### Risks and Mitigations

There are scalability implications to tracking termination state in EndpointSlice. For now we are assuming that the performance trade-offs are worthwhile but
future testing may change this decision. See [KEP 1672](../1672-tracking-terminating-endpoints) for more details.

## Design Details

### Additions to EndpointSlice

This work depends on the `Terminating` condition existing on the EndpointSlice API (see KEP 1672) in order to check the termination state of an endpoint.

### kube-proxy

Updates to kube-proxy when watching EndpointSlice:
* update kube-proxy endpoints info to track terminating endpoints based on endpoint.condition.terminating in EndpointSlice.
* update kube-proxy endpoints info to track endpoint readiness based on endpoint.condition.ready in EndpointSlice
* within the scope of the traffic policy for a Service, iterate the following set of endpoints, picking the first set that has at least 1 ready endpoint:
  * ready endpoints that are not terminating
  * ready endpoints that are terminating

In addition, kube-proxy's node port health check should fail if there are only `Terminating` endpoints, regardless of their readiness in order to:
* remove the node from a loadbalancer's node pool as quickly as possible
* gracefully handle any new connections that arrive before the loadbalancer is able to remove the node
* allow existing connections to gracefully terminate

### Test Plan

[X] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

##### Unit tests

- `pkg/proxy`: `07/2021` - Validating behavior in iptables and ipvs proxier. Also tests feature gate enablement.
- `pkg/proxy`: `03/2022` - All tests updated to cover all traffic policies (not just Local)

Links to added tests:
- https://github.com/kubernetes/kubernetes/blob/d436f5d0b7eb87f78eb31c12466e2591c24eef59/pkg/proxy/iptables/proxier_test.go#L5373
- https://github.com/kubernetes/kubernetes/blob/d436f5d0b7eb87f78eb31c12466e2591c24eef59/pkg/proxy/iptables/proxier_test.go#L6158
- https://github.com/kubernetes/kubernetes/blob/6e9845f766e4d34620835aaa1e5f864211471a50/pkg/proxy/ipvs/proxier_test.go#L4964
- https://github.com/kubernetes/kubernetes/blob/6e9845f766e4d34620835aaa1e5f864211471a50/pkg/proxy/ipvs/proxier_test.go#L5316
- https://github.com/kubernetes/kubernetes/blob/f2e5c16545027fbe04cc33d4ef59cd01de6b9967/pkg/proxy/topology_test.go#L48

##### Integration tests

N/A

##### e2e tests

E2E tests will be added to validate that no traffic is dropped during a rolling update for a Service. E2E tests should cover all permutations of externalTrafficPolicy
and internalTrafficPolicy.

- E2E test validating health check node port behavior: https://github.com/kubernetes/kubernetes/blob/4bc1398c0834a63370952702eef24d5e74c736f6/test/e2e/network/service.go#L2790
- E2E test validating fallback behavior for terminating endpoints when `externalTrafficPolicy: Cluster`: https://github.com/kubernetes/kubernetes/blob/4bc1398c0834a63370952702eef24d5e74c736f6/test/e2e/network/service.go#L3060
- E2E test validating fallback behavior for terminating endpoints when `externalTrafficPolicy: Local`: https://github.com/kubernetes/kubernetes/blob/4bc1398c0834a63370952702eef24d5e74c736f6/test/e2e/network/service.go#L3145
- E2E test validating fallback behaviro for terminating endpoints when `internalTrafficPolicy: Cluster`: https://github.com/kubernetes/kubernetes/blob/4bc1398c0834a63370952702eef24d5e74c736f6/test/e2e/network/service.go#L2889
- E2E test validating fallback behaviro for terminating endpoints when `internalTrafficPolicy: Local`: https://github.com/kubernetes/kubernetes/blob/4bc1398c0834a63370952702eef24d5e74c736f6/test/e2e/network/service.go#L2972
- E2E test performing rolling update with an LB using `externalTrafficiPolicy: Local`: https://github.com/kubernetes/kubernetes/blob/f333e5b4c5a174c63ac8e71f2c187f434ce56b1b/test/e2e/network/loadbalancer.go#L1283-L1299

### Graduation Criteria

#### Alpha

* kube-proxy internally tracks the `terminating` and `serving` condition from EndpointSlice
* kube-proxy falls back to terminating endpoints if and only if they are the only available endpoints.
* feature is only enabled if the feature gate `ProxyTerminatingEndpoints` is on.
* unit tests in kube-proxy (see [Test Plan](#test-plan) section)

#### Beta

* E2E tests are in place, exercising all permutations of internalTrafficPolicy and externalTrafficPolicy (see [Test Plan](#test-plan) section)
* Metrics to publish how many Services/Endpoints are routing traffic to terminating endpoints.
* Manual or automated rollback testing (see [Test Plan](#test-plan) section)

#### GA

* Feedback from users in issue [85643](https://github.com/kubernetes/kubernetes/issues/85643) that enabling ProxyTerminatingEndpoints can achieve zero downtime rolling updates when using `externalTrafficPolicy: Local`
* E2E tests demonstrating that rolling updates can be performed with negligible downtime (ideally 100%, but may not feasible without flakiness) with a Service LoadBalancer when using externalTrafficPolicy: Local

### Upgrade / Downgrade Strategy

Behavioral changes to terminating endpoints will apply when the feature gate is enabled. It is required that the cluster has the EndpointSlice API enabled and
the EndpointSliceTerminatingCondition feature is also enabled. On downgrade, the worse case scenario is that kube-proxy falls back to the existing behavior where it always
excludes terminating endpoints. See [Version Skew Strategy](#version-skew-strategy) below.

### Version Skew Strategy

The worse case version skew scenario is that kube-proxy falls back to the existing behavior today where traffic does not fall back to terminating endpoints.
This would either happen if a version of the control plane was not aware of the additions to EndpointSlice or if the version of kube-proxy did not know to consume the terminating condintion in EndpointSlice.

There's not much risk involved as the worse case scenario is falling back to existing behavior.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- [X] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: ProxyTerminatingEndpoints
  - Components depending on the feature gate: kube-proxy

###### Does enabling the feature change any default behavior?

Yes, when there are only terminating (and ready) endpoints, kube-proxy will route traffic to those endpoints. Before this change, kube-proxy
dropped or disallowed this traffic instead.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes.

###### What happens if we reenable the feature if it was previously rolled back?

kube-proxy will no longer drop traffic if only terminating endpoints are available.

###### Are there any tests for feature enablement/disablement?

Yes, there will be unit tests in kube-proxy with the feature gate enabled and disabled.

### Rollout, Upgrade and Rollback Planning

<!--
This section must be completed when targeting beta to a release.
-->

###### How can a rollout fail? Can it impact already running workloads?

<!--
Try to be as paranoid as possible - e.g., what if some components will restart
mid-rollout?
-->

A rollout can be negatively impacted if workloads are currently dependant on kube-proxy's behavior to never forward traffic to terminating endpoints.
Ideally workloads are configured such that their readiness probes fail when traffic is not desired, but workloads may exist relying on the current behavior.
When the rollout happens, workloads may unexpectedly receive traffic when terminating.


###### What specific metrics should inform a rollback?

`sync_proxy_rules_no_local_endpoints_total` can be used to inform rollback in scenarios where Services are dropping traffic to local endpoints.
If this metric increases dramatically (especially when there are no rollouts happening), it could mean there is a programming error in kube-proxy.
In general, we expect this metric to decrease during roll outs when this feature is enabled since nodes that only have terminating endpoints should
no longer be included in this metric.

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

Upgrade->downgrade->upgrade testing was done manually using the following steps:

Build and run the latest version of Kubernetes using Kind:
```
$ kind build node-image
$ kind create cluster --image kindest/node:latest
...
...
$ kubectl get no
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   21m   v1.26.0-beta.0.88+3cfa2453421710

```

Deploy a webserver. In this test the following Deployment and Service was used:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: agnhost-server
  labels:
    app: agnhost-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: agnhost-server
  template:
    metadata:
      labels:
        app: agnhost-server
    spec:
      containers:
      - name: agnhost
        image: registry.k8s.io/e2e-test-images/agnhost:2.40
        args:
        - serve-hostname
        - --port=80
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: agnhost-server
  labels:
    app: agnhost-server
spec:
  internalTrafficPolicy: Local
  selector:
    app: agnhost-server
  ports:
  - port: 80
    protocol: TCP
```

Before roll back, first verify that the `ProxyTerminatingEndpoint` feature is working. This is accomplished by
scaling down the `agnhost-server` deployment to 0 replicas, and checking that the server still accepts traffic
while it is terminating:

Retrieve the cluster IP:
```
$ kubectl get svc agnhost-server
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
agnhost-server   ClusterIP   10.96.132.199   <none>        80/TCP    6m40s
```

Send a request from inside the kind node container:
```
$ docker exec -ti kind-control-plane bash
root@kind-control-plane:/# curl 10.96.132.199
agnhost-server-6d66cfc94f-q5msk
```

Scale down `agnhost-server` deployment to 0 replicas, check the pod is terminating, and check that the
cluster IP works while the pod is terminating.
```
$ kubectl scale deploy/agnhost-server --replicas=0
deployment.apps/agnhost-server scaled
$ kubectl get po
NAME                              READY   STATUS        RESTARTS   AGE
agnhost-server-6d66cfc94f-x9kcw   1/1     Terminating   0          19s
$ docker exec -ti kind-control-plane bash
root@kind-control-plane:/# curl 10.96.132.199
agnhost-server-6d66cfc94f-x9kcw
```

Rollback the feature by disabling the feature gate in kube-proxy:
```
# edit kube-proxy ConfigMap and add `ProxyTerminatingEndpoints: false` to `featureGates` field
$ kubectl -n kube-system edit cm kube-proxy
configmap/kube-proxy edited
# restart kube-proxy
$ kubectl -n kube-system delete po -l k8s-app=kube-proxy
pod "kube-proxy-2ltb8" deleted

```

Verify that traffic cannot be routed to terminating endpoints anymore:
```
$ kubectl scale deploy/agnhost-server --replicas=0
deployment.apps/agnhost-server scaled
$ kubectl get po
NAME                              READY   STATUS        RESTARTS   AGE
agnhost-server-6d66cfc94f-qmftt   1/1     Terminating   0          12s
$ docker exec -ti kind-control-plane bash
root@kind-control-plane:/# curl 10.96.132.199
curl: (7) Failed to connect to 10.96.132.199 port 80 after 0 ms: Connection refused
```

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

No.

### Monitoring Requirements

###### How can an operator determine if the feature is in use by workloads?

In theory, all workloads receiving traffic through kube-proxy will be impacted by this feature when enabled. However, like other existing capabilities,
the traffic can be controlled by workloads through their readiness probes. Operators should assume that workloads passing the readiness probes can now receive traffic
regardless of their termination state. If this is undesired, workloads should be updated such that the readiness probe fails on termination.

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

- [X] Metrics
  - Metric name: `sync_proxy_rules_no_local_endpoints_total`
  - [Optional] Aggregation method:
  - Components exposing the metric:
    - kube-proxy
- [ ] Other (treat as last resort)
  - Details:

###### What are the reasonable SLOs (Service Level Objectives) for the above SLIs?

It is difficult to gauge a reasonable SLO because it could be expected for a cluster to be handling many terminating endpoints at a time
during large rolling updates. Whether those terminating pods should receive traffic is also dependant on the cluster topology and the
the workload characteristics.

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

No.

### Dependencies

<!--
This section must be completed when targeting beta to a release.
-->

###### Does this feature depend on any specific services running in the cluster?

This feature only depends on core components and APIs.

### Scalability

The biggest scalability concern is additional read/writes to the EndpointSlice API for track terminating endpoints. This is covered in more depth
as part of KEP-1672.

###### Will enabling / using this feature result in any new API calls?

No.

###### Will enabling / using this feature result in introducing new API types?

No. New additions to EndpointSlice is covered in KEP-1672.

###### Will enabling / using this feature result in any new calls to the cloud provider?

No.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

No.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

This could impact the existing SLI:

"Latency of programming in-cluster load balancing mechanism (e.g. iptables), measured from when service spec or list of its Ready pods change to when it is reflected in load balancing mechanism, measured as 99th percentile over last 5 minutes aggregated across all programmers."

This is because kube-proxy will be updated to handle terminating endpoints, expanding the total set of endpoints it needs to reconcile.


###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

More CPU/RAM may be consumed by kube-proxy to handle terminating endpoints, however we don't anticipate that it will be significant.

### Troubleshooting

###### How does this feature react if the API server and/or etcd is unavailable?

kube-proxy may forward traffic to an endpoint that has terminated already. However, this scenario
is possible today if apiserver becomes unavailable.

###### What are other known failure modes?

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

- Traffic is sent to terminating endpoints when the user did not want it.
   - Detection: typically by the workload
   - Mitigations: workload should be updated to fail readiness probe on termination.
   - Diagnostics: metrics should indicate that traffic is being forwarded to terminating endpoints.
   - Testing: there are no tests for this failure mode since routing traffic to terminating endpoints based on their readiness is the desired behavior.

###### What steps should be taken if SLOs are not being met to determine the problem?

It is highly recommended that all workloads have a readiness probe configured and handles termination signals from the kubelet appropriately.
As a last resort, the EndpointSlice resource and proxy rules from kube-proxy can be examined to determine why traffic may not be routing correctly
to terminating endpoints on a specific node.

## Implementation History

- [x] 2020-04-23: KEP accepted as implementable for v1.19
- [x] 2021-01-21: KEP scope expanded to include both internal and external traffic.
- [x] 1.24: implementation updated to handle all types of traffic policies.

## Drawbacks

* scalability: this KEP (and KEP 1672) would add more writes per endpoint to EndpointSlice as each terminating endpoint adds at least 1 and at
most 2 additional writes - 1 write for marking an endpoint as "terminating" and another if an endpoint changes it's readiness during termination.
* complexity: an additional corner case is added to kube-proxy adding to it's complexity.

## Alternatives

Some users work around this issue today by adding a preStop hook that sleeps for some duration. Though this may work in some scenarios, better handling from kube-proxy
would alleviate the need for this work around altogether.

