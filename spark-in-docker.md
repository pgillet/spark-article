This guide explains how to build an "official" Spark Docker image and how to run a basic Spark application with it.

# Spark Docker image

Spark (starting with version 2.3) ships with Dockerfiles that can be used to build different Spark Docker images (and 
customize them to match an individual application’s needs) to use with a Kubernetes backend. They can be found in the 
`kubernetes/dockerfiles/` directory.

Spark also ships with a `bin/docker-image-tool.sh` script that eases the building (and the publishing) of the Docker 
images.

Example usage to build an image with the Python binding (PySpark):

```bash
./bin/docker-image-tool.sh -t <tag> -p ./kubernetes/dockerfiles/spark/bindings/python/Dockerfile build
```

This will create a local Docker image named `spark-py:<tag>`. You can set the `<tag>` with the Spark's actual version. 

# Running the Examples

Once the image is bundled, you can launch a Spark application using the `bin/spark-submit` script.

```bash
# Run Spark locally with two worker threads
docker run --rm spark-py:3.0.1 /opt/spark/bin/spark-submit --master local[2] /opt/spark/examples/src/main/python/pi.py
```

# Recommendation

It is strongly recommended to start from an official _base_ image to create any custom Spark image.

