# Service Account for Driver Pods

Remember, Spark applications run as independent sets of processes on a cluster, coordinated by the `SparkContext`
object in your main program, called the `driver`. Once connected, Spark acquires `executors` on nodes in the cluster,
which are processes that run computations and store data for your application.

Thus, Spark driver pods need a Kubernetes service account in the pod's namespace that has permissions to create, get, 
list, and delete executor pods. Below an example RBAC setup that creates a driver service account named `yippee-spark` in
 the namespace `spark-jobs`, with a RBAC role binding giving the service account the needed permissions.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: yippee-spark
  namespace: spark-jobs
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: spark-jobs
  name: spark-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["*"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: spark-role-binding
  namespace: spark-jobs
subjects:
- kind: ServiceAccount
  name: yippee-spark
  namespace: spark-jobs
roleRef:
  kind: Role
  name: spark-role
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl create namespace spark-jobs
kubectl create -f k8s/yippee-spark-rbac.yaml
```

# Node Affinity

By default, the scheduler automatically places pods on nodes by ensuring nodes have sufficient free resources
, distributing pods evenly across nodes, etc.
But there are circumstances where you may want more control on a node where a pod lands, for example to ensure that a
 pod ends up on a memory or compute-optimized machine, or with an SSD attached to it.
In our case, Spark executors need more resources than drivers. We thus need to constrain driver pods and executor
 pods to only be able to run on particular node(s). We will use 
 [Node Affinities](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) with 
 [label selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) to make the selection.

Execute the following command for the node(s) intended to execute driver pods:

```bash
kubectl label nodes <node-name> type=driver
```

For executor pods:

```bash
kubectl label nodes <node-name> type=compute
```

# Pod Priority and Preemption

In my project, we aim to run multiple Spark jobs simultaneously in parallel. But some workloads have higher priority
 than others. If a job cannot be scheduled, the scheduler (here, Volcano) tries to preempt (evict) lower priority
  Pods to make scheduling of the pending Pod possible.
To use priority and preemption capabilities, we must first create the necessary `PriorityClasses`:

`priorities.yaml`
```yaml
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: routine
value: 2
preemptionPolicy: Never
globalDefault: false
description: "Routine priority"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: urgent
value: 10
preemptionPolicy: Never
globalDefault: false
description: "Urgent priority"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: exceptional
value: 50
preemptionPolicy: Never
globalDefault: false
description: "Exceptional priority"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: rush
value: 100
preemptionPolicy: PreemptLowerPriority
globalDefault: false
description: "Rush priority"
```

```bash
kubectl create -f k8s/priorities.yaml
```

Here, only the PriorityClass "rush" is allowed to preempt lower-priority pods. Pods with other priorities will be 
placed in the scheduling queue ahead of lower-priority pods, but they cannot preempt other pods. They'll just have 
to wait until sufficient resources are free to be scheduled.