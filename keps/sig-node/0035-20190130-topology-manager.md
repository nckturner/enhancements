---
title: Node Topology Manager
authors:
  - "@ConnorDoyle"
  - "@balajismaniam"
  - "@lmdaly"
owning-sig: sig-node
participating-sigs:
  - sig-node
reviewers:
  - "@vikasc"
  - "@derekwaynecarr"
  - "@jeremyeder"
  - "@RenaudWasTaken"
  - "@klueska"
  - "@nolancon"
approvers:
  - "@dawnchen"
  - "@derekwaynecarr"
editor: Louise Daly
creation-date: 2019-01-30
last-updated: 2019-01-30
status: implementable
see-also:
replaces:
superseded-by:
---

# Node Topology Manager

_Authors:_

* @ConnorDoyle - Connor Doyle &lt;connor.p.doyle@intel.com&gt;
* @balajismaniam - Balaji Subramaniam &lt;balaji.subramaniam@intel.com&gt;
* @lmdaly - Louise M. Daly &lt;louise.m.daly@intel.com&gt;

_Reviewers:_
* @klueska - Kevin Klues &lt;kklues@nvidia.com&gt;
* @nolancon - Conor Nolan &lt;conor.nolan@intel.com&gt;

## Table of Contents

<!-- toc -->
- [Overview](#overview)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
  - [User Stories](#user-stories)
- [Proposal](#proposal)
  - [Proposed Changes](#proposed-changes)
    - [New Component: Topology Manager](#new-component-topology-manager)
      - [Policies](#policies)
      - [Computing Preferred Affinity](#computing-preferred-affinity)
      - [New Interfaces](#new-interfaces)
    - [Feature Gate and Kubelet Flags](#feature-gate-and-kubelet-flags)
    - [Changes to Existing Components](#changes-to-existing-components)
- [Graduation Criteria](#graduation-criteria)
  - [Phase 1: Alpha (v1.16) [COMPLETED]](#phase-1-alpha-v116-completed)
  - [Phase 2: Beta (target v1.17)](#phase-2-beta-target-v117)
  - [GA (stable)](#ga-stable)
- [Challenges](#challenges)
- [Limitations](#limitations)
- [Alternatives](#alternatives)
- [References](#references)
<!-- /toc -->

# Overview

An increasing number of systems leverage a combination of CPUs and
hardware accelerators to support latency-critical execution and
high-throughput parallel computation. These include workloads in fields
such as telecommunications, scientific computing, machine learning,
financial services and data analytics. Such hybrid systems comprise a
high performance environment.

In order to extract the best performance, optimizations related to CPU
isolation and memory and device locality are required. However, in
Kubernetes, these optimizations are handled by a disjoint set of
components.

This proposal provides a mechanism to coordinate fine-grained hardware
resource assignments for different components in Kubernetes.

# Motivation

Multiple components in the Kubelet make decisions about system
topology-related assignments:

- CPU manager
  - The CPU manager makes decisions about the set of CPUs a container is
allowed to run on. The only implemented policy as of v1.8 is the static
one, which does not change assignments for the lifetime of a container.
- Device manager
  - The device manager makes concrete device assignments to satisfy
container resource requirements. Generally devices are attached to one
peripheral interconnect. If the device manager and the CPU manager are
misaligned, all communication between the CPU and the device can incur
an additional hop over the processor interconnect fabric.
- Container Network Interface (CNI)
  - NICs including SR-IOV Virtual Functions have affinity to one socket,
with measurable performance ramifications.

*Related Issues:*

- [Hardware topology awareness at node level (including NUMA)][k8s-issue-49964]
- [Discover nodes with NUMA architecture][nfd-issue-84]
- [Support VF interrupt binding to specified CPU][sriov-issue-10]
- [Proposal: CPU Affinity and NUMA Topology Awareness][proposal-affinity]

Note that all of these concerns pertain only to multi-socket systems. Correct
behavior requires that the kernel receive accurate topology information from
the underlying hardware (typically via the SLIT table). See section 5.2.16
and 5.2.17 of the
[ACPI Specification](http://www.acpi.info/DOWNLOADS/ACPIspec50.pdf) for more
information.

## Goals

- Arbitrate preferred NUMA Node affinity for containers based on input from
  CPU Manager and Device Manager.
- Provide an internal interface and pattern to integrate additional
  topology-aware Kubelet components.

## Non-Goals

- _Inter-device connectivity:_ Decide device assignments based on direct
  device interconnects. This issue can be separated from socket
  locality. Inter-device topology can be considered entirely within the
  scope of the Device Manager, after which it can emit possible
  socket affinities. The policy to reach that decision can start simple
  and iterate to include support for arbitrary inter-device graphs.
- _HugePages:_ This proposal assumes that pre-allocated HugePages are
  spread among the available memory nodes in the system. We further assume
  the operating system provides best-effort local page allocation for
  containers (as long as sufficient HugePages are free on the local memory
  node.
- _CNI:_ Changing the Container Networking Interface is out of scope for
  this proposal. However, this design should be extensible enough to
  accommodate network interface locality if the CNI adds support in the
  future. This limitation is potentially mitigated by the possibility to
  use the device plugin API as a stopgap solution for specialized
  networking requirements.

## User Stories

*Story 1: Fast virtualized network functions*

A user asks for a "fast network" and automatically gets all the various
pieces coordinated (hugepages, cpusets, network device) in a preferred NUMA Node
alignment, in most cases this will be the narrrowest possible set of NUMA Node(s).

*Story 2: Accelerated neural network training*

A user asks for an accelerator device and some number of exclusive CPUs
in order to get the best training performance, due to NUMA Node alignment of
the assigned CPUs and devices.

# Proposal

*Main idea: Two phase topology coherence protocol*

Topology affinity is tracked at the container level, similar to devices and
CPU affinity. At pod admission time, a new component called the Topology
Manager collects possible configurations from the Device Manager and the
CPU Manager. The Topology Manager acts as an oracle for local alignment by
those same components when they make concrete resource allocations. We
expect the consulted components to use the inferred QoS class of each
pod in order to prioritize the importance of fulfilling optimal locality.

## Proposed Changes

### New Component: Topology Manager

This proposal is focused on a new component in the Kubelet called the
Topology Manager. The Topology Manager implements the pod admit handler
interface and participates in Kubelet pod admission. When the `Admit()`
function is called, the Topology Manager collects topology hints from other
Kubelet components.

If the hints are not compatible, the Topology Manager may choose to
reject the pod. Behavior in this case depends on a new Kubelet configuration
value to choose the topology policy. The Topology Manager supports four
policies: `none`(default), `best-effort`, `restricted` and `single-numa-node`. 

A topology hint indicates a preference for some well-known local resources.
The Topology Hint currently consists of 
* A list of bitmasks denoting the possible NUMA Nodes where a request can be satisfied.
* A preferred field.
    * This field is defined as follows:
      * For each Hint Provider, there is a possible resource assignment that satisfies the request, such that the least possible number of NUMA nodes is involved (caculated as if the node were empty.)
      * There is a possible assignment where the union of involved NUMA nodes for all such resource is no larger than the width required for any single resource.

#### Policies

**none (default)**: Kubelet does not consult Topology Manager for placement decisions. 

**best-effort**: Topology Manager will provide the preferred allocation for the container based
on the hints provided by the Hint Providers. If an undesirable allocation is determined, the pod will be admitted with this undesirable allocation.

**restricted**: Topology Manager will provide the preferred allocation for the container based
on the hints provided by the Hint Providers. If an undesirable allocation is determined, the pod will be rejected. 
This will result in the pod being in a `Terminated` state, with a pod admission failure.

**single-numa-node**: Topology mananager will enforce an allocation of all resources on a single NUMA Node. If such
an allocation is not possible, the pod will be rejected. This will result in the pod being in a `Terminated` state, with a pod admission failure.

The Topology Manager component will be disabled behind a feature gate until
graduation from alpha to beta.

#### Computing Preferred Affinity

After collecting hints from all providers, the Topology Manager preforms the
affinity calcuation to determine the best fit Topology Hint.

The chosen Topology Manager policy then decicds to admit or reject the pod based on this hint.

**Affinity Calcuation:**
1. Loops through the list of hint providers and saves an accumulated list of 
   the hints returned by each hint provider.
2. Iterates through all permutations of hints accumulated in Step 1. The hint affinites are merged to a single hint
   by performing a bitwise AND. The preferred field on the merged hint is set to false if any of the hints in the 
   permutation returned a false preferred.
3. The hint with the narrowest preferred affinity is returned.
   * Narrowest in this case means the least number of NUMA nodes required to satisfy the resource request.      
4. If no hint with at least one NUMA Node set is found, return a default hint which is a hint
   with all NUMA Nodes set and preferred set to false.

**Policy Decisions:**

- **best-effort**
    * Admits the pod to the node regardless of the Topology Hint stored.
- **restricted**:
    * Admits the pod to the node if the preferred field of the Topology Hint is set to true.
- **single-numa-node**:
    * Admits the pod to the node if the preferred field of the Topology is set to true **and** the bitmask is set to a single NUMA node.

#### New Interfaces

```go
package bitmask

// BitMask interface allows hint providers to create BitMasks for TopologyHints
type BitMask interface {
	Add(sockets ...int) error
	Remove(sockets ...int) error
	And(masks ...BitMask)
	Or(masks ...BitMask)
	Clear()
	Fill()
	IsEqual(mask BitMask) bool
	IsEmpty() bool
	IsSet(socket int) bool
	IsNarrowerThan(mask BitMask) bool
	String() string
	Count() int
	GetSockets() []int
}

func NewBitMask(sockets ...int) (BitMask, error) { ... }

package topologymanager

// Manager interface provides methods for Kubelet to manage pod topology hints
type Manager interface {
    // Implements pod admit handler interface
    lifecycle.PodAdmitHandler
    // Adds a hint provider to manager to indicate the hint provider
    //wants to be consoluted when making topology hints
    AddHintProvider(HintProvider)
    // Adds pod to Manager for tracking
    AddContainer(pod *v1.Pod, containerID string) error
    // Removes pod from Manager tracking
    RemoveContainer(containerID string) error
    // Interface for storing pod topology hints
    Store
}

// TopologyHint encodes locality to local resources. Each HintProvider provides
// a list of these hints to the TopoologyManager for each container at pod
// admission time.
type TopologyHint struct {
    NUMANodeAffinity bitmask.BitMask
    // Preferred is set to true when the BitMask encodes a preferred
    // allocation for the Container. It is set to false otherwise.
    Preferred bool
}

// HintProvider is implemented by Kubelet components that make
// topology-related resource assignments. The Topology Manager consults each
// hint provider at pod admission time.
type HintProvider interface {
  // Returns hints if this hint provider has a preference; otherwise
  // returns `nil` to indicate "don't care".
  GetTopologyHints(pod v1.Pod, containerName string) map[string][]TopologyHint
}

// Store manages state related to the Topology Manager.
type Store interface {
  // GetAffinity returns the preferred affinity as calculated by the
  // TopologyManager across all hint providers for the supplied pod and
  // container.
  GetAffinity(podUID string, containerName string) TopologyHint
}
```

_Listing: Topology Manager and related interfaces (sketch)._

![topology-manager-components](https://user-images.githubusercontent.com/379372/47447523-8efd2b00-d772-11e8-924d-eea5a5e00037.png)

_Figure: Topology Manager components._

![topology-manager-instantiation](https://user-images.githubusercontent.com/379372/47447526-945a7580-d772-11e8-9761-5213d745e852.png)

_Figure: Topology Manager instantiation and inclusion in pod admit lifecycle._
 
### Feature Gate and Kubelet Flags
 
A new feature gate will be added to enable the Topology Manager feature. This feature gate will be enabled in Kubelet, and will be disabled by default in the Alpha release.  

 * Proposed Feature Gate:  
  `--feature-gate=TopologyManager=true`  
 
 This will be also followed by a Kubelet Flag for the Topology Manager Policy, which is described above. The `none` policy will be the default policy.
 
 * Proposed Policy Flag:  
 `--topology-manager-policy=none|best-effort|restricted|single-numa-node`  
 
### Changes to Existing Components

1. Kubelet consults Topology Manager for pod admission (discussed above.)
1. Add two implementations of Topology Manager interface and a feature gate.
    1. As much Topology Manager functionality as possible is stubbed when the
       feature gate is disabled.
    1. Add a functional Topology Manager that queries hint providers in order
       to compute a preferred socket mask for each container.
1. Add `GetTopologyHints()` method to CPU Manager.
    1. CPU Manager static policy calls `GetAffinity()` method of
       Topology Manager when deciding CPU affinity.
1. Add `GetTopologyHints()` method to Device Manager.
    1. Add `TopologyInfo` to Device structure in the device plugin
       interface. Plugins should be able to determine the NUMA Node(s)
       when enumerating supported devices. See the protocol diff below.
    1. Device Manager calls `GetAffinity()` method of Topology Manager when
       deciding device allocation.
 
```diff
diff --git a/pkg/kubelet/apis/deviceplugin/v1beta1/api.proto b/pkg/kubelet/apis/deviceplugin/v1beta1/api.proto
index efbd72c133..f86a1a5512 100644
--- a/pkg/kubelet/apis/deviceplugin/v1beta1/api.proto
+++ b/pkg/kubelet/apis/deviceplugin/v1beta1/api.proto
@@ -73,6 +73,10 @@ message ListAndWatchResponse {
 	repeated Device devices = 1;
 }

+message TopologyInfo {
+  repeated NUMANode nodes = 1;
+}
+
+message NUMANode {
+    int64 ID = 1;
+}
+
 /* E.g:
 * struct Device {
 *    ID: "GPU-fef8089b-4820-abfc-e83e-94318197576e",
 *    State: "Healthy",
+ *    Topology: 
+ *      Nodes: 
+ *        ID: 1 
@@ -85,6 +89,8 @@ message Device {
 	string ID = 1;
 	// Health of the device, can be healthy or unhealthy, see constants.go
 	string health = 2;
+	// Topology details of the device
+	TopologyInfo topology = 3;
 }
```

_Listing: Amended device plugin gRPC protocol._

![topology-manager-wiring](https://user-images.githubusercontent.com/379372/47447533-9a505680-d772-11e8-95ca-ef9a8290a46a.png)

_Figure: Topology Manager hint provider registration._

![topology-manager-hints](https://user-images.githubusercontent.com/379372/47447543-a0463780-d772-11e8-8412-8bf4a0571513.png)

_Figure: Topology Manager fetches affinity from hint providers._

# Graduation Criteria

## Phase 1: Alpha (v1.16) [COMPLETED]

* Feature gate is disabled by default.
* Alpha-level documentation.
* Unit test coverage.
* CPU Manager allocation policy takes topology hints into account.
* Device plugin interface includes NUMA Node ID.
* Device Manager allocation policy takes topology hints into account.

## Phase 2: Beta (target v1.17)

* Enable the feature gate by default.
* Provide beta-level documentation.
* Add node E2E tests.
* Allow pods in all QoS classes to request aligned resources.
* Guarantee aligned resources for multiple containers in a pod.
* Refactor to easily support different merge strategies for different policies.
* Add support for device-specific topology constraints beyond NUMA.

## GA (stable)

* Support hugepages alignment.
* User feedback.
* *TBD*

# Challenges

* Testing the Topology Manager in a continuous integration environment
  depends on cloud infrastructure to expose multi-node topologies
  to guest virtual machines.
* Implementing the `GetHints()` interface may prove challenging.

# Limitations

* Alignment is only possible for pods in the `Guaranteed` QoS class.
* Alignment is only guaranteed for a single container inside a pod.

# Alternatives

* [AutoNUMA][numa-challenges]: This kernel feature affects memory
  allocation and thread scheduling, but does not address device locality.

# References

* *TBD*

[k8s-issue-49964]: https://github.com/kubernetes/kubernetes/issues/49964
[nfd-issue-84]: https://github.com/kubernetes-incubator/node-feature-discovery/issues/84
[sriov-issue-10]: https://github.com/hustcat/sriov-cni/issues/10
[proposal-affinity]: https://github.com/kubernetes/community/pull/171
[numa-challenges]: https://queue.acm.org/detail.cfm?id=2852078
