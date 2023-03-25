# KEP-2593: Enhanced NodeIPAM to support Discontiguous Cluster CIDR

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Add more pod IPs to the cluster](#add-more-pod-ips-to-the-cluster)
    - [Add nodes with higher or lower capabilities](#add-nodes-with-higher-or-lower-capabilities)
    - [Provision discontiguous ranges](#provision-discontiguous-ranges)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Pre-Requisites](#pre-requisites)
  - [New Resource](#new-resource)
    - [Expected Behavior](#expected-behavior)
    - [Example: Allocations](#example-allocations)
  - [Controller](#controller)
    - [Data Structures](#data-structures)
    - [Dual-Stack Support](#dual-stack-support)
    - [Startup Options](#startup-options)
    - [Startup](#startup)
    - [Processing Queue](#processing-queue)
    - [Event Watching Loops](#event-watching-loops)
      - [Node Added](#node-added)
      - [Node Updated](#node-updated)
      - [Node Deleted](#node-deleted)
      - [ClusterCIDR Added](#clustercidr-added)
      - [ClusterCIDR Updated](#clustercidr-updated)
      - [ClusterCIDR Deleted](#clustercidr-deleted)
  - [kube-controller-manager](#kube-controller-manager)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha to Beta Graduation](#alpha-to-beta-graduation)
    - [Beta to  GA Graduation](#beta-to--ga-graduation)
    - [Make the Controller the new default](#make-the-controller-the-new-default)
    - [Mark the RangeAllocator as deprecated](#mark-the-rangeallocator-as-deprecated)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
    - [Upgrades](#upgrades)
    - [Downgrades](#downgrades)
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
  - [Share Resources with Service API](#share-resources-with-service-api)
    - [Pros](#pros)
    - [Cons](#cons)
  - [Nodes Register CIDR Request](#nodes-register-cidr-request)
    - [Pros](#pros-1)
    - [Cons](#cons-1)
<!-- /toc -->

## Release Signoff Checklist

<!--
For enhancements that make changes to code or processes/procedures in core
Kubernetes—i.e., [kubernetes/kubernetes], we require the following Release
Signoff checklist to be completed.

Check these off as they are completed for the Release Team to track. These
checklist items _must_ be updated for the enhancement to be released.
-->

Items marked with (R) are required *prior to targeting to a milestone /
release*.

-   [X] (R) Enhancement issue in release milestone, which links to KEP dir in
    [kubernetes/enhancements](not the initial KEP PR)
-   [X] (R) KEP approvers have approved the KEP status as `implementable`
-   [X] (R) Design details are appropriately documented
-   [X] (R) Test plan is in place, giving consideration to SIG Architecture and
    SIG Testing input (including test refactors)
-   [X] (R) Graduation criteria is in place
-   [X] (R) Production readiness review completed
-   [X] (R) Production readiness review approved
-   [ ] "Implementation History" section is up-to-date for milestone
-   [ ] User-facing documentation has been created in [kubernetes/website], for
    publication to [kubernetes.io]
-   [ ] Supporting documentation—e.g., additional design documents, links to
    mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

Today, when Kubernetes' NodeIPAM controller allocates IP ranges for podCIDRs for
nodes, it uses a single range allocated to the cluster (cluster CIDR). Each node
gets a range of a fixed size from the overall cluster CIDR. The size is
specified during cluster startup time and cannot be modified later on.

Kubernetes' IPAM capabilities are an optional behavior that comes with
Kubernetes out of the box. It is not required for Kubernetes to function, and
users may use alternate mechanisms.

This proposal enhances how pod CIDRs are allocated for nodes by adding a new
CIDR allocator that can be controlled by a new resource `ClusterCIDR`.
This would enable users to dynamically allocate more IP ranges for pods. The new
functionality would remain optional, and be an enhancement for those using the
built-in IPAM functionality.

## Motivation

Today, IP ranges for podCIDRs for nodes are allocated from a single range
allocated to the cluster (cluster CIDR). Each node gets a range of a fixed size
from the overall cluster CIDR. The size is specified during cluster startup time
and cannot be modified later on. This has multiple disadvantages:

*   There is just one cluster CIDR from which all pod CIDRs are allocated. This
    means that users need to provision the entire IP range up front accounting
    for the largest cluster that may be created. This can waste IP addresses.
*   If a cluster grows beyond expectations, there isn't a simple way to add more
    IP addresses.
*   The cluster CIDR is one large range. It may be difficult to find a
    contiguous block of IP addresses that satisfy the needs of the cluster.
*   Each node gets a fixed size IP range within a cluster. This means that if
    nodes are of different sizes and capacity, users cannot allocate a bigger
    pod range to a given node with larger capacity and a smaller range to nodes
    with lesser capacity. This wastes a lot of IP addresses.

### Goals

*   Support multiple discontiguous IP CIDR blocks for Cluster CIDR
*   Support node affinity of CIDR blocks
*   Extensible to allow different block sizes allocated to nodes
*   Does not require master or controller restart to add/remove ranges for pods.

### Non-Goals

*   Not providing a generalized IPAM API to Kubernetes. We plan to enhance the
    RangeAllocator’s current behavior (give each Node a /XX from the Cluster
    CIDR as its `PodCIDR`)
*   No change to the default behavior of a Kubernetes cluster.
    *   This will be an optional API and can be disabled (as today’s NodeIPAM
        controllers may also be disabled)

## Proposal

This proposal enhances how pod CIDRs are allocated for nodes by adding a new
CIDR allocator that can be controlled by a new resource 'ClusterCIDR'.
This enables users to dynamically allocate more IP ranges for pods. In addition,
it gives users the capability to control what ranges are allocated to specific
nodes as well as the size of the pod CIDR allocated to these nodes.

### User Stories

#### Add more pod IPs to the cluster

A user created a cluster with an initial clusterCIDR value of 10.1.0.0/20. Each
node is assigned a /24 pod CIDR so the user could create a maximum of 16 nodes.
However, the cluster needs to be expanded but the user does not have enough IPs
for pods.

With this enhancement, the user can now allocate an additional CIDR for pods;
eg. 10.2.0.0/20 with the same configuration to allocate a /24 pod CIDR. This
way, the cluster can now grow by an additional 16 nodes.

#### Add nodes with higher or lower capabilities

A user created a cluster with an ample sized cluster CIDR. All the initial nodes
are of uniform capacity capable of running a maximum of 256 pods and they are
each assigned a /24 pod CIDR. The user is planning to add more nodes to the
system which are capable of running 500 pods. However, they cannot take
advantage of the additional capacity because all nodes are assigned a /24 pod
CIDR. With this enhancement the user configures a new allocation which uses the
original cluster CIDR but allocates a /23 instead of a /24 to each node. They
use the node selector to allocate these IPs only to the nodes with the higher
capacity.

#### Provision discontiguous ranges

A user wants to create a cluster with 32 nodes each with a capacity to run 256
pods. This means that each node needs a /24 pod CIDR range and they need a total
range of /19. However, there aren't enough contiguous IPs in the user's network.
They can find 4 free ranges of size /21 but no single contiguous /19 range.

Using this enhancement, the user creates 4 different CIDR configurations each
with a /21 range. The CIDR allocator allocates a /24 range from any of these /21
ranges to the nodes and the user can now create the cluster.

### Notes/Constraints/Caveats

This feature does not expand the ability of the NodeIPAM controller to change
the `Node.Spec.PodCIDRs` field. Once that field is set, either by the controller
or a third party, it will be treated as immutable. This is particularly relevant
in situtaitons where users start modifying or deleting the `ClusterCIDR`.
Under no circumstances will the controller attempt to revoke the allocated
CIDRs (more details on this are discussed below).

### Risks and Mitigations

-   Racing kube-controller-managers. If multiples of the controller are running
    (as in a HA control plane), how do they coordinate?
    -   The controllers will coordinate using the existing
        kube-controller-manager leader election.

## Design Details

### Pre-Requisites

This KEP assumes that the only consumer of the `--cluster-cidr` value is the
NodeIPAM controller. [KEP
2450](https://github.com/kubernetes/enhancements/tree/master/keps/sig-network/2450-Remove-knowledge-of-pod-cluster-CIDR-from-iptables-rules)
proposed modifications to the kube-proxy to remove it's dependence on a
monolithic ClusterCIDR.  The kube-proxy flag `--detect-local-mode` must be set
to `NodeCIDR` to properly handle nodes having discontiguous Pod CIDRs.

Users not using kube-proxy must ensure that any components they have installed
do not assume Kubernetes has a single continguous Pod CIDR.

### New Resource

This KEP proposes adding a new built-in API called `ClusterCIDR`.

```go
type ClusterCIDR struct {
  metav1.TypeMeta
  metav1.ObjectMeta

  Spec ClusterCIDRSpec
  Status ClusterCIDRStatus
}

type ClusterCIDRSpec struct {
    // This defines which nodes the config is applicable to. A nil selector can
    // be applied to any node.
    // +optional
    NodeSelector *v1.NodeSelector

    // PerNodeHostBits defines the number of host bits to be configured per node.
    // A subnet mask determines how much of the address is used for network bits
    // and host bits. For example an IPv4 address of 192.168.0.0/24, splits the
    // address into 24 bits for the network portion and 8 bits for the host portion.
    // For a /24 mask for IPv4 or a /120 for IPv6, configure PerNodeHostBits=8
    // This field is immutable.
    // +optional
    PerNodeHostBits int32

    // IPv4CIDR defines an IPv4 IP block in CIDR notation(e.g. "10.0.0.0/8").
    // This field is immutable.
    // +optional
    IPv4CIDR string

    // IPv6CIDR defines an IPv6 IP block in CIDR notation(e.g. "fd12:3456:789a:1::/64").
    // This field is immutable.
    // +optional
    IPv6CIDR string
}

type ClusterCIDRStatus struct {
}
```

#### Expected Behavior

- `NodeSelector`, `PerNodeHostBits`, `IPv4CIDR`, and `IPv6CIDR` are immutable after creation.

- `PerNodeHostBits` is used to calculate the mask size PerNode for the specified CIDRs:

    ```IPv4CIDR.PerNodeMaskSize = 32 - PerNodeHostBits```

    ```IPv6CIDR.PerNodeMaskSize = 128 - PerNodeHostBits```

- Each node will be assigned all Pod CIDRs from a matching config. That is to
    say, you cannot assign only IPv4 addresses from a `ClusterCIDR` which
    specifies both IPv4 and IPv6. Consider the following example:

    ```go
    {
        PerNodeHostBits:     10,
        IPv4CIDR:            "10.0.0.0/20",	
        IPv6CIDR:            "fd12:3456:789a:1::/64",
    }
    ```
    Only 4  nodes may be allocated from this `ClusterCIDR` as only 4 IPv4
    Pod CIDRs can be partitioned from the IPv4 CIDR. The remaining IPv6 Pod
    CIDRs may be used if referenced in another `ClusterCIDR`.

- When there are multiple `ClusterCIDR` resources in the cluster, first
    collect the list of applicable `ClusterCIDR`. A `ClusterCIDR` is
    applicable if its `NodeSelector` matches the `Node` being allocated, and if
    it has free CIDRs to allocate.

    A nil `NodeSelector` functions as a default that applies to all nodes. This
    should be the fall-back and not take precedence if any other range matches.
    If there are multiple default ranges, ties are broken using the scheme
    outlined below.

    In the case of multiple matching ranges, attempt to break ties with the
    following rules:
    1.  Pick the `ClusterCIDR` whose `NodeSelector` matches the most
        labels/fields on the `Node`. For example,
        `{'node.kubernetes.io/instance-type': 'medium', 'rack': 'rack1'}` before
        `{'node.kubernetes.io/instance-type': 'medium'}`.
    1.  Pick the `ClusterCIDR` with the fewest Pod CIDRs allocatable. For
        example, `{CIDR: "10.0.0.0/16", PerNodeHostBits: "16"}` (1 possible Pod
        CIDR) is picked before `{CIDR: "192.168.0.0/20", PerNodeHostBits: "10"}`
        (4 possible Pod CIDRs)
    1.  Pick the `ClusterCIDR` whose `PerNodeHostBits` is the fewest IPs.
        For example, `5` (32 IPs) picked before `7` (128 IPs).
    1.  Break ties arbitrarily.

- When breaking ties between matching `ClusterCIDR`, if the most
    applicable (as defined by the tie-break rules) has no more free allocations,
    attempt to allocate from the next highest matching `ClusterCIDR`. For
    example consider a node with the labels:
    ```go
    {
        "node": "n1",
        "rack": "rack1",
    }
    ```
    If the following `ClusterCIDR` are programmed on the cluster, evaluate
    them from first to last using the first config with allocatable CIDRs. In
    the example below, the `ClusterCIDR` have already been sorted according
    to the tie-break rules.
    ```go
    {
        NodeSelector:        { MatchExpressions: { "node": "n1", "rack": "rack1" } },
        PerNodeHostBits:     6,
        IPv4CIDR:            "10.5.0.0/16",
				
    },
    {
        NodeSelector: { MatchExpressions: { "node": "n1" } },
        PerNodeHostBits:      4,
        IPv4CIDR:            "192.168.64.0/20",
    },
    {
        NodeSelector: { MatchExpressions: { "node": "n1" } },
        PerNodeHostBits:      4,
        IPv4CIDR:            "192.168.128.0/17",
    },
    {
        NodeSelector: nil,
        PerNodeHostBits:      6,
        IPv4CIDR:            "10.0.0.0/8",
    }
    ```

- The controller will add a finalizer to the `ClusterCIDR` object
    when it is created.

- On deletion of the `ClusterCIDR`, the controller checks to see if any
    Nodes are using `PodCIDRs` from this range -- if so it keeps the finalizer
    in place and waits for the Nodes to be deleted. When all Nodes using this
    `ClusterCIDR` are deleted, the finalizer is removed.

#### Example: Allocations

```go
[
    {
        // Default for nodes not matching any other rule
        NodeSelector: nil,
        PerNodeHostBits:      8,
        // For existing clusters this is the same as ClusterCIDR
        IPv4CIDR:            "10.0.0.0/8",
    },
    {
        // Another range, also allocate-able to any node
        NodeSelector: nil, 
        PerNodeHostBits:      8,
        IPv4CIDR:            "172.16.0.0/14",
    },
    {
        NodeSelector: { "node": "n1" },
        PerNodeHostBits:      6,
        IPv4CIDR:            "10.0.0.0/8",
    },
    {
        NodeSelector: { "node": "n2" },
        PerNodeHostBits:      6,
        IPv4CIDR:            "192.168.0.0/16",
    },
    {
        NodeSelector: { "node": "n3" },
        PerNodeHostBits:      6,
        IPv4CIDR:            "5.2.0.0/16",
        IPv6CIDR:            "fd12:3456:789a:1::/64",
    },
  ...
]
```

Given the above config, a valid potential configuration might be:

```
{"node": "n1"} --> "10.0.0.0/26"
{"node": "n2"} --> "192.16.0.0/26"
{"node": "n3"} --> "5.2.0.0/20", "fd12:3456:789a:1::/122"
{"node": "n4"} --> "172.16.0.0/24"
```

### Controller

Implement a new
[NodeIPAM controller](https://github.com/kubernetes/kubernetes/tree/master/pkg/controller/nodeipam)
The controller will set up watchers on the `ClusterCIDR` objects and the
`Node` objects.

This controller relies on being a single writer (just as the current NodeIPAM
controller does as well). In the case of HA control planes with multiple
replicas, there will have to be some form of leader election to enforce only 1
active leader. This KEP proposes re-using the kube-controller-manager leader
election to pick a active controller.

#### Data Structures

We will use maps to store the allocated ranges and which node is using the
range. Because the number of nodes is expected to be on the order of thousands,
more sophisticated data structures are likely not required.

Prior investgations [here](https://github.com/kubernetes/kubernetes/pull/90184)
suggest that maps storing allocations will perform well under the number of
nodes we expect.

#### Dual-Stack Support

The decision of whether to assign only IPv4, only IPv6, or both depends on the
CIDRs configured in a `ClusterCIDR` object. As described
[above](#expected-behavior), the controller creates an ordered list of
`ClusterCIDR` resources which apply to a given `Node` and allocates from
the first matching `ClusterCIDR` with CIDRs available.

The controller makes no guarantees that all Nodes are single-stack or that all
Nodes are dual-stack. This is to specifically allow users to upgrade existing
clusters.

#### Startup Options

The following startup options will be supported (via the
kube-controller-manager). They are optional, and intended to support migrating
from the existing NodeIPAM controller:
-   `serviceCIDRs` : In some situations, users have Service CIDRs which
    overlap with their Pod CIDR space. The controller will not allocate any IPs
    which fall within the provided Service CIDRs.
    
    Currently, this is specified to the kube-controller-manager by the
    `--service-cluster-ip-range` flag.
-   `clusterCIDR` : Users can specify to Kubernetes which CIDR to use for Pod
    IPs. This is a widely read configuration specified by the
    `--cluster-cidr` flag.
-   `nodeCIDRMaskSize` (in single-stack IPv4) : Defines the size of the per-node
    mask in the single-stack IPv4 case.

    Currently this is specified to the kube-controller-manager by the
    `--node-cidr-mask-size` flag.
-   `nodeCIDRMaskSizeIPv4` and `nodeCIDRMaskSizeIPv6` (in dual-stack mode):
    Defines the size of the per-node masks for IPv4 and IPv6 respectively.

    Currently this is specified to the kube-controller-manager by the
    `--node-cidr-mask-size-ipv4` and `--node-cidr-mask-size-ipv6` flags.

#### Startup

-   Fetch list of `ClusterCIDR` and build internal data structure
-   If they are set, read the `--cluster-cidr` and `--node-cidr-mask-size` flags
    and attempt to create `ClusterCIDR` with the name
    "created-from-flags-\<hash\>".
    -   In the dual-stack case, the flags `--node-cidr-mask-size-ipv4` and
        `--node-cidr-mask-size-ipv6` are used instead, they will also be used as
        necessary.
    -   The "created-from-flags-\<hash\>" object will always be created as long
        as the flags are set. The hash is arbitrarily assigned.
    -   If an un-deleted object with the name "created-from-flags-*" already
        exists, but it does not match the flag values, the controller will
        delete it and create a new object. The controller will ensure (on
        startup) that there is only one non-deleted `ClusterCIDR` with the
        name "create-from-flags-\<hash>". The "\<hash>" at the end of the name
        allows the controller to have multiple "created-from-flags" objects
        present (e.g. blocked on deletion because of the finalizer), without
        blocking startup.
    -   If some `Node`s were allocated Pod CIDRs from the old
        "created-from-flags-\<hash>" object, they will follow the standard
        lifecycle for deleting a `ClusterCIDR` object. The
        "created-from-flag-\<hash>" object the `Nodes` are allocated from will
        remain pending deletion (waiting for its finalizer to be cleared) until
        all `Nodes` using those ranges are re-created.
-   Fetch list of `Node`s. Check each node for `PodCIDRs`
    -   If `PodCIDR` is set, mark the allocation in the internal data structure
        and store this association with the node.
    -   If `PodCIDR` is set, but is not part of one of the tracked
        `ClusterCIDR`, emit a K8s event but do nothing.
    -   If `PodCIDR` is not set, save Node for allocation in the next step.
        After processing all nodes, allocate ranges to any nodes without Pod
        CIDR(s) [Same logic as Node Added event]

#### Processing Queue

The controller will maintain a queue of events that it is processing. `Node`
additions and `ClusterCIDR` additions will be appended to the queue.
Similarly, Node allocations which failed due to insufficient CIDRs can be
retried by adding them back on to the queue (with exponential backoff).

#### Event Watching Loops

##### Node Added

If the Node already has a `PodCIDR` allocated, mark the CIDRs as used.

Otherwise, go through the list of `ClusterCIDR`s and find ranges matching
the node selector from each family. Attempt to allocate Pod CIDR(s) with the
given per-node size. If that `ClusterCIDR` cannot fit a node, search for
another `ClusterCIDR`.

If no `ClusterCIDR` matches the node, or if all matching
`ClusterCIDR`s are full, raise a K8s event and put the Node on the
reconciliation queue (infinite retries). Upon successfully allocating CIDR(s),
update the node object with the podCIDRs.

##### Node Updated

Check that its Pod CIDR(s) match internal allocation.

-   If node.spec.PodCIDRs is already filled up, honor that allocation and mark
    those ranges as allocated.
-   If the node.spec.PodCIDRs is filled with a CIDR not from any
    `ClusterCIDR`, raise a K8sEvent.
-   If the ranges are already marked as allocated for some other node, raise
    another error event (there isn’t an obvious reconciliation step the
    controller can take unilaterally).

##### Node Deleted

Release said Node’s allocation from the internal data-structure.

If this Node is the last one using a particular `ClusterCIDR` that has
been slated for deletion, trigger the deletion flow again (so that the finalizer
is removed and internal data structures are cleaned up).

##### ClusterCIDR Added

Install a finalizer on the `ClusterCIDR` called "networking.kubernetes.io/cluster-cidr-config-finalizer".

Update internal representation of CIDRs to include the new range. Every failed
Node Allocation is stored in a queue, that will be tried again with the new
range by the reconciliation loop.

##### ClusterCIDR Updated

_`NodeSelector`, `IPv4`, and `IPv6` are immutable so any updates should be
rejected_

##### ClusterCIDR Deleted

1.  Update internal data structures to mark the range as terminating (so new
    nodes won't be added to it)
1.  Search the internal representation of the CIDR range to see if any Nodes are
    using the range.
    1.  If there are no nodes using the range, remove the finalizer and cleanup
        all internal state.
    1.  If there are nodes using the range, wait for them to be deleted before
        removing the finalizer and cleaning up. 

### kube-controller-manager

The flag `--cidr-allocator-type` will be amended to include a new type
"MultiCIDRRangeAllocator".

The list of current valid types is
[here](https://github.com/kubernetes/kubernetes/blob/1ff18a9c43f59ffed3b2d266b31e0d696d04eaff/pkg/controller/nodeipam/ipam/cidr_allocator.go#L38).

### Test Plan

[X] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

N/A

##### Unit tests

- Ensure that the controller scales to ~5,000 nodes -- memory usage and
    reasonable allocation times. 
- k8s.io/kubernetes/pkg/controller/nodeipam: 06/21/2022 - 51.2

##### Integration tests

- `TestIPAMMultiCIDRRangeAllocatorCIDRAllocate`: https://github.com/kubernetes/kubernetes/blob/b7ad17978eaba508c560f81bab2fb9d0b256e469/test/integration/clustercidr/ipam_test.go#L44
- `TestIPAMMultiCIDRRangeAllocatorCIDRRelease`: https://github.com/kubernetes/kubernetes/blob/b7ad17978eaba508c560f81bab2fb9d0b256e469/test/integration/clustercidr/ipam_test.go#L131
- `TestIPAMMultiCIDRRangeAllocatorClusterCIDRDelete`: https://github.com/kubernetes/kubernetes/blob/b7ad17978eaba508c560f81bab2fb9d0b256e469/test/integration/clustercidr/ipam_test.go#L208
- `TestIPAMMultiCIDRRangeAllocatorClusterCIDRTerminate`: https://github.com/kubernetes/kubernetes/blob/b7ad17978eaba508c560f81bab2fb9d0b256e469/test/integration/clustercidr/ipam_test.go#L304
- `TestIPAMMultiCIDRRangeAllocatorClusterCIDRTieBreak`: https://github.com/kubernetes/kubernetes/blob/b7ad17978eaba508c560f81bab2fb9d0b256e469/test/integration/clustercidr/ipam_test.go#L389

##### e2e tests

-   Run through some sample workflows. Just a few for example:
    -   Adding a node
    -   Adding a ClusterCIDR
    -   Deleting a ClusterCIDR that is in use
-   Run through the [user stories](#user-stories):
    -   Expand the ClusterCIDR (existing nodes without allocations are
        allocated and new nodes also get ranges.
    -   Use `NodeSelector` to allocate different sized CIDRs to different nodes.
    -   Create and use discontiguous ranges.

### Graduation Criteria

#### Alpha to Beta Graduation

-   Gather feedback from users about any issues
-   Tests are in testgrid

#### Beta to  GA Graduation

-   Wait for 1 release to receive any additional feedback

#### Make the Controller the new default

After the GA graduation, change the default NodeIPAM allocator from
RangeAllocator to MultiCIDRRangeAllocator. This will involve changing the
default value of the flag on the kube-controller-manager
(`--cidr-allocator-type`).

#### Mark the RangeAllocator as deprecated

In the same release that the MultiCIDRRangeAllocator is made the default,
mark the RangeAllocator as deprecated.

After 2 releases, the code can be removed.

### Upgrade / Downgrade Strategy

#### Upgrades

There is no change to the defaults as part of the alpha, so existing clusters
will upgrade seemlessly.

If we want to use the new controller, users will have to change the
`--cidr-allocator-type` flag on the kube-controller-manager. The new controller
will respect the existing flags for `--cluster-cidr` and
`--node-cidr-mask-size`.

Users will also have to change the kube-proxy flags as outlined in [KEP
2450](https://github.com/kubernetes/enhancements/tree/master/keps/sig-network/2450-Remove-knowledge-of-pod-cluster-CIDR-from-iptables-rules).
The flag `--detect-local-mode` must be set to `NodeCIDR` to properly handle
nodes having discontiguous Pod CIDRs.

#### Downgrades

Users may "downgrade" by switching back the `--cidr-allocator-type` to
"RangeAllocator". If users only use the existing flags (`--cluster-cidr` and
`--node-cidr-mask-size`), then downgrade will be seamless. The Node `PodCIDR`
allocations will persist even after the downgrade, and the old controller can
start allocating PodCIDRs

If users use the `ClusterCIDR` resource to specify CIDRs, switching to the
old controller will maintain any Node `PodCIDR` allocations that have already
been created. Users will have to manually remove the finalizer from the
`ClusterCIDR` objects before they can be deleted.

### Version Skew Strategy

<!--
If applicable, how will the component handle version skew with other
components? What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- Does this enhancement involve coordinating behavior in the control plane and
  in the kubelet? How does an n-2 kubelet without this feature available behave
  when this feature is used?
- Will any other components on the node change? For example, changes to CSI,
  CRI or CNI may require updating that component before the kubelet.
-->

As mentioned in the [pre-requisites](#pre-requisites) section, this feature
depends on certain configurations for the kube-proxy (assuming the kube-proxy is
being used). Those changes were added in release 1.18, so they should be
available for any user who wishes to use this feature.

Besides that, there is no coordination between multiple components required for
this feature. Nodes running older versions (n-2) will be perfectly compatible
with the new controller.

## Production Readiness Review Questionnaire

<!--

Production readiness reviews are intended to ensure that features merging into
Kubernetes are observable, scalable and supportable; can be safely operated in
production environments, and can be disabled or rolled back in the event they
cause increased failures in production. See more in the PRR KEP at
https://git.k8s.io/enhancements/keps/sig-architecture/1194-prod-readiness.

The production readiness review questionnaire must be completed and approved
for the KEP to move to `implementable` status and be included in the release.

In some cases, the questions below should also have answers in `kep.yaml`. This
is to enable automation to verify the presence of the review, and to reduce review
burden and latency.

The KEP must have a approver from the
[`prod-readiness-approvers`](http://git.k8s.io/enhancements/OWNERS_ALIASES)
team. Please reach out on the
[#prod-readiness](https://kubernetes.slack.com/archives/CPNHUMN74) channel if
you need any help or guidance.
-->

### Feature Enablement and Rollback

<!--
This section must be completed when targeting alpha to a release.
-->

###### How can this feature be enabled / disabled in a live cluster?

<!--
Pick one of these and delete the rest.
-->

-   [X] Feature Gate
    -   Feature gate name: MultiCIDRRangeAllocator
    -   Components depending on the feature gate: kube-controller-manager
        -   The feature gate will control whether the new controller can even be
            used, while the kube-controller-manager flag below will pick the
            active controller.
-   [X] Other
    -   Describe the mechanism:
        -   The feature is enabled by setting the kube-controller-manager flag
            `--cidr-allocator-type=MultiCIDRRangeAllocator`.
    -   Will enabling / disabling the feature require downtime of the control
        plane?
        -   Yes. Changing the kube-controller-manager flags will require
            restarting the component (which runs other controllers).
    -   Will enabling / disabling the feature require downtime or reprovisioning
        of a node?
        -   No. With the caveat that if the kube-proxy is in use, it must set
            the appropriate flags, as [described above](#pre-requisites).

###### Does enabling the feature change any default behavior?

No, simply switching to the new controller will not change any behavior. The
controller will continue to respect the old controller's flags.

Only after creating some `ClusterCIDR` objects will behavior change (that
too only for nodes created after that point).

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes, users can switch back to the old controller and delete the
`ClusterCIDR` objects. However, if any Nodes were allocated `PodCIDR` by
the new controller, those allocation will persist for the lifetime of the Node.
Users will have to recreate their Nodes to trigger another `PodCIDR` allocation
(this time performed by the old controller).

There should not be any effect on running workloads. The nodes will continue to
use their allocated `PodCIDR` even if the underlying `ClusterCIDR` object
is forceably deleted.

###### What happens if we reenable the feature if it was previously rolled back?

The controller is expected to read the existing set of `ClusterCIDR` as
well as the existing Node `PodCIDR` allocations and allocate new PodCIDRs
appropriately. 

###### Are there any tests for feature enablement/disablement?

Not yet, they will be added as part of the graduation to beta. They will test
the scenario where some Nodes already have PodCIDRs allocated to them
(potentially from CIDRs not tracked by any `ClusterCIDR`). This should be
sufficient to cover the enablement/disablement scenarios.

### Rollout, Upgrade and Rollback Planning

<!--
This section must be completed when targeting beta to a release.
-->

###### How can a rollout fail? Can it impact already running workloads?

<!--
Try to be as paranoid as possible - e.g., what if some components will restart
mid-rollout?
-->
kube-controller-manager needs to be restarted, the CIDR allocator will be switched 
to the new range allocator based on `--cidr-allocator-type=MultiCIDRRangeAllocator` 
flag. Rollout can fail if the `--cluster-cidr` field is updated during the 
kube-controller-manager restart. kube-controller-manager will crashloop and new
nodes will not have any PodCIDRs assigned and thus will not be ready. This
behavior is consistent with the existing reange-allocator behavior. The mitigation
is to set `--cluster-cidr` back to the original value.

Already running workloads will not be impacted.

###### What specific metrics should inform a rollback?

<!--
What signals should users be paying attention to when the feature is young
that might indicate a serious problem?
-->
multicidrset_allocation_tries_per_request metric must be monitored, if the value
is high(>25) in the buckets greater than 125 would indicate a large number of
failures while allocating the cidrs.

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

<!--
Describe manual testing that was done and the outcomes.
Longer term, we may want to require automated upgrade/rollback tests, but we
are missing a bunch of machinery and tooling and can't do that now.
-->

Upgrade->downgrade->upgrade testing was done manually using the following steps:

Build and run the latest version of Kubernetes using Kind:
```
$ kind build node-image
$ kind create cluster --image kindest/node:latest --config ~/k8sexample/kind/cluster-config-without-multipod.yaml 
...
...
sarveshr@sarveshr:~$ kubectl --context kind-multi-pod-cidr get node
NAME                           STATUS   ROLES           AGE   VERSION
multi-pod-cidr-control-plane   Ready    control-plane   21m   v1.27.0-alpha.1.226+1ded677b2a77a7
```

Initially the feature gate `MultiCIDRRangeAllocator` is set to `false`

```
$ vi cluster-config-without-multipod.yaml 

kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: multi-pod-cidr
featureGates:
  "MultiCIDRRangeAllocator": false
runtimeConfig:
  "api/alpha": "true"
networking:
  apiServerAddress: "127.0.0.1"
  apiServerPort: 6443
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
```

Make sure no ClusterCIDR objects are created

```
$ kubectl --context kind-multi-pod-cidr get cc -A                                                                                              
No resources found    
```

Upgrade to use MultiCIDRRangeAllocator, by logging into the Control plane node
and updating the kubeadm manifests as follows:

1. Update `/etc/kubernetes/manifests/kube-apiserver.yaml` by modifying
`--feature-gates=MultiCIDRRangeAllocator=false` to `--feature-gates=MultiCIDRRangeAllocator=true`
2. Update `/etc/kubernetes/manifests/kube-controller-manager.yaml` by modifying
`--cidr-allocator-type=RangeAllocator` to `--cidr-allocator-type=MultiCIDRRangeAllocator` and
`--feature-gates=MultiCIDRRangeAllocator=false` to `--feature-gates=MultiCIDRRangeAllocator=true`

kubelet will restart the kube-apiserver and kube-controller-manager pods as these are static manifests.

Validate that all pods are Running

```
$ kubectl --context kind-multi-pod-cidr get po -A                                                                                              
NAMESPACE            NAME                                                   READY   STATUS    RESTARTS      AGE                                                   
kube-system          coredns-56f4c55bf9-b7bds                               1/1     Running   0             3m26s                                                 
kube-system          coredns-56f4c55bf9-bgcxk                               1/1     Running   0             3m26s                                                 
kube-system          etcd-multi-pod-cidr-control-plane                      1/1     Running   0             3m38s                                                 
kube-system          kindnet-cj5sg                                          1/1     Running   2 (71s ago)   3m26s                                                 
kube-system          kube-apiserver-multi-pod-cidr-control-plane            1/1     Running   0             72s                                                   
kube-system          kube-controller-manager-multi-pod-cidr-control-plane   1/1     Running   0             19s                                                   
kube-system          kube-proxy-gwh45                                       1/1     Running   0             3m26s
kube-system          kube-scheduler-multi-pod-cidr-control-plane            1/1     Running   1 (95s ago)   3m38s
local-path-storage   local-path-provisioner-c8855d4bb-qxw7g                 1/1     Running   0             3m26s
```

Validate whether the Default ClusterCIDR object is created

```
$ kubectl --context kind-multi-pod-cidr get cc -A
NAME                   PERNODEHOSTBITS   IPV4            IPV6     AGE
default-cluster-cidr   8                 10.244.0.0/16   <none>   21s
```

Rollback to use RangeAllocator, by logging into the Control plane node
and updating the kubeadm manifests as follows:

1. Update `/etc/kubernetes/manifests/kube-apiserver.yaml` by modifying
   `--feature-gates=MultiCIDRRangeAllocator=true` to `--feature-gates=MultiCIDRRangeAllocator=false`
2. Update `/etc/kubernetes/manifests/kube-controller-manager.yaml` by modifying
   `--cidr-allocator-type=MultiCIDRRangeAllocator` to `--cidr-allocator-type=RangeAllocator` and
   `--feature-gates=MultiCIDRRangeAllocator=true` to `--feature-gates=MultiCIDRRangeAllocator=false`

kubelet will restart the kube-apiserver and kube-controller-manager pods as these are static manifests.

Validate that all pods are Running

```
$ kubectl --context kind-multi-pod-cidr get po -A
NAMESPACE            NAME                                                   READY   STATUS    RESTARTS      AGE
kube-system          coredns-56f4c55bf9-b7bds                               1/1     Running   0             16m
kube-system          coredns-56f4c55bf9-bgcxk                               1/1     Running   0             16m
kube-system          etcd-multi-pod-cidr-control-plane                      1/1     Running   0             16m
kube-system          kindnet-cj5sg                                          1/1     Running   4 (24s ago)   16m
kube-system          kube-apiserver-multi-pod-cidr-control-plane            1/1     Running   0             24s
kube-system          kube-controller-manager-multi-pod-cidr-control-plane   1/1     Running   0             11s
kube-system          kube-proxy-gwh45                                       1/1     Running   0             16m
kube-system          kube-scheduler-multi-pod-cidr-control-plane            1/1     Running   2 (49s ago)   16m
local-path-storage   local-path-provisioner-c8855d4bb-qxw7g                 1/1     Running   0             16m
```

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

<!--
Even if applying deprecation policies, they may still surprise some users.
-->
No

### Monitoring Requirements

<!--
This section must be completed when targeting beta to a release.
-->

###### How can an operator determine if the feature is in use by workloads?

<!--
Ideally, this should be a metric. Operations against the Kubernetes API (e.g.,
checking if there are objects with field X set) may be a last resort. Avoid
logs or events for this purpose.
-->

`multicidrset_cidrs_allocations_total` metric with value > 0 will indicate enablement. 
The operator can also check if ClusterCIDR objects are created, this can be done
using `kubectl get cc -A`

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

We have carried-over existing metrics to the new controller: https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/nodeipam/ipam/cidrset/metrics.go#L26-L68

They are:
- multicidrset_cidrs_allocations_total - Count of total number of CIDR allocations
- multicidrset_cidrs_releases_total - Count of total number of CIDR releases
- multicidrset_usage_cidrs - Gauge measuring the percentage of the provided CIDRs
    that have been allocated.
- multicidrset_allocation_tries_per_request - Histogram measuring CIDR allocation
    tries per request.

The health of the service can be determined if the *_total metrics increase with
the number of nodes and the multicidrset_usage_cidrs metric is greater or equal
than the number of nodes.

Note that this service is optional and would allocate the CIDRs only when
`--allocate-node-cidrs` is set to true in kube-controller-manager.

###### What are the reasonable SLOs (Service Level Objectives) for the above SLIs?

<!--
At a high level, this usually will be in the form of "high percentile of SLI
per day <= X". It's impossible to provide comprehensive guidance, but at the very
high level (needs more precise definitions) those may be things like:
  - per-day percentage of API calls finishing with 5XX errors <= 1%
  - 99% percentile over day of absolute value from (job creation time minus expected
    job creation time) for cron job <= 10%
  - 99,9% of /health requests per day finish with 200 code
-->
N/A

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

<!--
Describe the metrics themselves and the reasons why they weren't added (e.g., cost,
implementation difficulties, etc.).
-->
TBD

### Dependencies

<!--
This section must be completed when targeting beta to a release.
-->

###### Does this feature depend on any specific services running in the cluster?

<!--
Think about both cluster-level services (e.g. metrics-server) as well
as node-level agents (e.g. specific version of CRI). Focus on external or
optional services that are needed. For example, if this feature depends on
a cloud provider API, or upon an external software-defined storage or network
control plane.

For each of these, fill in the following—thinking about running existing user workloads
and creating new ones, as well as about cluster-level services (e.g. DNS):
  - [Dependency name]
    - Usage description:
      - Impact of its outage on the feature:
      - Impact of its degraded performance or high-error rates on the feature:
-->
No

### Scalability

<!--
For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them.

For beta, this section is required: reviewers must answer these questions.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

###### Will enabling / using this feature result in any new API calls?

By adding a new resource type, we will increase the number of API calls to watch
the `ClusterCIDR` objects. The new controller, which will replace the
existing NodeIPAM controller, will register a watch for `ClusterCIDR`s

On the write side, the current NodeIPAM controllers already make PATCH calls to
the `Node` objects to add PodCIDR information. That traffic should remain unchanged.

<!--
Describe them, providing:
  - API call type (e.g. PATCH pods)
  - estimated throughput
  - originating component(s) (e.g. Kubelet, Feature-X-controller)
Focusing mostly on:
  - components listing and/or watching resources they didn't before
  - API calls that may be triggered by changes of some Kubernetes resources
    (e.g. update of object X triggers new updates of object Y)
  - periodic API calls to reconcile state (e.g. periodic fetching state,
    heartbeats, leader election, etc.)
-->

###### Will enabling / using this feature result in introducing new API types?
Yes, the new `ClusterCIDR` type will be a pre-requisite for using this
feature.

In the worst case, there may as many `ClusterCIDR` objects as there are
nodes, so we intend to support hundreds of `ClusterCIDR` objects per
cluster. The resources are cluster scoped, not namespace-scoped.

###### Will enabling / using this feature result in any new calls to the cloud provider?

This feature shouldn't result in any direct changes in calls to cloud providers.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

No. Node `PodCIDR` allocations will not change.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

This should not affect any existing SLOs. The only potential impact here is on
Node startup latency -- specifically how long it takes to allocate a `PodCIDR`
for the Node.

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

We expect resource usage of the kube-controller-manager to scale with the number
of nodes and `ClusterCIDRs` in the cluster. Specifically CPU and RAM use
will increase as more nodes and more CIDRs need to be tracked.

We will have unit tests to ensure that such growth is "reasonable" --
proportional to the number of active PodCIDR allocations in the cluster.

### Troubleshooting

<!--
This section must be completed when targeting beta to a release.

The Troubleshooting section currently serves the `Playbook` role. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now, we leave it here.
-->

###### How does this feature react if the API server and/or etcd is unavailable?
MultiCIDRRangeAllocator is a part of the kube-controller-manager and if the
kube-controller-manager is not able to connect to the apiserver, the cluster
might have greater problems.

The MultiCIDRRangeAllocator will not be able to fetch node objects and thus the
node will remain in NotReady state as it will have no PodCIDRs assigned. This is
in line with the current RangeAllocator behavior.

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

###### What steps should be taken if SLOs are not being met to determine the problem?

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

## Drawbacks

<!--
Why should this KEP _not_ be implemented?
-->

## Alternatives

### Share Resources with Service API

There have also been discussions about updating the service API to have multiple
ranges. One proposal is to share a common `ClusterCIDR` resource between
both APIs.

The potential for divergence between Service CIDRs and Pod CIDRs is quite high,
as discussed in the cons section below.

```
ClusterCIDR {
  Type      CIDRType
  CIDR string # Example "10.0.0.0/8" or "fd12:3456:789a:1::/64"
  Selector  v1.LabelSelector # Specifies which Services or Nodes can be
                             # assigned IPs from this block.
  BlockSize string # How large of an IP block to allocate. For services
                   # this would always be "/32". Example "/24"
}

var (
  ServiceCIDR CIDRType = "service"
  ClusterCIDR CIDRType = "cluster"
)
```

#### Pros

-   First-party resource to allow editing of ClusterCIDR or ServiceCIDR without
    cluster restart
-   Single IPAM resource for K8s. Potentially extensible for more use cases down
    the line.

#### Cons

-   Need a strategy for supporting divergence of Service and NodeIPAM APIs in
    the future.
    -   Already BlockSize feels odd, as Service will not make use of it.
-   Any differences in how Service treats an object vs how NodeIPAM treats an
    object are likely to cause confusion.
    -   Enforce API level requirements across multiple unrelated controllers

### Nodes Register CIDR Request

Nodes might register a request for CIDR (as a K8s resource). The NodeIPAM
controllers would watch this resource and attempt to fulfill these requests.

The major goals behind this design is to provide more flexibility in IPAM.
Additionally, it ensures that nodes ask for what they need and users don’t need
to ensure that the `ClusterCIDR` and the Node’s `--max-pods` value are in
alignment.

A major factor in not recommending this strategy is the increased complexity to
Kubernetes’ IPAM model. One of the stated non-goals was that this proposal
doesn’t seek to provide a general IPAM solution or to drastically change how
Kubernetes does IPAM.

```
NodeCIDRRequest {
  NodeName  string # Name of node requesting the CIDR
  RangeSize string # Example "/24"
  CIDR      string # Populated by some IPAM controller. Example: "10.2.0.0/24"
}
```

#### Pros

-   Because the node is registering its request, it can ensure that it is asking
    for enough IPs to cover its `--max-pods` value.
-   Added flexibility to support different IPAM models:
    -   Example: Nodes can request additional Pod IPs on the fly. This can help
        address user requests for centralized IP handling as opposed to
        assigning them as chunks.

#### Cons

-   Requires changes to the kubelet in addition to change to NodeIPAM controller
    -   Kubelet needs to register the requests
-   Potentially more confusing API.
-   _Minor: O(nodes) more objects in etcd. Could be thousands in large
    clusters._
