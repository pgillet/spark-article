Whether it is Volcano or the default `kube-scheduler`, job preemption relies on job priorities. For two jobs, the 
scheduler decides whose priority is higher by comparing `.spec.priorityClassName` (then `createTime`).

The priority is propagated to driver and executor pods, whether with native spark-submit or with Spark Operator, and regardless of the node affinities.

# How to know which pods have been preempted

```bash
# List Events sorted by timestamp
kubectl get events --sort-by=.metadata.creationTimestamp
```

<pre>
LAST SEEN   TYPE      REASON             OBJECT                                                   MESSAGE
69s         Normal    Scheduled          podgroup/podgroup-6083b4f9-f240-4a0e-95f2-06882aec2942   pod group is ready
93s         Normal    Scheduled          pod/pyspark-pi-driver-routine-bf20cae50b6a8253           Successfully assigned spark-jobs/pyspark-pi-driver-routine-bf20cae50b6a8253 to gke-hippi-spark-k8s-clus-default-pool-0b72dd1d-jpfp
92s         Normal    Started            pod/pyspark-pi-driver-routine-bf20cae50b6a8253           Started container pyspark-pi
92s         Normal    Pulled             pod/pyspark-pi-driver-routine-bf20cae50b6a8253           Container image "eu.gcr.io/hippi-spark-k8s/spark-py:3.0.1" already present on machine
92s         Normal    Created            pod/pyspark-pi-driver-routine-bf20cae50b6a8253           Created container pyspark-pi
82s         Normal    Scheduled          pod/pythonpi-34b3597593d246a1-exec-2                     Successfully assigned spark-jobs/pythonpi-34b3597593d246a1-exec-2 to gke-hippi-spark-k8s-clus-default-pool-0b72dd1d-pvdl
82s         Normal    Scheduled          pod/pythonpi-34b3597593d246a1-exec-1                     Successfully assigned spark-jobs/pythonpi-34b3597593d246a1-exec-1 to gke-hippi-spark-k8s-clus-default-pool-0b72dd1d-wxck
82s         Normal    Created            pod/pythonpi-34b3597593d246a1-exec-1                     Created container spark-kubernetes-executor
82s         Normal    Started            pod/pythonpi-34b3597593d246a1-exec-1                     Started container spark-kubernetes-executor
82s         Normal    Pulled             pod/pythonpi-34b3597593d246a1-exec-1                     Container image "eu.gcr.io/hippi-spark-k8s/spark-py:3.0.1" already present on machine
81s         Normal    Created            pod/pythonpi-34b3597593d246a1-exec-2                     Created container spark-kubernetes-executor
81s         Normal    Pulled             pod/pythonpi-34b3597593d246a1-exec-2                     Container image "eu.gcr.io/hippi-spark-k8s/spark-py:3.0.1" already present on machine
80s         Normal    Started            pod/pythonpi-34b3597593d246a1-exec-2                     Started container spark-kubernetes-executor
42s         Normal    Killing            pod/pythonpi-34b3597593d246a1-exec-1                     Stopping container spark-kubernetes-executor
42s         Normal    Killing            pod/pythonpi-34b3597593d246a1-exec-2                     Stopping container spark-kubernetes-executor
42s         Warning   FailedScheduling   pod/pyspark-pi-driver-urgent-bb25fc0b8efe7c4d            all nodes are unavailable: 3 node(s) resource fit failed.
<b>42s         Normal    Killing            pod/pyspark-pi-driver-routine-bf20cae50b6a8253           Stopping container pyspark-pi
42s         Warning   Evict              pod/pyspark-pi-driver-routine-bf20cae50b6a8253           Pod is evicted, because of preempt</b>
34s         Warning   Unschedulable      podgroup/podgroup-ee4b9210-35c2-4d68-841a-2daf7712a816   0/1 tasks in gang unschedulable: pod group is not ready, 1 Pipelined, 1 minAvailable.
41s         Warning   FailedScheduling   pod/pyspark-pi-driver-urgent-bb25fc0b8efe7c4d            1/1 tasks in gang unschedulable: pod group is not ready, 1 Pipelined, 1 minAvailable.
18s         Normal    Scheduled          podgroup/podgroup-ee4b9210-35c2-4d68-841a-2daf7712a816   pod group is ready
33s         Normal    Scheduled          pod/pyspark-pi-driver-urgent-bb25fc0b8efe7c4d            Successfully assigned spark-jobs/pyspark-pi-driver-urgent-bb25fc0b8efe7c4d to gke-hippi-spark-k8s-clus-default-pool-0b72dd1d-jpfp
32s         Normal    Started            pod/pyspark-pi-driver-urgent-bb25fc0b8efe7c4d            Started container pyspark-pi
32s         Normal    Created            pod/pyspark-pi-driver-urgent-bb25fc0b8efe7c4d            Created container pyspark-pi
32s         Normal    Pulled             pod/pyspark-pi-driver-urgent-bb25fc0b8efe7c4d            Container image "eu.gcr.io/hippi-spark-k8s/spark-py:3.0.1" already present on machine
22s         Normal    Scheduled          pod/pythonpi-f36cce7593d332f1-exec-1                     Successfully assigned spark-jobs/pythonpi-f36cce7593d332f1-exec-1 to gke-hippi-spark-k8s-clus-default-pool-0b72dd1d-wxck
22s         Normal    Scheduled          pod/pythonpi-f36cce7593d332f1-exec-2                     Successfully assigned spark-jobs/pythonpi-f36cce7593d332f1-exec-2 to gke-hippi-spark-k8s-clus-default-pool-0b72dd1d-pvdl
22s         Normal    Pulled             pod/pythonpi-f36cce7593d332f1-exec-1                     Container image "eu.gcr.io/hippi-spark-k8s/spark-py:3.0.1" already present on machine
21s         Normal    Started            pod/pythonpi-f36cce7593d332f1-exec-1                     Started container spark-kubernetes-executor
21s         Normal    Created            pod/pythonpi-f36cce7593d332f1-exec-1                     Created container spark-kubernetes-executor
21s         Normal    Pulled             pod/pythonpi-f36cce7593d332f1-exec-2                     Container image "eu.gcr.io/hippi-spark-k8s/spark-py:3.0.1" already present on machine
21s         Normal    Created            pod/pythonpi-f36cce7593d332f1-exec-2                     Created container spark-kubernetes-executor
21s         Normal    Started            pod/pythonpi-f36cce7593d332f1-exec-2                     Started container spark-kubernetes-executor
</pre>

In the output above, we can see that the pod `pyspark-pi-driver-routine-bf20cae50b6a8253` has been "evicted because of 
preempt".

# Future work

Kubernetes provides containers with lifecycle hooks. The hooks enable containers to be aware of events in their
 management lifecycle and run code implemented in a handler when the corresponding lifecycle hook is executed.

In particular, the `PreStop` hook can be called immediately before a container is terminated due to preemption (among
 other events).
Thus, we can consider an action, whatever it is, to be triggered in case of preemtion. All you need to do is
 implement and register a handler for this hook.

See [Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/).

