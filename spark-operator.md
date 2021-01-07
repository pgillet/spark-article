# Configuring and installing the Kubernetes Operator for Apache Spark

In this section, you use [Helm](https://github.com/kubernetes/helm) to deploy the [Kubernetes Operator for Apache Spark](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator) from the incubator [Chart](https://github.com/helm/charts/tree/master/incubator/sparkoperator) repository. Helm is a package manager you can use to configure and deploy Kubernetes apps.

## Install Helm

1. Downlad and install the `Helm` binary:

```bash
wget https://get.helm.sh/helm-v3.3.4-linux-amd64.tar.gz
```

2. Unzip the file to your local system:

```bash
tar zxfv helm-v2.12.3-linux-amd64.tar.gz
cp linux-amd64/helm .
```

3. Ensure that Helm is properly installed by running the following command:

```bash
./helm version
```

If Helm is correctly installed, you should see the following output:

```bash
version.BuildInfo{Version:"v3.3.4", GitCommit:"a61ce5633af99708171414353ed49547cf05013d", GitTreeState:"clean", GoVersion:"go1.14.9"}
```

## Install the chart

```bash
helm repo add incubator http://storage.googleapis.com/kubernetes-charts-incubator
kubectl create namespace spark-operator
helm install spark-operator incubator/sparkoperator --namespace spark-operator --set enableWebhook=true --set enableBatchScheduler=true
```

The flag`enableBatchScheduler=true` enables Volcano. To install the operator with Vocano enabled, you must also install 
the mutating admission webhook with the flag `enableWebhook=true`.

Now you should see the operator running in the cluster by checking the status of the Helm release:


```bash
./helm status spark-operator --namespace spark-operator
```

## About the Spark Job Namespace and the Service Account for Driver Pods

We did not set a specific value for the Helm chart property `sparkJobNamespace` when installing the operator, that means 
the Spark Operator supports deploying `SparkApplications` to all namespaces.
As a consequence, the Spark Operator did not automatically create the service account for driver pods, and we must set 
up the RBAC for driver pods of our `SparkApplications` to be able to manipulate executor pods in a specific namespace.

See [Service Account for Driver Pods](../docs/gke.md#service-account-for-driver-pods) to set up the RBAC for driver 
pods.

See [About the Spark Job Namespace](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/blob/master/docs/quick-start-guide.md#about-the-spark-job-namespace) and [About the Service Account for Driver Pods](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/blob/master/docs/quick-start-guide.md#about-the-service-account-for-driver-pods) sections for more details.

# Running the Examples

To run the Spark Pi example provided within the operator, run the following command:

```bash
kubectl apply -f examples/spark-py-pi.yaml
```

According to our [configuration](../docs/gke.md#service-account-for-driver-pods), `.metadata.namespace` must be set to 
"spark-jobs" and  `.spec.driver.serviceAccount` is set to the name of the service account "hippi-spark" previously 
created.

# Limitations (in my humble opinion)

Some Spark properties are related to "deployment", typically set through configuration file or `spark-submit` command line options with "native" Spark. 
These properties will not be applied if passed directly to `.spec.sparkConf` in the `SparkApplication` custom resource. Indeed, `.spec.sparkConf` is only intended for properties that affect Spark runtime control, like `spark.task.maxFailures`.

**Example:**
Setting `spark.executor.instances` in `.spec.sparkConf` will not affect the number of executors. Instead, we have to set the field `.spec.executor.instances` in the `SparkApplication` yaml file.

It would be nice if we could set/override such properties in `.spec.sparkConf`. Thus, we could easily "templatize" a SparkApplication and set runtime parameters with Spark semantics. In other terms, we should be able to move the cursor as we want between native Spark and Spark Operator semantics.

The concerned properties identified so far:

- spark.kubernetes.driver.request.cores
- spark.kubernetes.executor.request.cores
- spark.kubernetes.executor.deleteOnTermination
- spark.driver.cores
- spark.executor.cores
- spark.executor.instances
- spark.kubernetes.container.image
- spark.kubernetes.driver.container.image
- spark.kubernetes.executor.container.image
- spark.kubernetes.container.image.pullPolicy

`spark.submit.pyFiles` and `spark.jars` may also be concerned. If they are, it is a problem as these properties are multi-valued: `.spec.deps.pyFiles` must be an array of strings, while the Spark property is only a string containing comma-separated Python dependencies, and in this case it is not easy to switch from the Spark semantics to the Spark Operator logic...

At the time of writing this manual, an [issue](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/issues/1109) has been opened in the Spark Operator Github repository. Case to follow...

# See also

- [Kubernetes Operator for Apache Spark - Quick Start Guide](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/blob/master/docs/quick-start-guide.md)
- [Integration with Volcano for Batch Scheduling](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/blob/master/docs/volcano-integration.md)
