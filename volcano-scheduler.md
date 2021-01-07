[Volcano](https://github.com/volcano-sh/volcano) is a batch scheduler for Kubernetes, well-suited for scheduling Spark 
applications pods with a better efficiency than the default [kube-scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/).

# Install Volcano

Install Volcano on the cluster:

```bash
kubectl apply -f https://raw.githubusercontent.com/volcano-sh/volcano/master/installer/volcano-development.yaml
```

# Enable job preemption

The preempt action is responsible for preemptive scheduling of high priority tasks in the same queue according to 
priority rules.

This action is disabled by default in Volcano.
To enable job preemption, edit the Volcano configuration as follows:

```bash
kubectl edit configmap volcano-scheduler-configmap --namespace volcano-system
```

And add `preempt` to the list of `actions`:

```yaml
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  volcano-scheduler.conf: |
    actions: "enqueue, allocate, preempt, backfill"
    tiers:
    - plugins:
      - name: priority
      - name: gang
      - name: conformance
    - plugins:
      - name: drf
      - name: predicates
      - name: proportion
      - name: nodeorder
      - name: binpack
kind: ConfigMap
metadata:
  ...

```

Note that job preemption in Volcano relies on the priority plugin that compares the priorities of two jobs or tasks. For 
two jobs, it decides whose priority is higher by comparing `job.spec.priorityClassName`. For two tasks, it decides whose 
priority is higher by comparing `task.priorityClassName`, `task.createTime`, and `task.id` in order.

# Enable Volcano scheduling in your workload

For your workload to be scheduled by Volcano, you just need to set `schedulerName: volcano` in your pod's `spec` (or
 `batchScheduler: volcano` in the `SparkApplication`'s `spec` if you use the Spark Operator). By default, the
  workload is scheduled with the default `kube-scheduler`.

**To be consistent, ensure that the same scheduler is used for driver and executor pods.**


