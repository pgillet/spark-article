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

Let's get back to Spark (native). We saw earlier the yaml file that defines a driver pod to run the Pi example 
program. As you can see, we have placeholders to specify the `namespace`, the `priorityClassName`, the 
`serviceAccountName`, the `nodeAffinity` and a `NAME_SUFFIX` to make the pod's name unique.

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

    # TODO: create the other resources
    
    print("Submitted %s" % (k8s_objects[0].metadata.labels["app-name"]))

if __name__ == "__main__":
    main()
```

# Putting the pieces together

We have now the general mechanics and we can create all the resources needed.
Remember, the driver pod consumes a `ConfigMap` to define environment variables and to mount configuration files 
in the Spark container (including the template for the executor pods). We also have a `Service` that allows 
executors to communicate back with the driver. And finally, we have another `Service`, backed by an `Ingress`, to 
expose the Spark UI.

We just iterate over the yaml files that define these resources and just call the same method `utils.create_from_dict`:

```python
    # List all YAML files in k8s/spark-native directory, except the driver pod definition file
    other_resources = listdir(k8s_dir)
    other_resources.remove("pyspark-pi-driver-pod.yaml")
    for f in other_resources:
        k8s_object_dict = create_k8s_object(os.path.join(k8s_dir, f), env_subst)
        pprint(k8s_object_dict)
        utils.create_from_dict(k8s_client, k8s_object_dict, verbose=verbose)

    print("Submitted %s" % (k8s_objects[0].metadata.labels["app-name"]))
```

Now that we've launched a _full_ Spark application, let's see what happens when we kill it :smiling_imp: or when the 
application 
completes normally.