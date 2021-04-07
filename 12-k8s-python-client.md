# Kubernetes Python Client

We need to operate Kubernetes as part of a Python client application. So, we need to interact with the Kubernetes
 REST API. Luckily we do not need to implement the API calls and manage HTTP requests/responses ourselves: we
  can rely on the [Kubernetes Python client](https://github.com/kubernetes-client/python), among other 
officially-supported Kubernetes client libraries for other languages such as Go, Java, dotnet, JavaScript and 
Haskell (there are also a lot of community-maintained client libraries for many languages).

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
[Github repository](https://githubcom/kubernetes-client/python/blob/master/examples/deployment_crud.py)), we create, 
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


def main():
    config.load_kube_config("path/to/kubeconfig_file")

    with open(path.join(path.dirname(__file__), "nginx-deployment.yaml")) as f:
        dep = yaml.safe_load(f)
        k8s_apps_v1 = client.AppsV1Api()
        resp = k8s_apps_v1.create_namespaced_deployment(
            body=dep, namespace="default")
        print("Deployment created. status='%s'" % resp.metadata.name)


if __name__ == '__main__':
    main()
```

This is the equivalent in Python of `kubectl create -f nginx-deployment.yaml`.

As you can see, you must call `create_namespaced_deployment` to create a Deployment. In the same way, you would 
call `create_namespaced_pod` to create a Pod, and so on. This is because the Python client is automatically 
generated following the `OpenAPI` specifications of the Kubernetes API.

It's a shame to have to call a specific method to create a particular type of object, even though the type of object 
itself is already specified in the manifest that we load through this method. This is also true for 
[custom resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/), that 
means object types that are not part of the core Kubernetes API, typically `SparkApplication` from the Spark Operator.
Indeed, you can pass a custom resource as a `Dict` to the `create_namespaced_custom_object` function, but you still 
need to requalify the object type in the other required arguments (see this 
[example](https://github.com/kubernetes-client/python/blob/master/examples/custom_object.py)).

Luckily, the Kubernetes Python Client provides a utility method that acts as an input hub for any kind of object.

```python
from os import path

import yaml

from kubernetes import client, config, utils


def main():
    config.load_kube_config("path/to/kubeconfig_file")

    with open(path.join(path.dirname(__file__), "nginx-deployment.yaml")) as f:
        dep = yaml.safe_load(f)
        k8s_client = client.ApiClient()
        resp = utils.create_from_dict(k8s_client, dep)
        print("Deployment created. status='%s'" % resp[0].metadata.name)


if __name__ == '__main__':
    main()
```

`utils.create_from_dict` is the magic method here. It only takes a `dict` holding valid kubernetes objects. It is a 
blessing to have found it, because it is well hidden in the client and not documented at all.


