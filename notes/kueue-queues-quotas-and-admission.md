# Kueue: Queues, Quotas, and Workload Admission

Kueue is a Kubernetes-native system for deciding which workloads can be admitted and when, based on queue policy and resource quota. The Kubernetes scheduler places admitted Pods on nodes. This overview follows a workload from submission through admission, quota sharing, priority, ResourceFlavors, and Deployment integration.

## Admission is not scheduling

Kueue checks queue policy and quota to decide whether a workload can proceed. It does not choose the particular node where a Pod runs. After Kueue admits a workload—and selects a ResourceFlavor when configured—the Kubernetes scheduler places its Pods on eligible nodes.

**Key distinction:** Kueue controls workload admission; the Kubernetes scheduler controls Pod placement.

## A LocalQueue is an entry point

A **LocalQueue** is a namespaced resource that groups a team's workloads and gives them a way to submit work to Kueue. It points to a **ClusterQueue**. A LocalQueue does not have its own quota or independent admission-ordering policy; the ClusterQueue determines how its workloads are evaluated.

A **ClusterQueue** is cluster-scoped. It defines the resource quota and queueing policy Kueue uses when considering workloads. Workloads from one or more LocalQueues can point to the same ClusterQueue.

**Key distinction:** The LocalQueue identifies where a team's work enters Kueue; the ClusterQueue owns the admission quota and ordering policy.

## Shared and separate ClusterQueues

When multiple teams' LocalQueues point to one ClusterQueue, they draw from the same quota. One team's admitted workloads can use all of it, leaving another team's workloads waiting. Separate LocalQueues do not provide separate resource guarantees.

Giving each team a separate ClusterQueue can protect its nominal quota from use by other queues. The trade-off is that unused quota in one ClusterQueue is not automatically available to another.

A **Cohort** groups ClusterQueues so they can borrow unused nominal quota from one another. Borrowing makes idle capacity available without merging the queues or erasing their nominal quotas. If the lending ClusterQueue later needs that quota, whether borrowed workloads can be reclaimed depends on the configured preemption policy. Reclaim can stop borrowed work and return it to its queue.

## Priority affects admission

A **WorkloadPriorityClass** gives a workload a priority for Kueue's admission decisions. Within a ClusterQueue, a higher-priority pending workload can be considered before lower-priority work, even if the latter arrived earlier.

Priority may also enable preemption: if the ClusterQueue's policy allows it, a higher-priority workload can preempt lower-priority work that is already admitted. The displaced workload returns to the queue. Preemption is policy-controlled; priority does not mean every running workload can always be interrupted.

## ResourceFlavors describe capacity choices

A **ResourceFlavor** represents a variation in capacity and associates it with compatible nodes, for example through node labels or taints. A ClusterQueue can define separate quotas for multiple flavors.

For illustration, one ClusterQueue defines two finite CPU budgets: 8 CPUs of x86 capacity and 4 CPUs of ARM capacity. It lists x86 first, expressing a preference for that flavor. A workload that supports either architecture can be admitted against the x86 quota when it fits. If it cannot fit within the remaining x86 quota, Kueue can consider ARM, provided the workload is compatible and ARM quota is available. The quotas are separate: unused ARM quota does not become extra x86 quota.

After Kueue selects a flavor, the flavor's constraints help direct the Pods to compatible nodes. The Kubernetes scheduler still chooses the particular node. Kueue does not make an x86-only workload compatible with ARM; a workload that may use either architecture must actually support both.

## Deployment Pods share ClusterQueue quota

With Kueue's Pod integration configured, Deployment Pods can be admitted individually. The Pods exist, but Kueue uses scheduling gates to keep unadmitted Pods from being scheduled. If quota permits two replicas but not a third, two Pods can be admitted while the third waits.

Deployment Pods and batch Jobs can use quota from the same ClusterQueue. Long-running replicas continue to hold quota, so a batch Job may wait. If an operator scales the Deployment down and the removed Pod terminates, its quota can become available for other work. Kueue does not decide the Deployment's desired replica count.

**Key distinction:** Kueue gates each Deployment Pod for admission; it does not suspend the Deployment as a whole.

## In short

**Submit through a LocalQueue → evaluate in a ClusterQueue → apply quota, flavor, and queueing policy → admit or wait → let the Kubernetes scheduler place admitted Pods on eligible nodes.**
