# Volume Topology-aware Scheduling

Authors: @msau42, @lichuqiang

This document presents a detailed design for making the default Kubernetes
scheduler aware of volume topology constraints, and making the
PersistentVolumeClaim (PVC) binding aware of scheduling decisions.

## Definitions
* Topology: Rules to describe accessibility of an object with respect to
  location in a cluster.
* Domain: A grouping of locations within a cluster. For example, 'node1',
  'rack10', 'zone5'.
* Topology Key: A description of a general class of domains. For example,
  'node', 'rack', 'zone'.
* Hierarchical domain: Domain that can be fully encompassed in a larger domain.
  For example, the 'zone1' domain can be fully encompassed in the 'region1'
  domain.
* Failover domain: A domain that a workload intends to run in at a later time.

## Goals
* Allow a Pod to request one or more Persistent Volumes (PV) with topology that
  are compatible with the Pod's other scheduling constraints, such as resource
  requirements and affinity/anti-affinity policies.
* Support arbitrary PV topology domains (i.e. node, rack, zone, foo, bar).
* Support topology for statically created PVs and dynamically provisioned PVs.
* No scheduling latency performance regression for Pods that do not use
  PVs with topology.
* Allow administrators to restrict allowed topologies per StorageClass.
* Allow storage providers to report capacity limits per topology domain.

## Non Goals
* Fitting a pod after the initial PVC binding has been completed.
    * The more constraints you add to your pod, the less flexible it becomes
in terms of placement.  Because of this, tightly constrained storage, such as
local storage, is only recommended for specific use cases, and the pods should
have higher priority in order to preempt lower priority pods from the node.
* Binding decision considering scheduling constraints from two or more pods
sharing the same PVC.
    * The scheduler itself only handles one pod at a time.  It’s possible the
two pods may not run at the same time either, so there’s no guarantee that you
will know both pod’s requirements at once.
    * For two+ pods simultaneously sharing a PVC, this scenario may require an
operator to schedule them together.  Another alternative is to merge the two
pods into one.
    * For two+ pods non-simultaneously sharing a PVC, this scenario could be
handled by pod priorities and preemption.
* Provisioning multi-domain volumes where all the domains will be able to run
  the workload. For example, provisioning a multi-zonal volume and making sure
  the pod can run in all zones.
    * Scheduler cannot make decisions based off of future resource requirements,
      especially if those resources can fluctuate over time. For applications that
      use such multi-domain storage, the best practice is to either:
        * Configure cluster autoscaling with enough resources to accomodate
          failing over the workload to any of the other failover domains.
        * Manually configure and overprovision the failover domains to
          accomodate the resource requirements of the workload.
* Scheduler supporting volume topologies that are independent of the node's
  topologies.
    * The Kubernetes scheduler only handles topologies with respect to the
      workload and the nodes it runs on. If a storage system is deployed on an
      independent topology, it will be up to provisioner to correctly spread the
      volumes for a workload. This could be facilitated as a separate feature
      by:
        * Passing the Pod's OwnerRef to the provisioner, and the provisioner
          spreading volumes for Pods with the same OwnerRef
        * Adding Volume Anti-Affinity policies, and passing those to the
          provisioner.


## Problem
Volumes can have topology constraints that restrict the set of nodes that the
volume can be accessed on.  For example, a GCE PD can only be accessed from a
single zone, and a local disk can only be accessed from a single node.  In the
future, there could be other topology domains, such as rack or region.

A pod that uses such a volume must be scheduled to a node that fits within the
volume’s topology constraints.  In addition, a pod can have further constraints
and limitations, such as the pod’s resource requests (cpu, memory, etc), and
pod/node affinity and anti-affinity policies.

Currently, the process of binding and provisioning volumes are done before a pod
is scheduled.  Therefore, it cannot take into account any of the pod’s other
scheduling constraints.  This makes it possible for the PV controller to bind a
PVC to a PV or provision a PV with constraints that can make a pod unschedulable.

### Examples
* In multizone clusters, the PV controller has a hardcoded heuristic to provision
PVCs for StatefulSets spread across zones.  If that zone does not have enough
cpu/memory capacity to fit the pod, then the pod is stuck in pending state because
its volume is bound to that zone.
* Local storage exasperates this issue.  The chance of a node not having enough
cpu/memory is higher than the chance of a zone not having enough cpu/memory.
* Local storage PVC binding does not have any node spreading logic.  So local PV
binding will very likely conflict with any pod anti-affinity policies if there is
more than one local PV on a node.
* A pod may need multiple PVCs.  As an example, one PVC can point to a local SSD for
fast data access, and another PVC can point to a local HDD for logging.  Since PVC
binding happens without considering if multiple PVCs are related, it is very likely
for the two PVCs to be bound to local disks on different nodes, making the pod
unschedulable.
* For multizone clusters and deployments requesting multiple dynamically provisioned
zonal PVs, each PVC is provisioned independently, and is likely to provision each PV
in different zones, making the pod unschedulable.

To solve the issue of initial volume binding and provisioning causing an impossible
pod placement, volume binding and provisioning should be more tightly coupled with
pod scheduling.


## Volume Topology Specification
First, volumes need a way to express topology constraints against nodes. Today, it
is done for zonal volumes by having explicit logic to process zone labels on the
PersistentVolume. However, this is not easily extendable for volumes with other
topology keys.

Instead, to support a generic specification, the PersistentVolume
object will be extended with a new NodeAffinity field that specifies the
constraints.  It will closely mirror the existing NodeAffinity type used by
Pods, but we will use a new type so that we will not be bound by existing and
future Pod NodeAffinity semantics.

```
type PersistentVolumeSpec struct {
    ...

    NodeAffinity *VolumeNodeAffinity
}

type VolumeNodeAffinity struct {
    // The PersistentVolume can only be accessed by Nodes that meet
    // these required constraints
    Required *NodeSelector
}
```

The `Required` field is a hard constraint and indicates that the PersistentVolume
can only be accessed from Nodes that satisfy the NodeSelector.

In the future, a `Preferred` field can be added to handle soft node constraints with
weights, but will not be included in the initial implementation.

The advantages of this NodeAffinity field vs the existing method of using zone labels
on the PV are:
* We don't need to expose first-class labels for every topology key.
* Implementation does not need to be updated every time a new topology key
  is added to the cluster.
* NodeSelector is able to express more complex topology with ANDs and ORs.
* NodeAffinity aligns with how topology is represented with other Kubernetes
  resources.

Some downsides include:
* You can have a proliferation of Node labels if you are running many different
  kinds of volume plugins, each with their own topology labeling scheme.
* The NodeSelector is more expressive than what most storage providers will
  need. Most storage providers only need a single topology key with
  one or more domains.  Non-hierarchical domains may present implementation
  challenges, and it will be difficult to express all the functionality
  of a NodeSelector in a non-Kubernetes specification like CSI.


### Example PVs with NodeAffinity
#### Local Volume
In this example, the volume can only be accessed from nodes that have the
label key `kubernetes.io/hostname` and label value `node-1`.
```
apiVersion: v1
kind: PersistentVolume
metadata:
  Name: local-volume-1
spec:
  capacity:
    storage: 100Gi
  storageClassName: my-class
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node-1
```

#### Zonal Volume
In this example, the volume can only be accessed from nodes that have the
label key `failure-domain.beta.kubernetes.io/zone' and label value
`us-central1-a`.
```
apiVersion: v1
kind: PersistentVolume
metadata:
  Name: zonal-volume-1
spec:
  capacity:
    storage: 100Gi
  storageClassName: my-class
  gcePersistentDisk:
    diskName: my-disk
    fsType: ext4
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: failure-domain.beta.kubernetes.io/zone
          operator: In
          values:
          - us-central1-a
```

#### Multi-Zonal Volume
In this example, the volume can only be accessed from nodes that have the
label key `failure-domain.beta.kubernetes.io/zone' and label value
`us-central1-a` OR 'us-central1-b.
```
apiVersion: v1
kind: PersistentVolume
metadata:
  Name: multi-zonal-volume-1
spec:
  capacity:
    storage: 100Gi
  storageClassName: my-class
  gcePersistentDisk:
    diskName: my-disk
    fsType: ext4
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: failure-domain.beta.kubernetes.io/zone
          operator: In
          values:
          - us-central1-a
          - us-central1-b
```

### Default Specification
Existing admission controllers and dynamic provisioners for zonal volumes
will be updated to specify PV NodeAffinity in addition to the existing zone
and region labels.  This will handle newly created PV objects.

Existing PV objects will have to be upgraded to use the new NodeAffinity field.
This does not have to occur instantaneously, and can be updated within the
deprecation period.  This can be done through a new PV update controller that
will take existing zone labels on PVs and translate that into PV.NodeAffinity.

An admission controller can validate that the labels and PV.NodeAffinity for
this existing zonal volume types are kept in sync.

### Bound PVC Enforcement
For PVCs that are already bound to a PV with NodeAffinity, enforcement is
simple and will be done at two places:
* Scheduler predicate: if a Pod references a PVC that is bound to a PV with
NodeAffinity, the predicate will evaluate the `Required` NodeSelector against
the Node's labels to filter the nodes that the Pod can be schedule to.  The
existing VolumeZone scheduling predicate will coexist with this new predicate
for several releases until PV NodeAffinity becomes GA and we can deprecate the
old predicate.
* Kubelet: PV NodeAffinity is verified against the Node when mounting PVs.

### Unbound PVC Binding
As mentioned in the problem statement, volume binding occurs without any input
about a Pod's scheduling constraints.  To fix this, we will delay volume binding
and provisioning until a Pod is created.  This behavior change will be opt-in as a
new StorageClass parameter.

Both binding decisions of:
* Selecting a precreated PV with NodeAffinity
* Dynamically provisioning a PV with NodeAffinity

will be considered by the scheduler, so that all of a Pod's scheduling
constraints can be evaluated at once.

The detailed design for implementing this new volume binding behavior will be
described later in the scheduler integration section.

## Delayed Volume Binding
Today, volume binding occurs immediately once a PersistentVolumeClaim is
created. In order for volume binding to take into account all of a pod's other scheduling
constraints, volume binding must be delayed until a Pod is being scheduled.

A new StorageClass field `VolumeBindingMode` will be added to control the volume
binding behavior.

```
type StorageClass struct {
    ...

    VolumeBindingMode *VolumeBindingMode
}

type VolumeBindingMode string

const (
    VolumeBindingImmediate VolumeBindingMode = "Immediate"
    VolumeBindingWaitForFirstConsumer VolumeBindingMode = "WaitForFirstConsumer"
)
```

`VolumeBindingImmediate`  is the default and current binding method.

This approach allows us to introduce the new binding behavior gradually and to
be able to maintain backwards compatibility without deprecation of previous
behavior. However, it has a few downsides:
* StorageClass will be required to get the new binding behavior, even if dynamic
provisioning is not used (in the case of local storage).
* We have to maintain two different code paths for volume binding.
* We will be depending on the storage admin to correctly configure the
StorageClasses for the volume types that need the new binding behavior.
* User experience can be confusing because PVCs could have different binding
behavior depending on the StorageClass configuration.  We will mitigate this by
adding a new PVC event to indicate if binding will follow the new behavior.


## Dynamic Provisioning with Topology
To make dynamic provisioning aware of pod scheduling decisions, delayed volume
binding must also be enabled. The scheduler will pass its selected node to the
dynamic provisioner, and the provisioner will create a volume in the topology
domain that the selected node is part of. The domain depends on the volume
plugin. Zonal volume plugins will create the volume in the zone where the
selected node is in. The local volume plugin will create the volume on the
selected node.

### End to End Zonal Example
This is an example of the most common use case for provisioning zonal volumes.
For this use case, the user's specs are unchanged. Only one change
to the StorageClass is needed to enable delayed volume binding.

1. Admin sets up StorageClass, setting up delayed volume binding.
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/gce-pd
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: pd-standard
```
2. Admin launches provisioner.  For in-tree plugins, nothing needs to be done.
3. User creates PVC. Nothing changes in the spec, although now the PVC won't be
   immediately bound.
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  storageClassName: standard
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```
4. User creates Pod. Nothing changes in the spec.
```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  ...
  volumes:
  - name: my-vol
    persistentVolumeClaim:
      claimName: my-pvc
```
5. Scheduler picks a node that can satisfy the Pod and passes it to the
   provisioner.
6. Provisioner dynamically provisions a PV that can be accessed from
   that node.
```
apiVersion: v1
kind: PersistentVolume
metadata:
  Name: volume-1
spec:
  capacity:
    storage: 100Gi
  storageClassName: standard
  gcePersistentDisk:
    diskName: my-disk
    fsType: ext4
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: failure-domain.beta.kubernetes.io/zone
          operator: In
          values:
          - us-central1-a
```
7. Pod gets scheduled to the node.


### Restricting Topology
For the common use case, volumes will be provisioned in whatever topology domain
the scheduler has decided is best to run the workload. Users may impose further
restrictions by setting label/node selectors, and pod affinity/anti-affinity
policies on their Pods. All those policies will be taken into account when
dynamically provisioning a volume.

While less common, administrators may want to further restrict what topology
domains are available to a StorageClass. To support these administrator
policies, a VolumeProvisioningTopology field can also be specified in the
StorageClass to restrict the allowed topology domains for dynamic provisioning.
This is not expected to be a common use case, and there are some caveats,
described below.

```
type StorageClass struct {
    ...

    AllowedTopology *VolumeProvisioningTopology
}

type VolumeProvisioningTopology struct {
    Required map[string]VolumeTopologyValues
}

type VolumeTopologyValues struct{
    Values []string
}
```

A nil value means there are no topology restrictions. A scheduler predicate
will evaluate a non-nil value when considering dynamic provisioning for a node.
When evaluating the specified topology against a node, the node must have a
label for each topology key specified and one of its allowed values. For
example, a topology specified with "zone: [us-central1-a, us-central1-b]" means
that the node must have the label "zone: us-central1-a" or "zone:
us-central1-b".

A StorageClass update controller can be provided to convert existing topology
restrictions specified under plugin-specific parameters to the new AllowedTopology
field.

The AllowedTopology will also be provided to provisioners as a new field, detailed in
the provisioner section.  Provisioners can use the allowed topology information
in the following scenarios:
* StorageClass is using the default immediate binding mode. This is the
  legacy topology-unaware behavior. In this scenario, the volume could be
  provisioned in a domain that cannot run the Pod since it doesn't take any
  scheduler input.
* For volumes that span multiple domains, the AllowedTopology can restrict those
  additional domains.  However, special care must be taken to avoid specifying
  conflicting topology constraints in the Pod. For example, the administrator could
  restrict a multi-zonal volume to zones 'zone1' and 'zone2', but the Pod could have
  constraints that restrict it to 'zone1' and 'zone3'.  If 'zone1'
  fails, the Pod cannot be scheduled to the intended failover zone.

Kubernetes will leave validation and enforcement of the AllowedTopology content up
to the provisioner.

TODO: would this be better represented as part of StorageClass quota since that
is where admins currently specify policies?
However, ResourceQuota currently only supports quantity, and topology
restrictions have to be evaluated during scheduling, not admission.

#### Zonal Example
This example restricts the volumes provisioned to zones us-central1-a and
us-central1-b.
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  Name: zonal-class
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
allowedTopology:
  required:
    failure-domain.beta.kubernetes.io/zone:
      values:
      - us-central1-a
      - us-central1-b
```

#### Multi-Zonal Example
This example restricts the volume's primary and failover zones
to us-central1-a and us-central1-b. Topologies that are incompatible with the
storage provider parameters will be enforced by the provisioner.
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  Name: multi-zonal-class
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
  replicas: 2
allowedTopology:
  required:
    failure-domain.beta.kubernetes.io/zone:
      values:
      - us-central1-a
      - us-central1-b
```

### Storage Provider Capacity Reporting
Some volume types have capacity limitations per topology domain. For example,
local volumes have limited capacity per node. Provisioners for these volume
types can report these capacity limitations through a new StorageClass status
field.

```
type StorageClass struct {
    ...

    Status *StorageClassStatus
}

type StorageClassStatus struct {
    Capacity *StorageClassCapacity
}

type StorageClassCapacity struct {
    TopologyKey string
    Capacities map[string]resource.Quantity
}
```

If no StorageClassCapacity is specified, then there are no capacity
limitations.

Capacity reporting is limited to a single topology key. `TopologyKey`
specifies the label key that represents that domain. Capacities is
a map with key equal to the label value, and value is the capacity for that
specific domain. Capacity is the total capacity available, including
capacity that has already been provisioned. A missing map entry for a topology
domain will be treated as zero capacity.

A scheduler predicate will use the reported capacity to determine if dynamic
provisioning is possible for a given node, taking into account already
provisioned capacity.

Downsides of this approach include:
* Multiple topology keys is not supported for capacity reporting. We have not
seen such use cases so far.
* There is no common mechanism to report capacity. Each provisioner has to
update their own fields in the StorageClass if necessary. CSI could potentially
address this.

#### Local Volume Example
This example specifies that node1 has 100Gi, and node2 has 200Gi available
for dynamic provisioning. All other nodes have 0Gi available for dynamic
provisioning.
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
...
status:
  capacity:
    topologyKey: kubernetes.io/hostname
    capacities:
      node1: 100Gi
      node2: 200Gi
```

TODO: should this be in the local volume spec instead?
Some downsides specific to the local volume implementation:
* StorageClass object size can be big for large clusters. Assuming that
  each node entry in the Capacities map could take up around 100 bytes
  (size of node name + resource.Quantity), a 1000 node cluster that uses
  dynamic provisioning of local volumes could add 100Ki to the StorageClass
  object.
* If each node has a local volume provisioner, there can be many
  StorageClass update conflicts if many try to update their capacity at the
  same time.
* A security concern is that each local volume provsioner could potentially
  modify the capacity reported by another node. This could cause
  provisioning to fail if a node does not have enough capacity, or it could
  cause the scheduler to skip nodes due to no capacity.


#### Relationship to StorageClassQuota
Although both the new StorageClassCapacity field and existing StorageClassQuota
manage storage consumption, they address different use cases.

The StorageClassCapacity field is actual available capacity per topology domain
reported by the storage system, while the StorageClassQuota represents policies
set by the cluster admin and does not take into account topology domain.

For example, a storage system is configured to support 200Gi in zone1 for a
StorageClass that is shared by multiple users.  An administrator can use
StorageClassQuota to restrict capacity usage for each user.


## Feature Gates
PersistentVolume.NodeAffinity and StorageClas.VolumeBindingMode fields will be
controlled by the VolumeScheduling feature gate, and must be configured in the
kube-scheduler, kube-controller-manager, and all kubelets.

StorageClass.AllowedTopology and StorageClassCapacity fields will be controlled
by the DynamicProvisioningScheduling feature gate, and must be configured in the
kube-scheduler and kube-controller-manager.


## Integrating volume binding with pod scheduling
For the new volume binding mode, the proposed new workflow is:
1. Admin statically creates PVs and/or StorageClasses.
2. User creates unbound PVC and there are no prebound PVs for it.
3. **NEW:** PVC binding and provisioning is delayed until a pod is created that
references it.
4. User creates a pod that uses the PVC.
5. Pod starts to get processed by the scheduler.
6. **NEW:** A new predicate function, called CheckVolumeBinding, will process
both bound and unbound PVCs of the Pod.  It will validate the VolumeNodeAffinity
for bound PVCs.  For unbound PVCs, it will try to find matching PVs for that node
based on the PV NodeAffinity.  If there are no matching PVs, then it checks if
dynamic provisioning is possible for that node based on StorageClass
AllowedTopologies and Capacity.
7. **NEW:** The scheduler continues to evaluate priorities.  A new priority
function, called PrioritizeVolumes, will get the PV matches per PVC per
node, and compute a priority score based on various factors.
8. **NEW:** After evaluating all the existing predicates and priorities, the
scheduler will pick a node, and call a new assume function, AssumeVolumes,
passing in the Node.  The assume function will check if any binding or
provisioning operations need to be done.  If so, it will update the PV cache to
mark the PVs with the chosen PVCs and queue the Pod for volume binding.
9. **NEW:** If PVC binding or provisioning is required, we do NOT AssumePod.
Instead, a new bind function, BindVolumes, will be called asynchronously, passing
in the selected node.  The bind function will prebind the PV to the PVC, or
trigger dynamic provisioning.  Then, it always sends the Pod through the
scheduler again for reasons explained later.
10. When a Pod makes a successful scheduler pass once all PVCs are bound, the
scheduler assumes and binds the Pod to a Node.
11. Kubelet starts the Pod.

This diagram depicts the new additions to the default scheduler:
![alt text](volume-topology-scheduling.png)

This new workflow will have the scheduler handle unbound PVCs by choosing PVs
and prebinding them to the PVCs.  The PV controller completes the binding
transaction, handling it as a prebound PV scenario.

Prebound PVCs and PVs will still immediately be bound by the PV controller.

Manual recovery by the user will be required in following error conditions:
* A Pod has multiple PVCs, and only a subset of them successfully bind.

The primary cause for these errors is if a user or external entity
binds a PV between the time that the scheduler chose the PV and when the
scheduler actually made the API update.  Some workarounds to
avoid these error conditions are to:
* Prebind the PV instead.
* Separate out volumes that the user prebinds from the volumes that are
available for the system to choose from by StorageClass.

### PV Controller Changes
When the feature gate is enabled, the PV controller needs to skip binding
unbound PVCs with VolumBindingWaitForFirstConsumer and no prebound PVs
to let it come through the scheduler path.

Dynamic provisioning will also be skipped if
VolumBindingWaitForFirstConsumer is set. The scheduler will signal to
the PV controller to start dynamic provisioning by setting the
`annSelectedNode` annotation in the PVC. If provisioning fails, the PV
controller can signal back to the scheduler to retry dynamic provisioning by
removing the `annSelectedNode` annotation.

No other state machine changes are required.  The PV controller continues to
handle the remaining scenarios without any change.

The methods to find matching PVs for a claim and prebind PVs need to be
refactored for use by the new scheduler functions.

### Dynamic Provisioning interface changes
The dynamic provisioning interfaces will be updated to pass in:
* selectedNode, when late binding is enabled on the StorageClass
* allowedTopologies, when it is set in the StorageClass

If selectedNode is set, the provisioner should get its appropriate topology
labels from the Node object, and provision a volume based on those topology
values. In the common use case for a volume supporting a single topology domain,
if nodeName is set, then allowedTopologies can be ignored.

In-tree provisioners:
```
Provision(selectedNode *string, allowedTopologies *storagev1.VolumeProvisioningTopology) (*v1.PersistentVolume, error)
```

External provisioners:
* selectedNode will be represented by the PVC annotation "volume.alpha.kubernetes.io/selectedNode"
* allowedTopologies must be obtained by looking at the StorageClass for the PVC.

#### New Permissions
Provisioners will need to be able to get Node and StorageClass objects.

### Scheduler Changes

#### Predicate
A new predicate function checks all of a Pod's unbound PVCs can be satisfied
by existing PVs or dynamically provisioned PVs that are
topologically-constrained to the Node.
```
CheckVolumeBinding(pod *v1.Pod, node *v1.Node) (canBeBound bool, err error)
```
1. If all the Pod’s PVCs are bound, return true.
2. Otherwise try to find matching PVs for all of the unbound PVCs in order of
decreasing requested capacity.
3. Walk through all the PVs.
4. Find best matching PV for the PVC where PV topology is satisfied by the Node.
5. Temporarily cache this PV choice for the PVC per Node, for fast
processing later in the priority and bind functions.
6. Return true if all PVCs are matched.
7. If there are still unmatched PVCs, check if dynamic provisioning is possible,
   by evaluating StorageClass.AllowedTopology, and StorageClassCapacity.  If so,
   temporarily cache this decision in the PVC per Node.
8. Otherwise return false.

#### Priority
After all the predicates run, there is a reduced set of Nodes that can fit a
Pod. A new priority function will rank the remaining nodes based on the
unbound PVCs and their matching PVs.
```
PrioritizeVolumes(pod *v1.Pod, filteredNodes HostPriorityList) (rankedNodes HostPriorityList, err error)
```
1. For each Node, get the cached PV matches for the Pod’s PVCs.
2. Compute a priority score for the Node using the following factors:
    1. How close the PVC’s requested capacity and PV’s capacity are.
    2. Matching static PVs is preferred over dynamic provisioning because we
       assume that the administrator has specifically created these PVs for
       the Pod.

TODO (beta): figure out weights and exact calculation

#### Assume
Once all the predicates and priorities have run, then the scheduler picks a
Node.  Then we can bind or provision PVCs for that Node.  For better scheduler
performance, we’ll assume that the binding will likely succeed, and update the
PV and PVC caches first.  Then the actual binding API update will be made
asynchronously, and the scheduler can continue processing other Pods.

For the alpha phase, the AssumeVolumes function will be directly called by the
scheduler.  We’ll consider creating a generic scheduler interface in a
subsequent phase.

```
AssumeVolumes(pod *v1.pod, node *v1.node) (pvcbindingrequired bool, err error)
```
1. If all the Pod’s PVCs are bound, return false.
2. For static PV binding:
    1. Get the cached matching PVs for the PVCs on that Node.
    2. Validate the actual PV state.
    3. Mark PV.ClaimRef in the PV cache.
    4. Cache the PVs that need binding in the Pod object.
3. For in-tree and external dynamic provisioning:
    1. Mark the PVC annSelectedNode in the PVC cache.
    2. If the StorageClass has reported capacity limits, cache the topologyKey
       value in the PVC annProvisionedTopology, and update the StorageClass
       capacity cache.
    3. Cache the PVCs that need provisioning in the Pod object.
4. Return true.

#### Bind
If AssumeVolumes returns pvcBindingRequired, then the BindPVCs function is called
as a go routine.  Otherwise, we can continue with assuming and binding the Pod
to the Node.

For the alpha phase, the BindVolumes function will be directly called by the
scheduler.  We’ll consider creating a generic scheduler interface in a subsequent
phase.

```
BindVolumes(pod *v1.Pod, node *v1.Node) (err error)
```
1. For static PV binding:
    1. Prebind the PV by updating the `PersistentVolume.ClaimRef` field.
    2. If the prebind fails, revert the cache updates.
2. For in-tree and external dynamic provisioning:
    1. Set `annScheduledNode` on the PVC.
3. Send Pod back through scheduling, regardless of success or failure.
    1. In the case of success, we need one more pass through the scheduler in
order to evaluate other volume predicates that require the PVC to be bound, as
described below.
    2. In the case of failure, we want to retry binding/provisioning.

TODO: pv controller has a high resync frequency, do we need something similar
for the scheduler too

#### Access Control
Scheduler will need PV update permissions for prebinding static PVs, and PVC
update permissions for triggering dynamic provisioning.

#### Pod preemption considerations
The CheckVolumeBinding predicate does not need to be re-evaluated for pod
preemption.  Preempting a pod that uses a PV will not free up capacity on that
node because the PV lifecycle is independent of the Pod’s lifecycle.

#### Other scheduler predicates
Currently, there are a few existing scheduler predicates that require the PVC
to be bound.  The bound assumption needs to be changed in order to work with
this new workflow.

TODO: how to handle race condition of PVCs becoming bound in the middle of
running predicates?  One possible way is to mark at the beginning of scheduling
a Pod if all PVCs were bound.  Then we can check if a second scheduler pass is
needed.

##### Max PD Volume Count Predicate
This predicate checks the maximum number of PDs per node is not exceeded.  It
needs to be integrated into the binding decision so that we don’t bind or
provision a PV if it’s going to cause the node to exceed the max PD limit.  But
until it is integrated, we need to make one more pass in the scheduler after all
the PVCs are bound.  The current copy of the predicate in the default scheduler
has to remain to account for the already-bound volumes.

##### Volume Zone Predicate
This predicate makes sure that the zone label on a PV matches the zone label of
the node.  If the volume is not bound, this predicate can be ignored, as the
binding logic will take into account zone constraints on the PV.

However, this assumes that zonal PVs like GCE PDs and AWS EBS have been updated
to use the new PV topology specification, which is not the case as of 1.8.  So
until those plugins are updated, the binding and provisioning decisions will be
topology-unaware, and we need to make one more pass in the scheduler after all
the PVCs are bound.

This predicate needs to remain in the default scheduler to handle the
already-bound volumes using the old zonal labeling.  It can be removed once that
mechanism is deprecated and unsupported.

##### Volume Node Predicate
This is a new predicate added in 1.7 to handle the new PV node affinity.  It
evaluates the node affinity against the node’s labels to determine if the pod
can be scheduled on that node.  If the volume is not bound, this predicate can
be ignored, as the binding logic will take into account the PV node affinity.

#### Caching
There are three new caches needed in the scheduler.

The first cache is for handling the PV/PVC API binding updates occurring
asynchronously with the main scheduler loop.  `AssumeVolumes` needs to store
the updated API objects before `BindVolumes` makes the API update, so
that future binding decisions will not choose any assumed PVs.  In addition,
if the API update fails, the cached updates need to be reverted and restored
with the actual API object.  The cache will return either the cached-only
object, or the informer object, whichever one is latest.  Informer updates
will always override the cached-only object.  The new predicate and priority
functions must get the objects from this cache instead of from the informer cache.
This cache only stores pointers to objects and most of the time will only
point to the informer object, so the memory footprint per object is small.

The second cache is for storing temporary state as the Pod goes from
predicates to priorities and then assume.  This all happens serially, so
the cache can be cleared at the beginning of each pod scheduling loop.  This
cache is used for:
* Indicating if all the PVCs are already bound at the beginning of the pod
scheduling loop.  This is to handle situations where volumes may have become
bound in the middle of processing the predicates.  We need to ensure that
all the volume predicates are fully run once all PVCs are bound.
* Caching PV matches per node decisions that the predicate had made.  This is
an optimization to avoid walking through all the PVs again in priority and
assume functions.
* Caching PVC dynamic provisioning decisions per node that the predicate had
  made.

The third cache is for storing provisioned capacity per topology domain for
those StorageClasses that report capacity limits. Since the capacity reported in
the StorageClass is total capacity, we need to calculate used capacity by adding
up the capacities of all the PVCs provisioned for that StorageClass in each
domain.  We can cache these used capacities so we don't have to recalculate them
every time in the predicate.


#### Performance and Optimizations
Let:
* N = number of nodes
* V = number of all PVs
* C = number of claims in a pod

C is expected to be very small (< 5) so shouldn’t factor in.

The current PV binding mechanism just walks through all the PVs once, so its
running time O(V).

Without any optimizations, the new PV binding mechanism has to run through all
PVs for every node, so its running time is O(NV).

A few optimizations can be made to improve the performance:

1. Optimizing for PVs that don’t use node affinity (to prevent performance
regression):
    1. Index the PVs by StorageClass and only search the PV list with matching
StorageClass.
    2. Keep temporary state in the PVC cache if we previously succeeded or
failed to match PVs, and if none of the PVs have node affinity.  Then we can
skip PV matching on subsequent nodes, and just return the result of the first
attempt.
2. Optimizing for PVs that have node affinity:
    1. When a static PV is created, if node affinity is present, evaluate it
against all the nodes.  For each node, keep an in-memory map of all its PVs
keyed by StorageClass.  When finding matching PVs for a particular node, try to
match against the PVs in the node’s PV map instead of the cluster-wide PV list.

For the alpha phase, the optimizations are not required.  However, they should
be required for beta and GA.

### Packaging
The new bind logic that is invoked by the scheduler can be packaged in a few
ways:
* As a library to be directly called in the default scheduler
* As a scheduler extender

We propose taking the library approach, as this method is simplest to release
and deploy.  Some downsides are:
* The binding logic will be executed using two different caches, one in the
scheduler process, and one in the PV controller process.  There is the potential
for more race conditions due to the caches being out of sync.
* Refactoring the binding logic into a common library is more challenging
because the scheduler’s cache and PV controller’s cache have different interfaces
and private methods.

#### Extender cons
However, the cons of the extender approach outweighs the cons of the library
approach.

With an extender approach, the PV controller could implement the scheduler
extender HTTP endpoint, and the advantage is the binding logic triggered by the
scheduler can share the same caches and state as the PV controller.

However, deployment of this scheduler extender in a master HA configuration is
extremely complex.  The scheduler has to be configured with the hostname or IP of
the PV controller.  In a HA setup, the active scheduler and active PV controller
could run on the same, or different node, and the node can change at any time.
Exporting a network endpoint in the controller manager process is unprecedented
and there would be many additional features required, such as adding a mechanism
to get a stable network name, adding authorization and access control, and
dealing with DDOS attacks and other potential security issues.  Adding to those
challenges is the fact that there are countless ways for users to deploy
Kubernetes.

With all this complexity, the library approach is the most feasible in a single
release time frame, and aligns better with the current Kubernetes architecture.

### Downsides

#### Unsupported Use Cases
The following use cases will not be supported for PVCs with a StorageClass with
VolumeBindingWaitForFirstConsumer:
* Directly setting Pod.Spec.NodeName
* DaemonSets

These two use cases will bypass the default scheduler and thus will not
trigger PV binding.

#### Custom Schedulers
Custom schedulers, controllers and operators that handle pod scheduling and want
to support this new volume binding mode will also need to handle the volume
binding decision.

There are a few ways to take advantage of this feature:
* Custom schedulers could be implemented through the scheduler extender
interface.  This allows the default scheduler to be run in addition to the
custom scheduling logic.
* The new code for this implementation will be packaged as a library to make it
easier for custom schedulers to include in their own implementation.

In general, many advanced scheduling features have been added into the default
scheduler, such that it is becoming more difficult to run without it.

#### HA Master Upgrades
HA masters adds a bit of complexity to this design because the active scheduler
process and active controller-manager (PV controller) process can be on different
nodes.  That means during an HA master upgrade, the scheduler and controller-manager
can be on different versions.

The scenario where the scheduler is newer than the PV controller is fine.  PV
binding will not be delayed and in successful scenarios, all PVCs will be bound
before coming to the scheduler.

However, if the PV controller is newer than the scheduler, then PV binding will
be delayed, and the scheduler does not have the logic to choose and prebind PVs.
That will cause PVCs to remain unbound and the Pod will remain unschedulable.

TODO: One way to solve this is to have some new mechanism to feature gate system
components based on versions.  That way, the new feature is not turned on until
all dependencies are at the required versions.

For alpha, this is not concerning, but it needs to be solved by GA.

### Other Alternatives Considered

#### One scheduler function
An alternative design considered was to do the predicate, priority and bind
functions all in one function at the end right before Pod binding, in order to
reduce the number of passes we have to make over all the PVs.  However, this
design does not work well with pod preemption.  Pod preemption needs to be able
to evaluate if evicting a lower priority Pod will make a higher priority Pod
schedulable, and it does this by re-evaluating predicates without the lower
priority Pod.

If we had put the MatchUnboundPVCs predicate at the end, then pod preemption
wouldn’t have an accurate filtered nodes list, and could end up preempting pods
on a Node that the higher priority pod still cannot run on due to PVC
requirements.  For that reason, the PVC binding decision needs to be have its
predicate function separated out and evaluated with the rest of the predicates.

#### Pull entire PVC binding into the scheduler
The proposed design only has the scheduler initiating the binding transaction
by prebinding the PV.  An alternative is to pull the whole two-way binding
transaction into the scheduler, but there are some complex scenarios that
scheduler’s Pod sync loop cannot handle:
* PVC and PV getting unexpectedly unbound or lost
* PVC and PV state getting partially updated
* PVC and PV deletion and cleanup

Handling these scenarios in the scheduler’s Pod sync loop is not possible, so
they have to remain in the PV controller.

#### Keep all PVC binding in the PV controller
Instead of initiating PV binding in the scheduler, have the PV controller wait
until the Pod has been scheduled to a Node, and then try to bind based on the
chosen Node.  A new scheduling predicate is still needed to filter and match
the PVs (but not actually bind).

The advantages are:
* Existing scenarios where scheduler is bypassed will work.
* Custom schedulers will continue to work without any changes.
* Most of the PV logic is still contained in the PV controller, simplifying HA
upgrades.

Major downsides of this approach include:
* Requires PV controller to watch Pods and potentially change its sync loop
to operate on pods, in order to handle the multiple PVCs in a pod scenario.
This is a potentially big change that would be hard to keep separate and
feature-gated from the current PV logic.
* Both scheduler and PV controller processes have to make the binding decision,
but because they are done asynchronously, it is possible for them to choose
different PVs.  The scheduler has to cache its decision so that it won't choose
the same PV for another PVC.  But by the time PV controller handles that PVC,
it could choose a different PV than the scheduler.
    * Recovering from this inconsistent decision and syncing the two caches is
very difficult.  The scheduler could have made a cascading sequence of decisions
based on the first inconsistent decision, and they would all have to somehow be
fixed based on the real PVC/PV state.
* If the scheduler process restarts, it loses all its in-memory PV decisions and
can make a lot of wrong decisions after the restart.
* All the volume scheduler predicates that require PVC to be bound will not get
evaluated.  To solve this, all the volume predicates need to also be built into
the PV controller when matching possible PVs.

#### Move PVC binding to kubelet
Looking into the future, with the potential for NUMA-aware scheduling, you could
have a sub-scheduler on each node to handle the pod scheduling within a node.  It
could make sense to have the volume binding as part of this sub-scheduler, to make
sure that the volume selected will have NUMA affinity with the rest of the
resources that the pod requested.

However, there are potential security concerns because kubelet would need to see
unbound PVs in order to bind them.  For local storage, the PVs could be restricted
to just that node, but for zonal storage, it could see all the PVs in that zone.

In addition, the sub-scheduler is just a thought at this point, and there are no
concrete proposals in this area yet.

## Binding multiple PVCs in one transaction
There are no plans to handle this, but a possible solution is presented here if the
need arises in the future.  Since the scheduler is serialized, a partial binding
failure should be a rare occurrence and would only be caused if there is a user or
other external entity also trying to bind the same volumes.

One possible approach to handle this is to rollback previously bound PVCs on
error.  However, volume binding cannot be blindly rolled back because there could
be user's data on the volumes.

For rollback, PersistentVolumeClaims will have a new status to indicate if it's
clean or dirty.  For backwards compatibility, a nil value is defaulted to dirty.
The PV controller will set the status to clean if the PV is Available and unbound.
Kubelet will set the PV status to dirty during Pod admission, before adding the
volume to the desired state.

If scheduling fails, update all bound PVCs with an annotation,
"pv.kubernetes.io/rollback".  The PV controller will only unbind PVCs that
are clean.  Scheduler and kubelet needs to reject pods with PVCs that are
undergoing rollback.

## Recovering from kubelet rejection of pod
We can use the same rollback mechanism as above to handle this case.
If kubelet rejects a pod, it will go back to scheduling.  If the scheduler
cannot find a node for the pod, then it will encounter scheduling failure and
initiate the rollback.


## Testing

### E2E tests
* StatefulSet, replicas=3, specifying pod anti-affinity
    * Positive: Local PVs on each of the nodes
    * Negative: Local PVs only on 2 out of the 3 nodes
* StatefulSet specifying pod affinity
    * Positive: Multiple local PVs on a node
    * Negative: Only one local PV available per node
* Multiple PVCs specified in a pod
    * Positive: Enough local PVs available on a single node
    * Negative: Not enough local PVs available on a single node
* Fallback to dynamic provisioning if unsuitable static PVs

### Unit tests
* All PVCs found a match on first node.  Verify match is best suited based on
capacity.
* All PVCs found a match on second node.  Verify match is best suited based on
capacity.
* Only 2 out of 3 PVCs have a match.
* Priority scoring doesn’t change the given priorityList order.
* Priority scoring changes the priorityList order.
* Don’t match PVs that are prebound


## Implementation Plan

### Alpha
* New feature gate for volume topology scheduling
* StorageClass API change
* Refactor PV controller methods into a common library
* PV controller: Delay binding and provisioning unbound PVCs
* Predicate: Filter nodes and find matching PVs
* Predicate: Check if provisioner exists for dynamic provisioning
* Update existing predicates to skip unbound PVC
* Bind: Trigger PV binding
* Bind: Trigger dynamic provisioning
a Pod (only if alpha is enabled)

### Beta
* Scheduler cache: Optimizations for no PV node affinity
* Priority: capacity match score
* Plugins: Convert all zonal volume plugins to use new PV node affinity (GCE PD,
AWS EBS, what else?)
* Make dynamic provisioning topology aware

### GA
* Predicate: Handle max PD per node limit
* Scheduler cache: Optimizations for PV node affinity


## Open Issues
* Can generic device resource API be leveraged at all?  Probably not, because:
    * It will only work for local storage (node specific devices), and not zonal
storage.
    * Storage already has its own first class resources in K8s (PVC/PV) with an
independent lifecycle.  The current resource API proposal does not have an a way to
specify identity/persistence for devices.
* Will this be able to work with the node sub-scheduler design for NUMA-aware
scheduling?
    * It’s still in a very early discussion phase.
