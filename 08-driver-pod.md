With native Spark, the main resource is the driver pod.
To run the Pi example program like with the Spark Operator, the driver pod must be created using the data in the 
following yaml file:

`pyspark-pi-driver-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app-name: pyspark-pi-${PRIORITY_CLASS_NAME}${NAME_SUFFIX}
    spark-role: driver
  name: pyspark-pi-${PRIORITY_CLASS_NAME}${NAME_SUFFIX}-driver
  namespace: ${NAMESPACE}
spec:
  containers:
  - name: pyspark-pi
    image: eu.gcr.io/yippee-spark-k8s/spark-py:3.0.1
    imagePullPolicy: IfNotPresent
    ports:
    - containerPort: 5678
      name: headless-svc
    - containerPort: 4040
      name: web-ui
    resources:
      requests:
        memory: 512Mi
        cpu: 1
      limits:
        cpu: 1200m
    env:
    # Overriding configuration directory
    - name: SPARK_CONF_DIR
      value: /spark-conf
    - name: SPARK_HOME
      value: /opt/spark
    # Configure all key-value pairs in ConfigMap as container environment variables
    envFrom:
      - configMapRef:
          name: pyspark-pi-${PRIORITY_CLASS_NAME}${NAME_SUFFIX}-cm
    args:
    - $(SPARK_HOME)/bin/spark-submit
    - /opt/spark/examples/src/main/python/pi.py
    - "10"
    volumeMounts:
      - name: spark-config
        mountPath: /spark-conf
        readOnly: true
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
            - key: type
              operator: In
              values: [${DRIVER_NODE_AFFINITIES}]
  priorityClassName: ${PRIORITY_CLASS_NAME}
  restartPolicy: OnFailure
  schedulerName: volcano
  serviceAccountName: ${SERVICE_ACCOUNT_NAME}
  volumes:
    # Add the executor pod template in read-only volume, for the driver to read
    - name: spark-config
      configMap:
        name: pyspark-pi-${PRIORITY_CLASS_NAME}${NAME_SUFFIX}-cm
        items:
        - key: spark-defaults.conf
          path: spark-defaults.conf
        - key: spark-env.sh
          path: spark-env.sh
        - key: executor-pod-template.yaml
          path: executor-pod-template.yaml
```

**Don't pay attention to the variable placeholders for now**, even though you can imagine what they can be used for.

As you can see, the driver pod depends on other Kubernetes resources. We will detail each of them.