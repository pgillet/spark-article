# Service Account for Driver Pods

Spark driver pods need a Kubernetes service account in the pod's namespace that has permissions to create, get,
list, and delete executor pods. Below an example RBAC setup that creates a driver service account named `yippee-spark`  
in the namespace `spark-jobs`, with a RBAC role binding giving the service account the needed permissions.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: yippee-spark
  namespace: spark-jobs
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: spark-jobs
  name: spark-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["*"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: spark-role-binding
  namespace: spark-jobs
subjects:
- kind: ServiceAccount
  name: yippee-spark
  namespace: spark-jobs
roleRef:
  kind: Role
  name: spark-role
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl create namespace spark-jobs
kubectl create -f k8s/yippee-spark-rbac.yaml
```

# Node Affinity

You can constrain driver pods and executor pods to only be able to run on particular node(s).

Execute the following command for the node(s) intended to execute driver pods:

```bash
kubectl label nodes <node-name> type=driver
```

For executor pods:

```bash
kubectl label nodes <node-name> type=compute
```

# Pod Priority and Preemption

To use priority and preemption capabilities, create the necessary `PriorityClasses`:

```bash
kubectl create -f k8s/priorities.yaml
```
