We are soon coming to the end of our journey. Before wishing you good night, let's take a look at how we can monitor 
a Spark application from Python code.

# Monitoring

## Getting the status of a Spark application

Many operations in the Kubernetes Python client can be _watched_, that means that our program can call a method of the
Python API periodically until you get the desired result or the watch expires. We get the status as follows:

```python
from kubernetes import client, watch

app_name = 'pyspark-pi-routine-bf20cae50b6a8253'
v1 = client.CoreV1Api()
    
count = 10
w = watch.Watch()
label_selector = 'app-name=%s,spark-role=driver' % app_name
for event in w.stream(v1.list_namespaced_pod, namespace='spark-jobs', label_selector=label_selector, timeout_seconds=60):
    print('Event: %s' % event['object'].status.phase)
    count -= 1
    if not count:
        w.stop()
```

## Getting the logs

Getting the logs of the driver pod is just as easy:

```python
from kubernetes import client, watch

v1 = client.CoreV1Api()
pod_name = 'pyspark-pi-routine-bf20cae50b6a8253-driver'

def logs(pod_name):
    w = watch.Watch()
    for event in w.stream(v1.read_namespaced_pod_log, pod_name=pod_name, namespace='spark-jobs', _request_timeout=300):
        yield event

# We surely don't want to block the main thread while reading the logs
def consumer():
    for log in logs(pod_name):
        print(log)

t = Thread(target=consumer)
t.start()
```

## Getting the Ingress URI

```python
from kubernetes import client, watch

app_name = 'pyspark-pi-routine-bf20cae50b6a8253'

networking_v1_beta1_api = client.NetworkingV1beta1Api()
w = watch.Watch()
label_selector = 'app-name=%s' % app_name
for event in w.stream(networking_v1_beta1_api.list_namespaced_ingress, namespace=namespace,
                      label_selector=label_selector,
                      timeout_seconds=30):
    ingress = event['object'].status.load_balancer.ingress
    if ingress:
        external_ip = ingress[0].ip
        print('Event: The Spark Web UI is available at http://%s/%s' % (external_ip, app_name))
        w.stop()
    else:
        print('Event: Ingress not yet available')
```

