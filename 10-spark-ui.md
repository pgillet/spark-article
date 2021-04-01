# Ingress

The Spark UI is accessible by creating a service of type `ClusterIP` which exposes the UI from the driver pod:

`spark-ui-svc.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app-name: spark-${PRIORITY_CLASS_NAME}${NAME_SUFFIX}
  name: spark-${PRIORITY_CLASS_NAME}${NAME_SUFFIX}-ui-svc
  namespace: spark-jobs
spec:
  ports:
  - name: spark-driver-ui-port
    port: 4040
    protocol: TCP
    targetPort: 4040
  selector:
    app-name: spark-${PRIORITY_CLASS_NAME}${NAME_SUFFIX}
    spark-role: driver
  type: ClusterIP
```

With this service alone, the UI is only accessible from inside the cluster. We must then create an `Ingress` to expose 
the UI outside the cluster. In order for the Ingress resource to work, the cluster must have an ingress controller 
running. You can choose the 
[ingress controller implementation](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) 
that best fits your cluster. 
[Traefik](https://doc.traefik.io/traefik/providers/kubernetes-ingress/) and 
[Nginx](https://kubernetes.github.io/ingress-nginx/) are very popular choices. The ingress below is configured for 
Nginx: 

`spark-ui-ingress.yaml`
```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  labels:
    app-name: spark-${PRIORITY_CLASS_NAME}${NAME_SUFFIX}
  name: spark-${PRIORITY_CLASS_NAME}${NAME_SUFFIX}-ui-ingress
  namespace: spark-jobs
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/proxy-redirect-from: "http://$host/"
    nginx.ingress.kubernetes.io/proxy-redirect-to: "http://$host/spark-${PRIORITY_CLASS_NAME}${NAME_SUFFIX}/"
spec:
  rules:
  - http:
      paths:
      - path: /spark-${PRIORITY_CLASS_NAME}${NAME_SUFFIX}(/|$)(.*)
        backend:
          serviceName: spark-${PRIORITY_CLASS_NAME}${NAME_SUFFIX}-ui-svc
          servicePort: 4040
```

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
 
With hostname wildcards, and therefore without the HTTP redirect, the UI service could be switched to `NodePort` 
type (a NodePort service exposes the Service on each Node's IP at a static port) and still be compatible with the 
Ingress.
The UI would thus be accessible through both the Ingress with its external URL configured, and the `NodePort` 
service at `http://<node-ip>:<service-port>`. A service of type `NodePort` is still relevant in a private cloud.