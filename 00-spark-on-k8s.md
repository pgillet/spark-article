# My Journey With Spark On Kubernetes... In Python

_I am talking about a time that under 20s cannot know_. Until not long ago, the way to go to run Spark on a 
cluster was either with Spark's own standalone cluster manager, Mesos or YARN. In the meantime, the Kingdom of 
Kubernetes has risen and spread widely.

And when it comes to run Spark on Kubernetes, you have now two choices:

- Use "native" Spark's Kubernetes capabilities: Spark can run on clusters managed by Kubernetes since Spark 2.3.
  Kubernetes support was still flagged as experimental until very recently, but as per 
  [SPARK-33005 Kubernetes GA Preparation](https://issues.apache.org/jira/browse/SPARK-33005), Spark on Kubernetes is 
  now fully supported and production ready! 🎉

- Use the Spark Operator, proposed and maintained by Google, which is still in beta version (and always will be).
