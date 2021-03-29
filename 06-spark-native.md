Let's take a closer look at the Pi example from the Spark Operator. A single yaml file is needed:

```yaml
apiVersion: "sparkoperator.k8s.io/v1beta2"
kind: SparkApplication
metadata:
  name: pyspark-pi
  namespace: spark-jobs
spec:
  batchScheduler: volcano
  batchSchedulerOptions:
    priorityClassName: routine
  type: Python
  pythonVersion: "2"
  mode: cluster
  image: "gcr.io/spark-operator/spark-py:v3.0.0"
  imagePullPolicy: Always
  mainApplicationFile: local:///opt/spark/examples/src/main/python/pi.py
  sparkVersion: "3.0.0"
  restartPolicy:
    type: OnFailure
    onFailureRetries: 3
    onFailureRetryInterval: 10
    onSubmissionFailureRetries: 5
    onSubmissionFailureRetryInterval: 20
  timeToLiveSeconds: 86400
  driver:
    affinity:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: type
              operator: In
              values: [driver]
    cores: 1
    coreLimit: "1200m"
    memory: "512m"
    labels:
      version: 3.0.0
    serviceAccount: yippee-spark
  executor:
    affinity:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: type
              operator: In
              values: [compute]
    cores: 1
    instances: 2
    memory: "512m"
    labels:
      version: 3.0.0
```

Pretty simple, right!?

The Spark Operator aims to make specifying and running Spark applications in a cloud-native way and as easy and 
idiomatic as running other workloads on Kubernetes. It uses Kubernetes 
[custom resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) for 
specifying, running, and monitoring Spark applications.

With the high-level resource `SparkApplication`, the operator greatly reduces the boilerplate YAML configuration 
files and takes care of all the needed plumbing for you: networking between the driver and its executors, garbage 
collection, pod configuration, access to the driver UI. 

In use, the operator is way much easier than `spark-submit`. But spark-submit is definitely not going away and is 
still the Spark native way of launching applications. In the long term, for application submission, the operator 
will not semantically nor functionally diverge from spark-submit and will always use it under the hood. More 
importantly, the spark-submit script use all of Spark’s supported cluster managers through a uniform interface so 
you don’t have to configure your application especially for each one (still, that shouldn't prevent the Apache 
Spark project from developing its own operator in my opinion).

Eventually, choosing between the Spark Operator and spark-submit is a matter of if you are more _Kubernetes_-centric 
and you run Spark workloads among other types of workloads, or you do Spark first, and Kubernetes is just a mean to 
allocate resources on a cluster.

In the rest of this article, we will see how the magic of the Spark operator operates, by reproducing all of its 
internals with spark-submit.


