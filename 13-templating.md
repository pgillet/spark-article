There is an even more direct method `utils.create_from_yaml`, which reads Kubernetes objects from a yaml file.
But we cannot use it, as we need to "parameterize" our yaml files before submitting them to the Kubernetes Python 
client.

# Templating

As you know, when you apply a manifest file to Kubernetes - the YAML-formatted resource descriptions that Kubernetes
 can understand - you must specify the resource name which must be unique for that type of resource (and within the
  same namespace), otherwise Kubernetes will complain that the resource already exists.

For example, you can only have one Pod named `myapp-1234` within the same namespace, but you can have one Pod and one
 Deployment that are each named `myapp-1234`.

As we want to run multiple Spark jobs simultaneously, and as these Spark jobs are mostly identical except for a few
 runtime parameters, we need to parameterize, or _templatize_, our Kubernetes YAML files.
 
Normally, you don't do that, at least that's not in Kubernetes' philosophy: Kubernetes files should be template-free
 and should only be patched, by the means of [Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/) 
 for instance. [Helm](https://helm.sh/) has also its own templating system.  

You cannot use such a tool with the Kubernetes Python client.
Instead, we are going to substitute references to variables of the form `$VAR` or `${VAR}` with the corresponding
 values, exactly like [envsubst](https://www.gnu.org/software/gettext/manual/html_node/envsubst-Invocation.html), but
  programmatically.

Let's get back to Spark. The yaml below defines a driver pod (in native Spark) that runs the Pi example program. As you 
can see, we have placeholders to specify the `namespace`, the `priorityClassName`, the `serviceAccountName`, the 
`nodeAffinity` and a `NAME_SUFFIX` to make the pod's name unique.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app-name: pyspark-pi-${PRIORITY_CLASS_NAME}${NAME_SUFFIX}
    spark-role: driver
  name: pyspark-pi-${PRIORITY_CLASS_NAME}${NAME_SUFFIX}-driver
  namespace: ${NAMESPACE}
spec:
  containers:
  - name: pyspark-pi
    image: eu.gcr.io/yippee-spark-k8s/spark-py:3.0.1
    imagePullPolicy: IfNotPresent
    ports:
    - containerPort: 5678
      name: headless-svc
    - containerPort: 4040
      name: web-ui
    resources:
      requests:
        memory: 512Mi
        cpu: 1
      limits:
        cpu: 1200m
    env:
    # Overriding configuration directory
    - name: SPARK_CONF_DIR
      value: /spark-conf
    - name: SPARK_HOME
      value: /opt/spark
    # Configure all key-value pairs in ConfigMap as container environment variables
    envFrom:
      - configMapRef:
          name: pyspark-pi-${PRIORITY_CLASS_NAME}${NAME_SUFFIX}-cm
    args:
    - $(SPARK_HOME)/bin/spark-submit
    - /opt/spark/examples/src/main/python/pi.py
    - "10"
    volumeMounts:
      - name: spark-config
        mountPath: /spark-conf
        readOnly: true
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
            - key: type
              operator: In
              values: [${DRIVER_NODE_AFFINITIES}]
  priorityClassName: ${PRIORITY_CLASS_NAME}
  restartPolicy: OnFailure
  schedulerName: volcano
  serviceAccountName: ${SERVICE_ACCOUNT_NAME}
  volumes:
    # Add the executor pod template in read-only volume, for the driver to read
    - name: spark-config
      configMap:
        name: pyspark-pi-${PRIORITY_CLASS_NAME}${NAME_SUFFIX}-cm
        items:
        - key: spark-defaults.conf
          path: spark-defaults.conf
        - key: spark-env.sh
          path: spark-env.sh
        - key: executor-pod-template.yaml
          path: executor-pod-template.yaml
```

Now, in the Python code, we replace at runtime these variables with the desired values before creating the pod:

```python
import binascii
import os
from os import listdir
from pprint import pprint

import yaml
from kubernetes import client, config, utils


def create_k8s_object(yaml_file=None, env_subst=None):
    with open(yaml_file) as f:
        str = f.read()
        if env_subst:
            for env, value in env_subst.items():
                str = str.replace(env, value)
        return yaml.safe_load(str)


def main():
    # Configs can be set in Configuration class directly or using helper utility
    config.load_kube_config("path/to/kubeconfig_file")

    name_suffix = "-" + binascii.b2a_hex(os.urandom(8))
    priority_class_name = "routine"
    env_subst = {"${NAMESPACE}": "spark-jobs",
                 "${SERVICE_ACCOUNT_NAME}": "yippee-spark",
                 "${DRIVER_NODE_AFFINITIES}": "driver",
                 "${EXECUTOR_NODE_AFFINITIES}": "compute",
                 "${NAME_SUFFIX}": name_suffix,
                 "${PRIORITY_CLASS_NAME}": priority_class_name}

    k8s_client = client.ApiClient()
    verbose = True

    # Create driver pod
    k8s_dir = os.path.join(os.path.dirname(__file__), "k8s/spark-native")
    k8s_object_dict = create_k8s_object(os.path.join(k8s_dir, "pyspark-pi-driver-pod.yaml"), env_subst)
    pprint(k8s_object_dict)
    k8s_objects = utils.create_from_dict(k8s_client, k8s_object_dict, verbose=verbose)

    print("Submitted %s" % (k8s_objects[0].metadata.labels["app-name"]))

if __name__ == "__main__":
    main()
```
