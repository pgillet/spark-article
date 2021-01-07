This guide explains how to create and configure the [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/docs) (GKE) cluster that we will use to deploy our Spark applications.

# Prerequisites

1. Install `gcloud` as necessary. gcloud can be installed as a part of the [Google Cloud SDK](https://cloud.google.com/sdk/). 
2. Install `kubectl`.

# Before you begin

Set up default `gcloud` settings:

```bash
gcloud auth login
gcloud config set project <project-id>
gcloud config set compute/zone <compute-zone>
gcloud config set compute/region <compute-region>

gcloud components update
```

with the following configuration:

| Property     | Value                   |
|--------------|:------------------------|
| region       | europe-west1            |
| zone         | europe-west1-b          |
| project-id   | hippi-spark-k8s         |
| cluster name | hippi-spark-k8s-cluster |

# Create a GKE cluster:

```bash
gcloud beta container --project "hippi-spark-k8s" clusters create "hippi-spark-k8s-cluster" --zone "europe-west1-b" --no-enable-basic-auth --cluster-version "1.15.12-gke.5000" --machine-type "n2-standard-2" --image-type "COS" --disk-type "pd-standard" --disk-size "100" --metadata disable-legacy-endpoints=true --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --num-nodes "3" --enable-stackdriver-kubernetes --enable-ip-alias --network "projects/hippi-spark-k8s/global/networks/default" --subnetwork "projects/hippi-spark-k8s/regions/europe-west1/subnetworks/default" --default-max-pods-per-node "110" --no-enable-master-authorized-networks --addons HorizontalPodAutoscaling,HttpLoadBalancing --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0
```

## Configure cluster access for kubectl

Generate a `kubeconfig` context entry by running the following command:

```bash
gcloud container clusters get-credentials hippi-spark-k8s-cluster
```

You can modify the default namespace for your `kubectl` commands in this context :

```bash
kubectl config set-context $(kubectl config current-context) --namespace=<namespace>
```

# Spark Docker image

Spark ships with a `bin/docker-image-tool.sh` script that can be used to build (and publish) the Docker image to use 
with the Kubernetes backend.

```bash
./bin/docker-image-tool.sh -t <tag> -p ./kubernetes/dockerfiles/spark/bindings/python/Dockerfile build
```

Push the image to the Container Registry of Google Cloud:

```bash
docker tag spark-py:3.0.1 eu.gcr.io/<project-id>/spark-py:<tag>
gcloud auth configure-docker
docker push eu.gcr.io/<project-id>/spark-py:<tag>
```

with `project-id=hippi-spark-k8s` and `tag` set to the version of Spark, here `3.0.1`.
