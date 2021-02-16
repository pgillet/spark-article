We need to operate Kubernetes as part of a Python client application. So, we need to interact with the Kubernetes
 REST API. Luckily we do not need to implement the API calls and manage HTTP requests/responses ourselves: we
  can rely on the [Kubernetes Python client](https://github.com/kubernetes-client/python), among other officially
  -supported Kubernetes client libraries for other languages such as Go, Java, dotnet, JavaScript and Haskell (there
   are also a lot of community-maintained client libraries for many languages).

When using the Kubernetes Python Client library, we must first load authentication and cluster information.

# Service Account and Role Binding

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

# The Easy Way

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

# The Hard Way

## Fetch credentials

Here, we're going to configure the Python client in the most programmatic way possible.  
First, we need to fetch the credentials to access the Kubernetes cluster. We’ll store these in python environmental 
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

## Python sample usage

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