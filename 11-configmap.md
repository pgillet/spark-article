# ConfigMap

We use a [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/) for setting Spark configuration
 data separately from the driver pod definition.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app-name: spark-${PRIORITY_CLASS_NAME}${NAME_SUFFIX}
  name: spark-${PRIORITY_CLASS_NAME}${NAME_SUFFIX}-cm
  namespace: ${NAMESPACE}
data:
  # Comma-separated list of .zip, .egg, or .py files dependencies for Python apps.
  # spark.submit.pyFiles can be used instead in spark-defaults.conf below.
  # PYTHONPATH: ...
  spark-env.sh: |
    #!/usr/bin/env bash

    export DUMMY=dummy
    echo "Here we are, ${DUMMY}!"
  spark-defaults.conf: |
    spark.master                                k8s://https://kubernetes.default
    spark.submit.deployMode                     client

    spark.executor.instances                    2
    spark.executor.cores                        1
    spark.executor.memory                       512m

    spark.kubernetes.executor.container.image   eu.gcr.io/yippee-spark-k8s/spark-py:3.0.1
    spark.kubernetes.container.image.pullPolicy IfNotPresent
    spark.kubernetes.namespace                  spark-jobs
    # Must match the mount path of the ConfigMap volume in driver pod
    spark.kubernetes.executor.podTemplateFile   /spark-conf/executor-pod-template.yaml
    spark.kubernetes.pyspark.pythonVersion      2
    spark.kubernetes.driver.pod.name            spark-${PRIORITY_CLASS_NAME}${NAME_SUFFIX}-driver

    spark.driver.host                           spark-${PRIORITY_CLASS_NAME}${NAME_SUFFIX}-driver-svc
    spark.driver.port                           5678

    # Config params for use with an ingress to expose the Web UI
    spark.ui.proxyBase                          /spark-${PRIORITY_CLASS_NAME}${NAME_SUFFIX}
    # spark.ui.proxyRedirectUri                   http://<load balancer static IP address>
  executor-pod-template.yaml: |
    apiVersion: v1
    kind: Pod
    metadata:
      labels:
        app-name: spark-${PRIORITY_CLASS_NAME}${NAME_SUFFIX}
        spark-role: executor
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: type
                operator: In
                values: [${EXECUTOR_NODE_AFFINITIES}]
      priorityClassName: ${PRIORITY_CLASS_NAME}
      schedulerName: volcano
```

## Executor Pod Template

We use a template file in the `ConfigMap` to define the executor pod configuration. 
[Templates](https://spark.apache.org/docs/latest/running-on-kubernetes.html#pod-template) are typically used for
 configuring Spark pods that cannot be configured otherwise, via Spark properties or environment variables
 . Thus, template files mostly contain fine-grained configuration related to deployment at the Kubernetes level: 
here, the node affinity and the priority class name.

To make the pod template file accessible to the spark-submit process, we must set the Spark property 
`spark.kubernetes.executor.podTemplateFile` with its local pathname in the driver pod. To do so, the file will be 
automatically mounted onto a volume in the driver pod when it’s created.

# Object Names and Labels

We use the `app-name` label to semantically group all the Kubernetes resources related to a single Spark application.
In our case, this label provides uniqueness, and we do not expect multiple Spark applications to carry the same value 
for this label (at least in the _namespace-time_ considered).

For a given Spark application, all the object names are then derived from the `app-name`. We simply add a suffix which 
also qualifies the type of the object: `-driver` for the driver pod, `-driver-svc` for the driver service, `-ui-svc` 
for the Spark UI service, `-ui-ingress` for the Spark UI ingress, and `-cm` for the ConfigMap.

We also set the label `spark-role` at the pod level to differentiate the drivers from their executors.

This naming and labeling is consistent with the Spark operator. As a result, Spark applications can be treated and 
filtered in the same way, whether they are triggered by the Spark operator or by spark-submit. :thumbsup:

