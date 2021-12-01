# NUMA Manager

_Authors:_

* @ConnorDoyle - Connor Doyle &lt;connor.p.doyle@intel.com&gt;
* @balajismaniam - Balaji Subramaniam &lt;balaji.subramaniam@intel.com&gt;
* @lmdaly - Louise M. Daly &lt;louise.m.daly@intel.com&gt;

**Contents:**

* [Overview](#overview)
* [Motivation](#motivation)
  * [Goals](#goals)
  * [Non-Goals](#non-goals)
  * [User Stories](#user-stories)
* [Proposal](#proposal)
  * [User Stories](#user-stories)
  * [Proposed Changes](#proposed-changes)
    * [New Component: NUMA Manager](#new-component-numa-manager)
      * [Computing Preferred Affinity](#computing-preferred-affinity)
      * [New Interfaces](#new-interfaces)
    * [Changes to Existing Components](#changes-to-existing-components)
* [Graduation Criteria](#graduation-criteria)
  * [alpha (target v1.11)](#alpha-target-v1.11)
  * [beta](#beta)
  * [GA (stable)](#ga-stable)
* [Challenges](#challenges)
* [Limitations](#limitations)
* [Alternatives](#alternatives)
* [Reference](#reference)

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
  - NICs including SR-IOV Virtual Functions have affinity to one NUMA node,
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

- Arbitrate preferred NUMA node affinity for containers based on input from
  CPU manager and Device Manager.
- Provide an internal interface and pattern to integrate additional
  topology-aware Kubelet components.

## Non-Goals

- _Inter-device connectivity:_ Decide device assignments based on direct
  device interconnects. This issue can be separated from NUMA node
  locality. Inter-device topology can be considered entirely within the
  scope of the Device Manager, after which it can emit possible
  NUMA affinities. The policy to reach that decision can start simple
  and iterate to include support for arbitrary inter-device graphs.
- _HugePages:_ This proposal assumes that pre-allocated HugePages are
  spread among the available NUMA nodes in the system. We further assume
  the operating system provides best-effort local page allocation for
  containers (as long as sufficient HugePages are free on the local NUMA
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
pieces coordinated (hugepages, cpusets, network device) co-located on a
NUMA node.

*Story 2: Accelerated neural network training*

A user asks for an accelerator device and some number of exclusive CPUs
in order to get the best training performance, due to NUMA-alignment of
the assigned CPUs and devices.

# Proposal

*Main idea: Two Phase NUMA coherence protocol*

NUMA affinity is tracked at the container level, similar to devices and
CPU affinity. At pod admission time, a new component called the NUMA Manager
collects possible NUMA configurations from the Device Manager and the
CPU Manager. The NUMA manager acts as an oracle for NUMA node affinity by
those same components when they make concrete resource allocations. We
expect the consulted components to use the inferred QoS class of each
pod in order to prioritize the importance of fulfilling optimal NUMA
affinity.

## Proposed Changes

### New Component: NUMA Manager

This proposal is focused on a new component in the Kubelet called the
NUMA Manager. The NUMA Manager implements the pod admit handler
interface and participates in Kubelet pod admission. When the `Admit()`
function is called, the NUMA manager collects NUMA hints from other
Kubelet components.

If the NUMA hints are not compatible, the NUMA manager could choose to
reject the pod. The details of what to do in this situation needs more
discussion. For example, the NUMA manager could enforce strict NUMA
alignment for Guaranteed QoS pods. Alternatively, the NUMA manager could
simply provide best-effort NUMA alignment for all pods. The NUMA manager could
use `softAdmitHandler` to keep the pod in `Pending` state.

The NUMA Manager component will be disabled behind a feature gate until
graduation from alpha to beta.

#### Computing Preferred Affinity

A NUMA hint is a list of possible NUMA node masks. After collecting hints
from all providers, the NUMA Manager must choose some mask that is
present in all lists. Here is a sketch:

1. Apply a partial order on each list: number of bits set in the
   mask, ascending. This biases the result to be more precise if
   possible.
1. Iterate over the permutations of preference lists and compute
   bitwise-and over the masks in each permutation.
1. Store the first non-empty result and break out early.
1. If no non-empty result exists, return an error.

The behavior when a match does not exist should be configurable. The Kubelet
could support a config option to require strict NUMA assignment when set to
`true`. A `false` value would mean best-effort NUMA alignment.

#### New Interfaces

```go
package numamanager

// NUMAManager helps to coordinate NUMA-related resource assignments
// within the Kubelet.
type Manager interface {
  lifecycle.PodAdmitHandler
  Store
  AddHintProvider(HintProvider)
  RemovePod(podName string)
}

// NUMAMask is a bitmask-like type denoting a subset of available NUMA nodes.
type NUMAMask struct{} // TBD

// NUMAStore manages state related to the NUMA manager.
type Store interface {
  // GetAffinity returns the preferred NUMA affinity for the supplied
  // pod and container.
  GetAffinity(podName string, containerName string) NUMAMask
}

// HintProvider is implemented by Kubelet components that make
// NUMA-related resource assignments. The NUMA manager consults each
// hint provider at pod admission time.
type HintProvider interface {
  // Returns a mask if this hint provider has a preference; otherwise
  // returns `_, false` to indicate "don't care".
  GetNUMAHints(pod v1.Pod, containerName string) ([]NUMAMask, bool)
}
```

_NUMA Manager and related interfaces (sketch)._

![numa-manager-components](https://user-images.githubusercontent.com/379372/35370509-13dd9488-0143-11e8-998b-6b5115982842.png)

_NUMA Manager components._

![numa-manager-instantiation](https://user-images.githubusercontent.com/379372/35370513-17f90f70-0143-11e8-88e3-f199e9717946.png)

_NUMA Manager instantiation and inclusion in pod admit lifecycle._

### Changes to Existing Components

1. Kubelet consults NUMA Manager for pod admission (discussed above.)
1. Add two implementations of NUMA Manager interface and a feature gate.
    1. As much NUMA Manager functionality as possible is stubbed when the
       feature gate is disabled.
    1. Add a functional NUMA manager that queries hint providers in order
       to compute a preferred NUMA node mask for each container.
1. Add `GetNUMAHints()` method to CPU Manager.
    1. CPU Manager static policy calls `GetAffinity()` method of NUMA
       manager when deciding CPU affinity.
1. Add `GetNUMAHints()` method to Device Manager.
    1. Add NUMA Node ID to Device structure in the device plugin
       interface. Plugins should be able to determine the NUMA node
       easily when enumerating supported devices. For example, Linux
       exposes the node ID in sysfs for PCI devices:
       `/sys/devices/pci*/*/numa_node`. NOTE: this is `-1` on many
       public cloud instances and single-node machines.
    1. Device Manager calls `GetAffinity()` method of NUMA manager when
       deciding device allocation.

![numa-manager-wiring](https://user-images.githubusercontent.com/379372/35370514-1e10fb84-0143-11e8-84d3-99c9ca3af111.png)

_NUMA Manager hint provider registration._

![numa-manager-hints](https://user-images.githubusercontent.com/379372/35370517-234a5d34-0143-11e8-845a-80e5c66c7b72.png)

_NUMA Manager fetches affinity from hint providers._

# Graduation Criteria

## Alpha (target v1.11)

* Feature gate is disabled by default.
* Alpha-level documentation.
* Unit test coverage.
* CPU Manager allocation policy takes NUMA hints into account.
* Device plugin interface includes NUMA node ID.
* Device Manager allocation policy takes NUMA hints into account.

## Beta

* Feature gate is enabled by default.
* Alpha-level documentation.
* Node e2e tests.
* User feedback.

## GA (stable)

* *TBD*

# Challenges

* Testing the NUMA Manager in a continuous integration environment
  depends on cloud infrastructure to expose multi-node NUMA topologies
  to guest virtual machines.
* Implementing the `GetNUMAHints()` interface may prove challenging.

# Limitations

* *TBD*

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
