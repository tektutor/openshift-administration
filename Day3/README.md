## Day 3

## Info - Monitoring and Logging
<pre>
- Monitoring and logging in OpenShift are critical for maintaining the health, 
  performance, and security of applications and the cluster itself 
- OpenShift, built on Kubernetes, integrates tools and frameworks to help administrators 
  and developers observe system behavior in real time and retrospectively
</pre>

#### OpenShift Monitoring Stack
<pre>
- OpenShift includes a pre-configured monitoring stack that uses:
  - Prometheus – Metrics collection
  - Alertmanager – Handling alerts triggered by Prometheus
  - Thanos – Optional, for long-term storage and high availability of metrics
  - Grafana – Visualization of metrics
</pre>

#### Cluster vs User Workload Monitoring
<pre>
- Cluster Monitoring: Collects metrics about OpenShift components and the platform itself
- User Workload Monitoring: Allows monitoring of applications deployed by users
</pre>

## Lab - Enable Monitoring for user deployed applications in Openshift

By default, Openshift only monitors cluster components only, to enable monitoring for your applictions, you need to enable as shown below
<pre>
# cluster-monitoring-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true  
</pre>

Apply the configmap in your cluster
```
oc apply -f cluster-monitoring-config.yaml
```

Let's create a project
```
oc new-project jegan-monitoring
```

Create a deloyment demo-app.yaml
<pre>
# demo-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: demo-monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/metrics"
        prometheus.io/port: "8080"
    spec:
      containers:
      - name: demo-app
        image: quay.io/prometheus/busybox-http-exporter
        ports:
        - containerPort: 8080
        args:
        - --metric=counter:requests_total  
</pre>

Create the deployment
```
oc apply -f demo-app.yaml
```

Create a ServiceMonitor for Prometheus to scrape the app metrics
<pre>
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: demo-app
  namespace: demo-monitoring
  labels:
    release: "cluster"
spec:
  selector:
    matchLabels:
      app: demo-app
  endpoints:
  - port: metrics
    interval: 15s
    path: /metrics 
</pre>

Let's expose the demo-app with a service
<pre>
# demo-app-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-app
  namespace: demo-monitoring
  labels:
    app: demo-app
spec:
  ports:
  - name: metrics
    port: 8080
    targetPort: 8080
  selector:
    app: demo-app  
</pre>
Let's create the service and service monitor
```
oc apply -f demo-app-service.yaml
oc apply -f demo-app-servicemonitor.yaml
```

Let's access the Openshift's built-in Graphana
```
oc get route -n openshift-monitoring grafana
```

We need to login with Openshift credentials (OAuth)
We need to create a dashboard with Graphana with Prometheus datasource
Example Prometheus query to see your demo app metrics:
```
requests_total{app="demo-app"}
```
Logging setup using OpenShift Cluster Logging Operator (EFK)
```
oc apply -f https://raw.githubusercontent.com/openshift/cluster-logging-operator/master/deploy/cluster-logging-operator.yaml
```

Enable Cluster Logging with Elasticsearch, Fluentd, and Kibana
```
# cluster-logging.yaml
apiVersion: "logging.openshift.io/v1"
kind: "ClusterLogging"
metadata:
  name: "instance"
  namespace: "openshift-logging"
spec:
  managementState: "Managed"
  logStore:
    type: "elasticsearch"
    elasticsearch:
      nodeCount: 3
      storage:
        storageClassName: "gp2"  # or your storage class
        size: 200Gi
  visualization:
    type: "kibana"
  collection:
    logs:
      type: "fluentd"
```

Let's create the elk stack
```
oc apply -f cluster-logging.yaml
```

View logs in Kibana
```
oc get route -n openshift-logging
```
<pre>
- Search for logs
- You can search for logs by namespace, pod, container, etc
- Try filtering for your demo-monitoring namespace logs
</pre>

