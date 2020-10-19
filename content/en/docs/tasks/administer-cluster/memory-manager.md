---
title: Memory Manager

reviewers:
- klueska
- derekwaynecarr
- TBD

content_type: task
min-kubernetes-server-version: v1.20
---

<!-- overview -->

{{< feature-state state="alpha" for_k8s_version="v1.20" >}}

The *Memory Manager* enables the feature of guaranteed memory (and hugepages) allocation for pods in the Guaranteed [QoS class](/docs/tasks/configure-pod-container/quality-service-pod/#qos-classes). You can select from two memory allocation strategies. The first one, the *single-NUMA* strategy, is intended for high-performance and performance-sensitive applications. The second one, the *multi-NUMA* strategy, complements the overall design while it overcomes the situation that cannot be managed with the *single-NUMA* strategy.

With the *multi-NUMA* strategy, whenever the amount of memory demanded by a pod is in the excess of a single NUMA node capacity, the guaranteed memory is provisioned across multiple NUMA nodes.

In both scenarios, the *Memory Manager* employs hint generation protocol to yield the most suitable NUMA affinity for a pod, and it feeds the central manager (*Topology Manager*) with those affinity hints. Moreover, *Memory Manager* ensures that the memory which a pod requests is allocated from a minimum number of NUMA nodes.

## {{% heading "prerequisites" %}}

{{< include "task-tutorial-prereqs.md" >}} {{< version-check >}}

<!-- steps -->

## How Memory Manager Operates?

The Memory Manager currently offers the guaranteed memory (and hugepages) allocation for Pods in Guaranteed QoS class. To immediately put the Memory Manager into operation follow the guidelines in section [Memory Manager configuration](#memory-manager-configuration)), and subsequently, prepare and deploy a `Guaranteed` pod as illustrated in section [Placing a Pod in the Guaranteed QoS class](placing-a-pod-in-the-guaranteed-qos-class).

The Memory Manager is a Hint Provider, and it provides topology hints for the Topology Manager which then aligns the requested resources according to these topology hints. It also enforces `cgroups` (i.e. `cpuset.mems`) for pods. The complete flow diagram concerning pod admission and deployment process is illustrated in [Memory Manager KEP: Design Overview][4].

During this process, the Memory Manager updates its internal counters stored in [Node Map and Memory Maps][2] to manage guaranteed memory allocation. 

The Memory Manager updates the Node Map during the startup and runtime as follows.

### Startup

This is an optional step and occurs only if a node administrator employed `--reserved-memory` (section [Reserved memory flag](#reserved-memory-flag)). In this case, the Node Map becomes updated to reflect this reservation as illustrated in [Memory Manager KEP: Memory Maps at start-up (with examples)][5].

### Runtime

Reference [Memory Manager KEP: Memory Maps at runtime (with examples)][6] illustrates how a successful pod deployment affects the Node Map, and it also relates to how potential Out-of-Memory (OOM) situations are handled further by Kubernetes or operating system.

Important topic in the context of Memory Manager operation is the management of NUMA groups. Each time pod's memory request is in excess of single NUMA node capacity, the Memory Manager attempts to create a group that comprises several NUMA nodes and features extend memory capacity. The problem has been solved as elaborated in [Memory Manager KEP: How to enable the guaranteed memory allocation over many NUMA nodes?][3]. Also, reference [Memory Manager KEP: Simulation - how the Memory Manager works? (by examples)][1] illustrates how the management of groups occurs. 

## Memory Manager configuration

Other Managers should be first pre-configured (section [Pre-configuration](#pre-configuration)). Next, the Memory Manger feature should be enabled (section [Enable the Memory Manager feature](#enable-the-memory-manager-feature)) and be run with `static` policy (section [static policy](#static-policy)). Optionally, some amount of memory can be reserved for system or kubelet processes to increase node stability (section [Reserved memory flag](#reserved-memory-flag)).

### Pre-configuration

To align memory resources with other requested resources in a Pod Spec:
- the CPU Manager should be enabled and proper CPU Manager policy should be configured on a Node. See [control CPU Management Policies](/docs/tasks/administer-cluster/cpu-management-policies/);
- the Topology Manager should be enabled and proper Topology Manager policy should be configured on a Node. See [control Topology Management Policies](/docs/tasks/administer-cluster/topology-manager/).

### Enable the Memory Manager feature 

Support for the Memory Manager requires `MemoryManager` [feature gate](/docs/reference/command-line-tools-reference/feature-gates/) to be enabled. 

That is, the `kubelet` must be started with the following flag:

`--feature-gates=MemoryManager=true`

### Policies 

Memory Manager supports two policies. You can select a policy via a `kubelet` flag `--memory-manager-policy`.

Two policies can be selected:

* `none` (default)
* `static`

#### none policy {#policy-none}

This is the default policy and does not affect the memory allocation in any way.
It acts the same as if the Memory Manager is not present at all.

The `none` policy returns default topology hint. This special hint denotes that Hint Provider (Memory Manger in this case) has no preference for NUMA affinity with any resource.

#### static policy {#policy-static}

In the case of the `Guaranteed` pod, the `static` Memory Manger policy returns topology hints relating to the set of NUMA nodes where the memory can be guaranteed.

In the case of the `BestEffort` or `Burstable` pod, the `static` Memory Manager policy sends back the default topology hint as there is no request for the guaranteed memory.  

### Reserved memory flag

[Node Allocatable Feature](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/) is commonly used by node administrators to reserve K8S node system resources for the kubelet or operating system processes in order to enhance the node stability. A dedicated set of flags can be used for this purpose to set the total amount of reserved memory for a node. This pre-configured value is subsequently utilized to calculate the real amount of node's "allocatable" memory available to pods. Also, K8S scheduler incorporates "allocatable" to optimise pod scheduling process. The foregoing flags include `--kube-reserved`, `--system-reserved` and `--eviction-threshold`. The sum of their values will account for the total amount of reserved memory.    

A new `--reserved-memory` flag was added to Memory Manager to allow for this total reserved memory to be split (by a node administrator) and accordingly reserved across many NUMA nodes. 

Syntax: 

`--reserved-memory=[{numa-node=int,type=string,limit=string}][,][...]`
* `numa-node` index, e.g. `0`
* `type` of memory:
  * `memory` - conventional memory
  * `hugepages-2Mi` or `hugepages-1Gi` - hugepages 
* `limit` - the amount of reserved memory, e.g. `1Gi`

Example usage:

`--reserved-memory={numa-node=0,type=memory,limit=1Gi},{numa-node=1,type=memory,limit=2Gi}`

When you specify values for `--reserved-memory` flag, you must comply with the setting that you prior provided via Node Allocatable Feature flags. That is, the following rule must be obeyed for each memory type: 

`sum(reserved-memory(i)) = kube-reserved + system-reserved + eviction-threshold`, 

where `i` is an index of a NUMA node. 

If you do not follow the formula above, the Memory Manager will show an error on startup.

In other words, the example above illustrates that for the conventional memory (`type=memory`), we reserve `3Gi` in total, i.e.: 

`sum(reserved-memory(i)) = reserved-memory(0) + reserved-memory(1) = 1Gi + 2Gi = 3Gi`

An example of Node Allocatable Feature flags [configuration](https://kubernetes.io/docs/tasks/administer-cluster/out-of-resource/):
* `--kube-reserved=cpu=500m,memory=50Mi`
* `--system-reserved=cpu=123m,memory=333Mi`
* `--eviction-hard=memory.available<500Mi`

_NOTICE: hard eviction threshold is not equal to zero by default but `100Mi`, so do not forget to decrease the total amount set via `--reserved-memory` by this `100Mi`. Otherwise, the Memory Manager will display an error. Here is an example of a correct configuration:_

```
--feature-gates=MemoryManager=true 
--kube-reserved=cpu=4,memory=4Gi 
--system-reserved=cpu=1,memory=1Gi 
--memory-manager-policy=static 
--reserved-memory={numa-node=0,type=memory,limit=3Gi},{numa-node=1,type=memory,limit=2148Mi}
```
Let us validate the configuration above:
1. `kube-reserved + system-reserved + eviction-hard(default) = reserved-memory(0) + reserved-memory(1)`
2. `4Gi + 1Gi + 100Mi = 3Gi + 2148Mi`
3. `5120Mi + 100Mi = 3072Mi + 2148Mi`
4. `5220Mi = 5220Mi` (correct!)

## Placing a Pod in the Guaranteed QoS class

If the selected policy is anything other than `none`, Memory Manager would consider Pods in Guaranteed QoS class and provide topology hints to the Topology Manager. Otherwise, the Memory Manager returns default topology hint to the Topology Manager.

The following pod specifications lend a Pod to Guaranteed QoS class.

```yaml
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      limits:
        memory: "200Mi"
        cpu: "2"
        example.com/device: "1"
      requests:
        memory: "200Mi"
        cpu: "2"
        example.com/device: "1"
```

This pod with integer CPU request runs in the `Guaranteed` QoS class because `requests` are equal to `limits`.

```yaml
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      limits:
        memory: "200Mi"
        cpu: "300m"
        example.com/device: "1"
      requests:
        memory: "200Mi"
        cpu: "300m"
        example.com/device: "1"
```

Also, this pod with sharing CPU request runs in the `Guaranteed` QoS class because `requests` are equal to `limits`.

Notice that both CPU and memory requests must be specified for a Pod to lend it to Guaranteed QoS class.

## Troubleshooting

The following means can be used to troubleshoot the reason why a pod could not be deployed or became rejected at a node:
- pod status - indicates topology affinity errors
- system logs - include valuable information for debugging, e.g., about generated hints
- state file - the dump of internal state of the Memory Manager (includes [Node Map and Memory Maps][2]) 

### Pod status (TopologyAffinityError)

This error typically occurs in the following situations:
* a node has not enough resources available to satisfy the pod's request
* the pod's request is rejected due to particular Topology Manager policy constraints 

The error appears in the status of a pod:
```
# kubectl get pods
NAME         READY   STATUS                  RESTARTS   AGE
guaranteed   0/1     TopologyAffinityError   0          113s
```

Use `kubectl describe pod <id>` or `kubectl get events` to obtain detailed error message:
```
Warning  TopologyAffinityError  10m   kubelet, dell8  Resources cannot be allocated with Topology locality
```

### System logs

Investigate system logs with respect to a particular pod:

```
# journalctl -u kubelet | grep 821c5b92-a74f-4677-aa7d-acfbf70287ab
```

Sample output:
```
Nov 17 10:57:41 dell8 kubelet[52570]: I1117 10:57:41.086255   52570 topology_manager.go:198] [topologymanager] TopologyHints for pod 'guaranteed_default(821c5b92-a74f-4677-aa7d-acfbf70287ab)', container 'guaranteed': map[cpu:[{01 true} {10 true} {11 false}]]
Nov 17 10:57:41 dell8 kubelet[52570]: I1117 10:57:41.086311   52570 topology_manager.go:198] [topologymanager] TopologyHints for pod 'guaranteed_default(821c5b92-a74f-4677-aa7d-acfbf70287ab)', container 'guaranteed': map[memory:[{11 true}]]
```

The set of hints that Memory Manager generated for the pod is shown in the output, i.e. `map[memory:[{11 true}]]`. Also, the set of hints generated by CPU Manager is present, i.e. `map[cpu:[{01 true} {10 true} {11 false}]]`.

Topology Manager merges these hint to calculate single best hint shown below.

Sample output:
```
Nov 18 10:41:22 dell8 kubelet[7227]: I1118 10:41:22.451426    7227 scope_container.go:53] [topologymanager] Best TopologyHint for (pod: gpod_default(2a97a5c6-0577-4956-9f61-153b735e95f6) container: gpod): {01 true}
```

The best hint (`{01 true}`) indicates where to allocate all the resources. Topology Manager tests this hint against its current policy, and based on the verdict, it either admits the pod to the node or rejects it.  

Also, the grep might be used to search for occurrence of `[memorymanager]` in the logs:
```
Nov 03 13:23:00 dell8 kubelet[48363]: I1103 13:23:00.347272   48363 memory_manager.go:204] [memorymanager] Set container "046927bb304a02c1969d6389b28c6f279069e4d91bf6f726e028d714c06db0b1" cpuset.mems to "0"
```

### State file

Let us first deploy a sample `Guaranteed` pod whose specification is as follows:
```
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed
spec:
  containers:
  - name: guaranteed
    image: consumer
    imagePullPolicy: Never
    resources:
      limits:
        cpu: "2"
        memory: 150Gi
      requests:
        cpu: "2"
        memory: 150Gi
    command: ["sleep","infinity"]
``` 

Next, let us log into the node where it was deployed and examine the state file in `/var/lib/kubelet/memory_manager_state`:
```
{
   "policyName":"static",
   "machineState":{
      "0":{
         "numberOfAssignments":1,
         "memoryMap":{
            "hugepages-1Gi":{
               "total":0,
               "systemReserved":0,
               "allocatable":0,
               "reserved":0,
               "free":0
            },
            "memory":{
               "total":134987354112,
               "systemReserved":3221225472,
               "allocatable":131766128640,
               "reserved":131766128640,
               "free":0
            }
         },
         "nodes":[
            0,
            1
         ]
      },
      "1":{
         "numberOfAssignments":1,
         "memoryMap":{
            "hugepages-1Gi":{
               "total":0,
               "systemReserved":0,
               "allocatable":0,
               "reserved":0,
               "free":0
            },
            "memory":{
               "total":135286722560,
               "systemReserved":2252341248,
               "allocatable":133034381312,
               "reserved":29295144960,
               "free":103739236352
            }
         },
         "nodes":[
            0,
            1
         ]
      }
   },
   "entries":{
      "fa9bdd38-6df9-4cf9-aa67-8c4814da37a8":{
         "guaranteed":[
            {
               "numaAffinity":[
                  0,
                  1
               ],
               "type":"memory",
               "size":161061273600
            }
         ]
      }
   },
   "checksum":4142013182
}
```

It can be deduced from the state file that the pod was affined to both NUMA nodes, i.e.:

```
"numaAffinity":[
   0,
   1
],
``` 

This automatically implies that Memory Manager instantiated a new group that comprises these two NUMA nodes, i.e. `0` and `1` indexed NUMA nodes. 

Notice that the management of groups is handled in a relatively complex manner, and further elaboration is provided in Memory Manager KEP in [this][1] and [this][3] sections.

In order to analyse memory resources available in a group, the corresponding entries from NUMA nodes belonging to the group must be added up.  

For example, the total amount of free "conventional" memory in the group can be computed by adding up the free memory available at every NUMA node in the group, i.e., in the `"memory"` section of NUMA node `0` (`"free":0`) and NUMA node `1` (`"free":103739236352`). So, the total amount of free "conventional" memory in this group is equal to `0 + 103739236352` bytes.  

The line `"systemReserved":3221225472` indicates that the administrator of this node reserved `3221225472` bytes (i.e. `3Gi`) to serve kubelet and system processes at NUMA node `0`, by using `--reserved-memory` flag.

## Resources

- [Memory Manager KEP: Design Overview][4] 

- [Memory Manager KEP: Memory Maps at start-up (with examples)][5]

- [Memory Manager KEP: Memory Maps at runtime (with examples)][6]

- [Memory Manager KEP: Simulation - how the Memory Manager works? (by examples)][1]

- [Memory Manager KEP: The Concept of Node Map and Memory Maps][2]

- [Memory Manager KEP: How to enable the guaranteed memory allocation over many NUMA nodes?][3]

[1]: https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/1769-memory-manager#simulation---how-the-memory-manager-works-by-examples
[2]: https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/1769-memory-manager#the-concept-of-node-map-and-memory-maps
[3]: https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/1769-memory-manager#how-to-enable-the-guaranteed-memory-allocation-over-many-numa-nodes
[4]: https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/1769-memory-manager#design-overview
[5]: https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/1769-memory-manager#memory-maps-at-start-up-with-examples
[6]: https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/1769-memory-manager#memory-maps-at-runtime-with-examples