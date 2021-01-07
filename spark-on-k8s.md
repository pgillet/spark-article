# My Journey With Spark On Kubernetes

When it comes to run Spark on Kubernetes, you have two choices:

- Use "native" Spark Kubernetes capabilities: Spark can run on clusters managed by Kubernetes since Spark 2.3.
  Kubernetes support is still flagged as experimental but it is sufficiently mature to think about running production workloads.

- Use the Spark Operator, proposed and maintained by Google, which is still in beta version (and always will be)
