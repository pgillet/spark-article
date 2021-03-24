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

```python
import binascii
import os
from os import listdir
from pprint import pprint

import yaml
from kubernetes import config, utils
from kubernetes.client import ApiClient


def create_k8s_object(yaml_file=None, env_subst=None):
    with open(yaml_file) as f:
        str = f.read()
        if env_subst:
            for env in env_subst:
                str = str.replace(env, env_subst[env])
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

    k8s_client = ApiClient()
    verbose = True

    # Create driver pod
    k8s_dir = os.path.join(os.path.dirname(__file__), "k8s/spark-native")
    k8s_object_dict = create_k8s_object(os.path.join(k8s_dir, "pyspark-pi-driver-pod.yaml"), env_subst)
    pprint(k8s_object_dict)
    k8s_objects = utils.create_from_dict(k8s_client, k8s_object_dict, verbose=verbose)

    # Prepare ownership on dependent objects
    owner_refs = [{"apiVersion": "v1",
                   "controller": True,
                   "kind": "Pod",
                   "name": k8s_objects[0].metadata.name,
                   "uid": k8s_objects[0].metadata.uid}]

    # List all YAML files in k8s/spark-native directory, except the driver pod definition file
    other_resources = listdir(k8s_dir)
    other_resources.remove("pyspark-pi-driver-pod.yaml")
    for f in other_resources:
        k8s_object_dict = create_k8s_object(os.path.join(k8s_dir, f), env_subst)
        # Set ownership
        k8s_object_dict["metadata"]["ownerReferences"] = owner_refs
        pprint(k8s_object_dict)
        utils.create_from_dict(k8s_client, k8s_object_dict, verbose=verbose)

    print("Submitted %s" % (k8s_objects[0].metadata.labels["app-name"]))

if __name__ == "__main__":
    main()
```

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