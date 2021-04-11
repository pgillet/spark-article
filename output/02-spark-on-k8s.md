We need to operate Kubernetes as part of a Python client application. So, we need to interact with the Kubernetes
 REST API. Luckily we do not need to implement the API calls and manage HTTP requests/responses ourselves: we
  can rely on the [Kubernetes Python client](https://github.com/kubernetes-client/python), among other 
officially-supported Kubernetes client libraries for other languages such as Go, Java, .NET, JavaScript and 
Haskell (there are also a lot of community-maintained client libraries for many languages).

<!--more-->

# Kubernetes Python Client

When using the Kubernetes Python Client library, we must first load authentication and cluster information.

## Load Authentication And Cluster Information

First, you need to setup the required service account and roles.

`k8s/python-client-sa-rbac.yaml`
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: python-client-sa
  namespace: spark-jobs
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: spark-jobs
  name: python-client-role
rules:
- apiGroups: [""]
  resources: ["configmaps", "pods", "pods/log", "pods/status", "services"]
  verbs: ["*"]
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses", "ingresses/status"]
  verbs: ["*"]
- apiGroups: ["sparkoperator.k8s.io"]
  resources: [sparkapplications]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: python-client-role-binding
  namespace: spark-jobs
subjects:
- kind: ServiceAccount
  name: python-client-sa
  namespace: spark-jobs
roleRef:
  kind: Role
  name: python-client-role
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: python-client-cluster-role-binding
subjects:
- kind: ServiceAccount
  name: python-client-sa
  namespace: spark-jobs
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl create -f k8s/python-client-sa-rbac.yaml
```

This command creates a new service account named `python-client-sa`, a new role with the needed permissions in the 
`spark-jobs` namespace and then binds the new role to the newly created service account.

**WARNING**: The `python-client-sa` is the service account that will provide the identity for the Kubernetes Python 
Client in our application. Do not confuse this service account with the `yippee-spark` service account for 
driver pods.

### The Easy Way

In this method, we can use an helper utility to load authentication and cluster information from a `kubeconfig` file and
 store them in `kubernetes.client.configuration`.

```python
from kubernetes import config, client

config.load_kube_config("path/to/kubeconfig_file")

v1 = client.CoreV1Api()

print("Listing pods with their IPs:")

ret = v1.list_namespaced_pod(namespace="spark-jobs")
for i in ret.items:
    print("%s\t%s\t%s" % (i.status.pod_ip, i.metadata.namespace, i.metadata.name))
```

But we **DO NOT** want to rely on the default `kubeconfig` file, denoted by the environment variable `KUBECONFIG` or
, failing that, in `~/.kube/config`. This `kubeconfig` file is yours, as user of the `kubectl` command. Concretely
, with this `kubeconfig` file, you have the right to do almost everything in the Kubernetes cluster, and in all 
namespaces. Instead, we're going to generate one especially for the service account created above, with the help of 
the script `kubeconfig-gen.sh`:

```bash
#!/usr/bin/env bash

# set -eux

# Reads the API server name from the default `kubeconfig` file.
# Here we suppose that the kubectl command-line tool is already configured to communicate with our cluster.
APISERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')

SERVICE_ACCOUNT_NAME=${1:-python-client-sa}
NAMESPACE=${2:-spark-jobs}
SECRET_NAME=$(kubectl get serviceaccount ${SERVICE_ACCOUNT_NAME} -n ${NAMESPACE} -o jsonpath='{.secrets[0].name}')
TOKEN=$(kubectl get secret ${SECRET_NAME} -n ${NAMESPACE} -o jsonpath='{.data.token}' | base64 --decode)
CACERT=$(kubectl get secret ${SECRET_NAME} -n ${NAMESPACE} -o jsonpath="{['data']['ca\.crt']}")


cat > kubeconfig-sa << EOF
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: ${CACERT}
    server: ${APISERVER}
  name: default-cluster
contexts:
- context:
    cluster: default-cluster
    namespace: ${NAMESPACE}
    user: ${SERVICE_ACCOUNT_NAME}
  name: default-context
current-context: default-context
users:
- user:
    token: ${TOKEN}
  name: ${SERVICE_ACCOUNT_NAME}
EOF
```

The `kubeconfig` file thus created configures access to the cluster for the `python-client-sa` service account, with
 only the rights needed for our client application and in the single namespace `spark-jobs` (_"principle of least
  privilege"_).

### The Hard Way

#### Fetch credentials

Here, we're going to configure the Python client in the most programmatic way possible.  
First, we need to fetch the credentials to access the Kubernetes cluster. We’ll store these in Python environmental
variables.

```bash
export APISERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
SECRET_NAME=$(kubectl get serviceaccount python-client-sa -o jsonpath='{.secrets[0].name}')
export TOKEN=$(kubectl get secret ${SECRET_NAME} -o jsonpath='{.data.token}' | base64 --decode)
export CACERT=$(kubectl get secret ${SECRET_NAME} -o jsonpath="{['data']['ca\.crt']}")
```

Note that environment variables are captured the first time the `os` module is imported, typically during IDE/Python 
startup. Changes to the environment made after this time are not reflected in `os.environ` (except for changes made by 
modifying os.environ directly).

#### Python sample usage

```python
import base64
import os
from tempfile import NamedTemporaryFile

from kubernetes import client

api_server = os.environ["APISERVER"]
cacert = os.environ["CACERT"]
token = os.environ["TOKEN"]

# Set the configuration
configuration = client.Configuration()
with NamedTemporaryFile(delete=False) as cert:
    cert.write(base64.b64decode(cacert))
    configuration.ssl_ca_cert = cert.name
configuration.host = api_server
configuration.verify_ssl = True
configuration.debug = False
configuration.api_key = {"authorization": "Bearer " + token}
client.Configuration.set_default(configuration)

v1 = client.CoreV1Api()

print("Listing pods with their IPs:")

ret = v1.list_namespaced_pod(namespace="spark-jobs")
for i in ret.items:
    print("%s\t%s\t%s" % (i.status.pod_ip, i.metadata.namespace, i.metadata.name))
```

## Getting Started

### Kubernetes Object Management

With the Kubernetes Python Client, you can create and manage Kubernetes objects programmatically.

In the following example (provided in the 
[GitHub repository](https://githubcom/kubernetes-client/python/blob/master/examples/deployment_crud.py)), we create, 
update then delete a `Deployment` using `AppsV1Api`:

```python
"""
Creates, updates, and deletes a deployment using AppsV1Api.
"""

from kubernetes import client, config

DEPLOYMENT_NAME = "nginx-deployment"


def create_deployment_object():
    # Configureate Pod template container
    container = client.V1Container(
        name="nginx",
        image="nginx:1.15.4",
        ports=[client.V1ContainerPort(container_port=80)],
        resources=client.V1ResourceRequirements(
            requests={"cpu": "100m", "memory": "200Mi"},
            limits={"cpu": "500m", "memory": "500Mi"}
        )
    )
    # Create and configurate a spec section
    template = client.V1PodTemplateSpec(
        metadata=client.V1ObjectMeta(labels={"app": "nginx"}),
        spec=client.V1PodSpec(containers=[container]))
    # Create the specification of deployment
    spec = client.V1DeploymentSpec(
        replicas=3,
        template=template,
        selector={'matchLabels': {'app': 'nginx'}})
    # Instantiate the deployment object
    deployment = client.V1Deployment(
        api_version="apps/v1",
        kind="Deployment",
        metadata=client.V1ObjectMeta(name=DEPLOYMENT_NAME),
        spec=spec)

    return deployment


def create_deployment(api_instance, deployment):
    # Create deployement
    api_response = api_instance.create_namespaced_deployment(
        body=deployment,
        namespace="default")
    print("Deployment created. status='%s'" % str(api_response.status))


def update_deployment(api_instance, deployment):
    # Update container image
    deployment.spec.template.spec.containers[0].image = "nginx:1.16.0"
    # Update the deployment
    api_response = api_instance.patch_namespaced_deployment(
        name=DEPLOYMENT_NAME,
        namespace="default",
        body=deployment)
    print("Deployment updated. status='%s'" % str(api_response.status))


def delete_deployment(api_instance):
    # Delete deployment
    api_response = api_instance.delete_namespaced_deployment(
        name=DEPLOYMENT_NAME,
        namespace="default",
        body=client.V1DeleteOptions(
            propagation_policy='Foreground',
            grace_period_seconds=5))
    print("Deployment deleted. status='%s'" % str(api_response.status))


def main():
    # Configs can be set in Configuration class directly or using helper
    # utility. If no argument provided, the config will be loaded from
    # default location.
    config.load_kube_config("path/to/kubeconfig_file")
    apps_v1 = client.AppsV1Api()

    deployment = create_deployment_object()

    create_deployment(apps_v1, deployment)

    update_deployment(apps_v1, deployment)

    delete_deployment(apps_v1)


if __name__ == '__main__':
    main()
```

This is great, but this involves mastering the client's API, and above all we must configure our objects 
_imperatively_: we specify the desired operation (create, replace, etc.) on Python objects that represent Kubernetes 
objects. Here, we prefer to manage our objects in a _declarative_ way and operate on object configuration files (stored 
locally along the Python code source) like we usually do with the `kubectl` command. Indeed, Python code should only 
be a simple execution backend to trigger Kubernetes operations, and _business_ logic, so to speak, should be 
concentrated in manifest files.

The deployment we created above is the same as in the `nginx-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.15.4
        ports:
        - containerPort: 80
```

We can directly load the manifest as follows:

```python
from os import path
import yaml
from kubernetes import client, config


config.load_kube_config("path/to/kubeconfig_file")

with open(path.join(path.dirname(__file__), "nginx-deployment.yaml")) as f:
    dep = yaml.safe_load(f)
    k8s_apps_v1 = client.AppsV1Api()
    resp = k8s_apps_v1.create_namespaced_deployment(
        body=dep, namespace="default")
    print("Deployment created. status='%s'" % resp.metadata.name)
```

This is the equivalent in Python of `kubectl create -f nginx-deployment.yaml`.

As you can see, you must call `create_namespaced_deployment` to create a Deployment. In the same way, you would 
call `create_namespaced_pod` to create a Pod, and so on. This is because the Python client is automatically 
generated following the `OpenAPI` specifications of the Kubernetes API.

It's a shame to have to call a specific method to create a particular type of object, even though the type of object 
itself is already specified in the manifest that we load through this method. Luckily, the Kubernetes Python Client 
provides a utility method that acts as an input hub for any kind of object.

```python
import os
import yaml
from kubernetes import client, config, utils


config.load_kube_config("path/to/kubeconfig_file")

with open(os.path.join(os.path.dirname(__file__), "nginx-deployment.yaml")) as f:
    dep = yaml.safe_load(f)
    k8s_client = client.ApiClient()
    resp = utils.create_from_dict(k8s_client, dep)
    print("Deployment created. status='%s'" % resp[0].metadata.name)
```

`utils.create_from_dict` is the magic method here. It only takes a `Dict` holding valid kubernetes objects. It is a 
blessing to have found it, because it is well hidden in the client and not documented at all.

So, to launch a Spark job with spark-submit, you could just call the code snippet above with a single YAML file which 
groups all the needed resources (separated by --- in YAML). 

But what about the Spark Operator? `utils.create_from_dict` does not support 
[custom resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/), that 
means object types that are not part of the core Kubernetes API, namely `SparkApplication` from the Spark Operator.
To run a Spark job with the Spark Operator, you have no other choice than calling the 
`create_namespaced_custom_object` function of `CustomObjectsApi`:

```python
import os
import yaml
from kubernetes import client, config, utils


config.load_kube_config("path/to/kubeconfig_file")

with open(os.path.join(os.path.dirname(__file__), "k8s/spark-operator/pyspark-pi.yaml")) as f:
    dep = yaml.safe_load(f)
    custom_object_api = client.CustomObjectsApi()

    custom_object_api.create_namespaced_custom_object(
        group="sparkoperator.k8s.io",
        version="v1beta2",
        namespace="spark-jobs",
        plural="sparkapplications",
        body=dep,
    )
    print("SparkApplication created")
```

Regarding spark-submit, there is an even more direct method `utils.create_from_yaml`, which reads Kubernetes objects 
from a YAML file. But we cannot use it, as we need to "parameterize" our YAML files before submitting them to the 
Kubernetes Python client.

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

Let's get back to Spark (native). We saw earlier the YAML file that defines a driver pod to run the Pi example 
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

We just iterate over the YAML files that define these resources and just call the same method `utils.create_from_dict`:

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
application completes normally.

# Garbage collection

## Killed applications
The role of the Kubernetes garbage collector is to delete certain objects that once had an owner, but no longer have 
one. The goal is to make sure that the garbage collector properly deletes resources that are no longer needed when
 killing a Spark application.
It is important to free up the resources of the Kubernetes cluster when you are going to run tens / hundreds of Spark 
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
through code (and that's why we cannot put all the resources in a single YAML file).

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

## Applications normally completed
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

On the native Spark side, there is nothing in the doc that specifies how driver pods are ultimately deleted. We 
could set up a simple Kubernetes [`CronJob`](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
that would run periodically to delete them automatically.

At the time of writing this article, there are pending requests in Kubernetes to support TTL in `Pods` like in  
`Jobs`: _"TTL controller only handles Jobs for now, and may be expanded to handle other resources that will finish  
execution, such as Pods and custom resources."_

## Background cascading deletion
When killing an application from the Python code, we delete the owner object using the *Background* cascading
 deletion policy.
In background cascading deletion, Kubernetes deletes the owner object immediately and the garbage collector then
 deletes the dependents in the background. This is useful so as not to delay the main execution thread.

To delete a Spark job launched by `spark-submit`:

```python
from kubernetes import client


core_v1_api = client.CoreV1Api()
core_v1_api.delete_namespaced_pod("driver-pod-name", "spark-jobs", propagation_policy="Background")
```

To delete a Spark job launched with the Spark Operator, we must delete the enclosing `SparkApplication` resource:

```python
from kubernetes import client, config

custom_object_api = client.CustomObjectsApi()
custom_object_api.delete_namespaced_custom_object(
    group="sparkoperator.k8s.io",
    version="v1beta2",
    namespace="spark-jobs",
    plural="sparkapplications",
    name="app_name",
    propagation_policy="Background")
```

# Pod priority & preemption

Whether it is Volcano or the default `kube-scheduler`, job preemption relies on job priorities. For two jobs, the 
scheduler decides whose priority is higher by comparing `.spec.priorityClassName` (then `createTime`).

The priority is propagated to driver and executor pods, whether with native spark-submit or with Spark Operator, and
 regardless of the node affinities.

## How to know which pods have been preempted

You can retrieve high-level information on what is happening in the cluster. To list all events in the namespace 
`spark-jobs` you can use:

```bash
# List Events sorted by timestamp
kubectl get events --sort-by=.metadata.creationTimestamp --namespace=spark-jobs
```

<pre>
LAST SEEN   TYPE      REASON             OBJECT                                                   MESSAGE
69s         Normal    Scheduled          podgroup/podgroup-6083b4f9-f240-4a0e-95f2-06882aec2942   pod group is ready
93s         Normal    Scheduled          pod/pyspark-pi-driver-routine-bf20cae50b6a8253           Successfully assigned spark-jobs/pyspark-pi-driver-routine-bf20cae50b6a8253 to gke-yippee-spark-k8s-clus-default-pool-0b72dd1d-jpfp
92s         Normal    Started            pod/pyspark-pi-driver-routine-bf20cae50b6a8253           Started container pyspark-pi
92s         Normal    Pulled             pod/pyspark-pi-driver-routine-bf20cae50b6a8253           Container image "eu.gcr.io/yippee-spark-k8s/spark-py:3.0.1" already present on machine
92s         Normal    Created            pod/pyspark-pi-driver-routine-bf20cae50b6a8253           Created container pyspark-pi
82s         Normal    Scheduled          pod/pythonpi-34b3597593d246a1-exec-2                     Successfully assigned spark-jobs/pythonpi-34b3597593d246a1-exec-2 to gke-yippee-spark-k8s-clus-default-pool-0b72dd1d-pvdl
82s         Normal    Scheduled          pod/pythonpi-34b3597593d246a1-exec-1                     Successfully assigned spark-jobs/pythonpi-34b3597593d246a1-exec-1 to gke-yippee-spark-k8s-clus-default-pool-0b72dd1d-wxck
82s         Normal    Created            pod/pythonpi-34b3597593d246a1-exec-1                     Created container spark-kubernetes-executor
82s         Normal    Started            pod/pythonpi-34b3597593d246a1-exec-1                     Started container spark-kubernetes-executor
82s         Normal    Pulled             pod/pythonpi-34b3597593d246a1-exec-1                     Container image "eu.gcr.io/yippee-spark-k8s/spark-py:3.0.1" already present on machine
81s         Normal    Created            pod/pythonpi-34b3597593d246a1-exec-2                     Created container spark-kubernetes-executor
81s         Normal    Pulled             pod/pythonpi-34b3597593d246a1-exec-2                     Container image "eu.gcr.io/yippee-spark-k8s/spark-py:3.0.1" already present on machine
80s         Normal    Started            pod/pythonpi-34b3597593d246a1-exec-2                     Started container spark-kubernetes-executor
42s         Normal    Killing            pod/pythonpi-34b3597593d246a1-exec-1                     Stopping container spark-kubernetes-executor
42s         Normal    Killing            pod/pythonpi-34b3597593d246a1-exec-2                     Stopping container spark-kubernetes-executor
42s         Warning   FailedScheduling   pod/pyspark-pi-driver-urgent-bb25fc0b8efe7c4d            all nodes are unavailable: 3 node(s) resource fit failed.
<b>42s         Normal    Killing            pod/pyspark-pi-driver-routine-bf20cae50b6a8253           Stopping container pyspark-pi
42s         Warning   Evict              pod/pyspark-pi-driver-routine-bf20cae50b6a8253           Pod is evicted, because of preempt</b>
34s         Warning   Unschedulable      podgroup/podgroup-ee4b9210-35c2-4d68-841a-2daf7712a816   0/1 tasks in gang unschedulable: pod group is not ready, 1 Pipelined, 1 minAvailable.
41s         Warning   FailedScheduling   pod/pyspark-pi-driver-urgent-bb25fc0b8efe7c4d            1/1 tasks in gang unschedulable: pod group is not ready, 1 Pipelined, 1 minAvailable.
18s         Normal    Scheduled          podgroup/podgroup-ee4b9210-35c2-4d68-841a-2daf7712a816   pod group is ready
33s         Normal    Scheduled          pod/pyspark-pi-driver-urgent-bb25fc0b8efe7c4d            Successfully assigned spark-jobs/pyspark-pi-driver-urgent-bb25fc0b8efe7c4d to gke-yippee-spark-k8s-clus-default-pool-0b72dd1d-jpfp
32s         Normal    Started            pod/pyspark-pi-driver-urgent-bb25fc0b8efe7c4d            Started container pyspark-pi
32s         Normal    Created            pod/pyspark-pi-driver-urgent-bb25fc0b8efe7c4d            Created container pyspark-pi
32s         Normal    Pulled             pod/pyspark-pi-driver-urgent-bb25fc0b8efe7c4d            Container image "eu.gcr.io/yippee-spark-k8s/spark-py:3.0.1" already present on machine
22s         Normal    Scheduled          pod/pythonpi-f36cce7593d332f1-exec-1                     Successfully assigned spark-jobs/pythonpi-f36cce7593d332f1-exec-1 to gke-yippee-spark-k8s-clus-default-pool-0b72dd1d-wxck
22s         Normal    Scheduled          pod/pythonpi-f36cce7593d332f1-exec-2                     Successfully assigned spark-jobs/pythonpi-f36cce7593d332f1-exec-2 to gke-yippee-spark-k8s-clus-default-pool-0b72dd1d-pvdl
22s         Normal    Pulled             pod/pythonpi-f36cce7593d332f1-exec-1                     Container image "eu.gcr.io/yippee-spark-k8s/spark-py:3.0.1" already present on machine
21s         Normal    Started            pod/pythonpi-f36cce7593d332f1-exec-1                     Started container spark-kubernetes-executor
21s         Normal    Created            pod/pythonpi-f36cce7593d332f1-exec-1                     Created container spark-kubernetes-executor
21s         Normal    Pulled             pod/pythonpi-f36cce7593d332f1-exec-2                     Container image "eu.gcr.io/yippee-spark-k8s/spark-py:3.0.1" already present on machine
21s         Normal    Created            pod/pythonpi-f36cce7593d332f1-exec-2                     Created container spark-kubernetes-executor
21s         Normal    Started            pod/pythonpi-f36cce7593d332f1-exec-2                     Started container spark-kubernetes-executor
</pre>

In the output above, we can see that the pod `pyspark-pi-driver-routine-bf20cae50b6a8253` has been "evicted because of 
preempt".

## Future work

Kubernetes provides containers with lifecycle hooks. The hooks enable containers to be aware of events in their
 management lifecycle and run code implemented in a handler when the corresponding lifecycle hook is executed.

In particular, the `PreStop` hook can be called immediately before a container is terminated due to preemption (among
other events).
Thus, we can consider an action, whatever it is, to be triggered in case of preemption. All you need to do is
implement and register a handler for this hook.

See [Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/).

We are soon coming to the end of our journey. Before wishing you good night, let's take a look at how we can monitor 
a Spark application from Python code.

# Monitoring

## Getting the status of a Spark application

Many operations in the Kubernetes Python client can be _watched_. This allows our Python program to watch for 
changes in a specific resource until you get the desired result or the watch expires.

Here, we want to monitor the lifecycle of the pod driver, starting in the `Pending` phase, moving through `Running` 
if the Spark container starts OK, and then through either the `Succeeded` or `Failed` phases: 

```python
from kubernetes import client, watch

app_name = 'pyspark-pi-routine-bf20cae50b6a8253'
v1 = client.CoreV1Api()
    
count = 2
w = watch.Watch()
label_selector = 'app-name=%s,spark-role=driver' % app_name
for event in w.stream(v1.list_namespaced_pod, namespace='spark-jobs', label_selector=label_selector, timeout_seconds=60):
    print('Event: %s' % event['object'].status.phase)
    count -= 1
    if not count:
        w.stop()
```

Here, the driver pod (hence, the Spark application) is expected to complete, successfully or not, within 60 seconds or 
less. Its status should only change twice during this period: ideally `Pending` > `Running` > `Succeeded`. 

## Getting the logs

It is possible to retrieve the logs of the driver pod and mix them into those of the host application.
Getting the logs is just as easy:

```python
from kubernetes import client, watch
from threading import Thread

v1 = client.CoreV1Api()
pod_name = 'pyspark-pi-routine-bf20cae50b6a8253-driver'

def logs(pod_name):
    w = watch.Watch()
    for event in w.stream(v1.read_namespaced_pod_log, pod_name=pod_name, namespace='spark-jobs', _request_timeout=300):
        yield event

# We surely don't want to block the main thread while reading the logs
def consumer():
    for log in logs(pod_name):
        print(log)

t = Thread(target=consumer)
t.start()
```

## Getting the Ingress URI

Once a Spark application has started, the ingress (at least, its public IP address) that exposes the Spark UI may 
take a while before it becomes available. Here too, we can monitor the Ingress resource as follows:

```python
from kubernetes import client, watch

app_name = 'pyspark-pi-routine-bf20cae50b6a8253'

networking_v1_beta1_api = client.NetworkingV1beta1Api()
w = watch.Watch()
label_selector = 'app-name=%s' % app_name
for event in w.stream(networking_v1_beta1_api.list_namespaced_ingress, namespace=namespace,
                      label_selector=label_selector,
                      timeout_seconds=30):
    ingress = event['object'].status.load_balancer.ingress
    if ingress:
        external_ip = ingress[0].ip
        print('Event: The Spark Web UI is available at http://%s/%s' % (external_ip, app_name))
        w.stop()
    else:
        print('Event: Ingress not yet available')
```

# Conclusion

What a hell of a journey!

We have seen that the Python code for launching or deleting Spark applications is slightly different depending on 
whether we are using the Spark operator or spark-submit. But since we name and label the Kubernetes objects 
consistently between the two, and as we set the ownership relationships properly, we can monitor our Spark 
applications and manage their lifecycle equally. 

Would you like to know more? The Python scripts explained in this last article are available in this [GitHub 
repository](https://github.com/pgillet/k8s-python-client-examples). Serve yourself.