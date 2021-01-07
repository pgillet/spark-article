# Ingress Controller

Install [Traefik](https://doc.traefik.io/traefik/providers/kubernetes-ingress/) or 
[Nginx](https://kubernetes.github.io/ingress-nginx/) as Ingress Controller for your Kubernetes cluster.

# Ingress

The Spark UI is accessible by creating a service of type `ClusterIP` which exposes the UI from the driver pod (see 
[spark-ui-svc.yaml](../spark_client/kubernetes/k8s/spark-native/spark-ui-svc.yaml)). This is only accessible from
 within the cluster. We must then create an Ingress to expose the UI outside the cluster (see 
 [spark-ui-ingress.yaml](../spark_client/kubernetes/k8s/spark-native/spark-ui-ingress.yaml)).
 
The Ingress is backed by as many services as driver pods that run concurrently in the cluster. Each UI must then be
 addressed with a unique `path` in the Ingress. The path in question is directly derived from the name of the Spark
  application, with its unique ID. 
  
The interface is always available at `http://<driver-pod-ip>:4040` (from inside the cluster), as each driver pod is
 assigned a unique IP and there is only one `SparkContext` running in the pod host (otherwise, the UI would be bound
  to successive ports beginning with 4040: 4041, 4042, etc).
  
So, natively, the different UIs that might be available at a given time on a same host are only differentiated by their
 assigned port number. The UI has been designed following that principle, and all HTTP operations are performed
  relatively to the same root path in the URL, which is...`/`.
  
So for the UI to be effectively accessible through the Ingress, we must set up HTTP redirection with an alternative
 root path (the one configured in the Ingress). To work smoothly, the UI itself must be aware of this redirection by
  setting `spark.ui.proxyBase` to this root path...and that's it!

## Spark Operator

The operator supports creating an optional Ingress for the Spark Web UI. This can be turned on by setting the
 `ingress-url-format` command-line flag. The `ingress-url-format` should be a template like 
 `{{$appName}}.{ingress_suffix}/{{$appNamespace}}/{{$appName}}`. The `{ingress_suffix}` should be replaced by the user
  to indicate the cluster's Ingress url and the operator will replace the `{{$appName}}` and `{{$appNamespace}}` with
   the appropriate value. Please note that Ingress support requires that cluster's Ingress url routing is correctly
    setup. For e.g. if the `ingress-url-format` is `{{$appName}}.ingress.cluster.com`, it requires that anything
     `*.ingress.cluster.com` should be routed to the Ingress controller on the K8s cluster.
     
Example of Spark Operator install with the `ingress-url-format` command-line flag:

```bash
./helm install spark-operator incubator/sparkoperator --namespace spark-operator --set enableWebhook=true --set enableBatchScheduler=true --set ingressUrlFormat="\{\{\$appName\}\}.ingress.cluster.com"
```

Note that curly braces must be escaped.

# Future Work

The Spark Operator Ingress could not be enabled during the development, as a DNS name is required. Instead, the same
 Ingress as for native Spark is "grafted" to the `SparkApplication`, with path-based routing.
 
The operation proposed by the Spark Operator, with routing based on hostname wildcards (for example 
`*.ingress.cluster.com`), is nevertheless interesting as it would overcome the problem of HTTP redirect described
 above.
 
With hostname wildcards, and therefore without the HTTP redirect, the UI service could be switched (back) to `NodePort` 
type and still be compatible with the Ingress.
The UI would thus be accessible through both the Ingress and the `NodePort` service at 
`http://<node-ip>:<service-port>`. A service of `NodePort` type is still relevant in a private cloud.