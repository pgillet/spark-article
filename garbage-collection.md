# Killed applications
The role of the Kubernetes garbage collector is to delete certain objects that once had an owner, but no longer have 
one. The goal is to make sure that the garbage collector properly deletes resources that are no longer needed when
 killing a Spark application.
It is important to free up the resources of the k8s cluster when you are going to run tens / hundreds of Spark 
applications in parallel.

For this, certain Kubernetes objects can be declared owners of other objects. "Owned" objects are called *dependent* on  
the owner object. Each dependent object has a `metadata.ownerReferences` field that points to the owning object. When  
deleting an owner object, all dependent objects are also automatically deleted (*cascading deletion*) by default.

Example of an executor pod owned by its driver pod:
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    spark-role: executor
  ownerReferences:
    - apiVersion: v1
      controller: true
      kind: Pod
      name: pyspark-pi-routine-0245dc3d340cd533-driver
      uid: 3b10fa97-c847-4fce-b3e1-71f779cffbef
...
```

The Spark Operator automatically sets the value of `ownerReference`, at different levels: the custom `SparkApplication` 
resource owns the driver pod which owns its executors.

For applications that are submitted *natively* (without the Spark Operator), the highest level owner object is the 
driver pod: the executor pods automatically set the `ownerReference` field, pointing to the driver pod. But we must 
manage the ownership relationship ourselves for the other `ConfigMap`, `Service` and `Ingress` resources.
For this, we must retrieve the auto-generated `uid` of the newly created driver pod and inject it into the dependent 
objects: it is impossible to manually set the `uid` in the YAML definition files, this can only be done at runtime 
through code.

# Applications normally completed
When an application completes normally, the executor pods terminate and are cleaned up, but the driver pod persists
 logs and remains in "completed" state in the Kubernetes API "until it's eventually garbage collected or cleaned up
 manually".

Note that in the completed state, the driver pod does not use any compute or memory resources.

The Spark Operator has TTL support for `SparkApplications` through the optional field named `.spec.timeToLiveSeconds`,  
which if set, defines the Time-To-Live (TTL) duration in seconds for a `SparkAplication` after its termination. The  
`SparkApplication` object will be garbage collected if the current time is more than the `.spec.timeToLiveSeconds` since  
its termination. The example below illustrates how to use the field:

```yaml
spec:
  timeToLiveSeconds: 86400
```

On the native Spark side, there is nothing in the doc that specifies how driver pods are ultimately deleted.  
We could set up a simple Kubernetes [`CronJob`](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)  
that would run periodically to delete them automatically.

At the time of writing this manual, there are pending requests in Kubernetes to support TTL in `Pods` like in  
`Jobs`: _"TTL controller only handles Jobs for now, and may be expanded to handle other resources that will finish  
execution, such as Pods and custom resources."_

# Background cascading deletion
When killing an application from the Python code, we delete the owner object using the *Background* cascading
 deletion policy.
In background cascading deletion, Kubernetes deletes the owner object immediately and the garbage collector then
 deletes the dependents in the background. This is useful so as not to delay the main execution thread.