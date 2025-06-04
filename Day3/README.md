## Day 3

## Info - Tekton Jargons
<pre>
- Red Hat Pipeline operator installs Tekton Knative Serverless Pipeline framework into Openshift
- the operators adds many Custom Resource like
  - Task
  - TaskRun
  - Pipeline
  - PipelineRun
  - PipelineResources
  - Catalog
  - Tekton Trigger
</pre>


## Info - What is a Task?
<pre>
- is one of the  Custom Resources added by Red Hat openshift operator using CRD
- Task can be executed independently outside a Tekton Pipeline
- Task supports two scopes
  - Project scope
  - Cluster scope
- Each task contains one or more steps
- Task is the smallest independent unit that can be deployed in Tekton under Openshift/Kubernetes
- Examples
  - A Task can be used to clone source code from GitHub, BitBucket, GitLab, any version control
  - A Task can be used to build a container image, etc.,
  - A Task can be used to build application binary from source code, etc.,
</pre>

## Info - What is a Step in Tekton Task?
<pre>
- Step is a Custom Resource added by Tekton in our openshift cluster using Custom Resource Definition(CRD)
- Each Step runs in a separate Container
- Steps can' be executed independently
- Steps always runs within a Tekton Task
</pre>

## Info - Tekton TaskRun
<pre>
- is a custom Resource added by Tekton
- Task can be thought of like a Class, while TaskRun is running instance of a Task
- TaskRun helps us supply Task arguments/parameters that are required for a Task to run
- Tekton TaskRun controller will spin off a Pod for each instance of TaskRun
</pre>

## Info - Pipeline
<pre>
- is a Custom Resource added by Tekton
- it is a collection of many Tasks executed in sequence or parallel
- results in an output based on your inputs given to CI/CD pipeline
- Examples
  - a First Task could clone your source code from your GitHub Repository
  - a Second Task could compile your application from the First Task clones the source code
  - a Third Task could run Unit test cases against your compiled application binary if the Second Task succeeds
  - a Fourth Task could package the application binaries if the Third Task succeeds
  - a Fifth Task could deploy the binaries to JFrog Artifactory Server or Sonatype Nexus Server if the Fourth Task succeeds
  - a Sixth Task could run in parallel to Fifth Task to deploy the microservice(application) to Openshift if Fourth Task succeeds
</pre>

## Info - PipelineRun
<pre>
- is a Custom Resource added by Tekton
- Pipeline is the blueprint/specification of PipelineRun
- PipelineRun is the running instance of the Pipeline
- PipelineRun helps us provide arguments to Pipeline
</pre>

## Info - Tekton Trigger
<pre>
- is a component that allows Tekton Pipeline to detect events from variety of sources
- it also helps us detect code changes from GitHub, or any version control
- Tekton Triggers can invoke TaskRun and/or PipelineRun with the parameters retrieved from events
- Tekton Triggers can invoke TekTon Pipeline
- Example
  - helps in triggering a Tekton CI/CD pipeline based on code commit to GitHub repo or similar version controls
</pre>

## Info - Tekton Catalog
<pre>
- Reusable Task and Pipeline Resources from a website ( Tekton Hub - hub.tekton.dev )
</pre>
![image](https://github.com/user-attachments/assets/525e00d0-42a4-40b2-8eb7-763b0bdf6c5b)
![image](https://github.com/user-attachments/assets/a758ff48-8d90-45e2-808e-b774b01039a8)


## Info - Tekton Dashboard
<pre>
- Web based Dashboard or webconsole for Tekton
- Openshift integrated the Tekton Dashboard within Openshift webconsole
- in case of Kubernetes, Tekton provides an independent Tekton Dashboard that can be accessed outside the Kubernetes cluster
</pre>

## Info - Tekton Hub has many reusable Tasks and Pipeline code
<pre>
https://hub.tekton.dev/
</pre>
![image](https://github.com/user-attachments/assets/2fbcebb9-539d-4bfd-9777-a33626160208) 

## Info - Tekton Workspace
<pre>
- is a folder where a Tekton Task can store the output
- the output stored by a Task can be fed as an input to other Tasks in the Tekton Pipeline
- For example
  - Task 1 - Clone source code from GitHub Repo and stores it in the output workspaces ( external NFS path )
  - Task 2 - Uses the already clone source and build the source using maven
</pre>

## Lab - Cloning TekTutor Training Repository
```
cd ~
git clone https://github.com/tektutor/cd ~/openshift-administration.git
```

## Lab - Creating and running your first Tekton Task
```
cd ~/openshift-administration
git pull
cd Day3/tekton
cat first-tekton-task.yml
oc apply -f first-tekton-task
oc get taks
tkn task list
tkn task start helloworld-task
tkn taskrun logs helloworld-task-run-xnn4j -f -n jegan
```
Expected output
![image](https://github.com/user-attachments/assets/b8b95cf9-e759-4765-838b-7bf533fa43a0)
![image](https://github.com/user-attachments/assets/4ebd983d-3f55-428c-8a09-0c31113ed8b5)

## Lab - Tekton Client tool usage details
```
tkn

tkn tasks
tkn task
tkn t

tkn taskruns
tkn taskrun
tkn tr

tkn pipelines
tkn pipeline
tkn p

tkn pipelineruns
tkn pipelinerun
tkn pr
```

Expected output
![image](https://github.com/user-attachments/assets/3d3670ff-535d-4e36-86a0-ab13fc6ce58e)
![image](https://github.com/user-attachments/assets/a0fb4ed0-8db9-4fd6-aab2-e0686582f2fc)
![image](https://github.com/user-attachments/assets/05499b28-4a68-46af-9ebb-a5374c319bde)
![image](https://github.com/user-attachments/assets/7cdfc74d-219a-4071-93d5-8c42c52698fe)

## Lab - Tasks with multiple steps
```
cd ~/openshift-administration
git pull
cd Day3/tekton
cat tasks-with-multiple-steps.yml
oc apply -f tasks-with-multiple-steps.yml
tkn task list
tkn task start task-with-multiple-steps --show-log
tkn taskrun list
```

Expected output
![image](https://github.com/user-attachments/assets/ff4308af-795f-4abc-b4b6-47cc77e8bcff)
![image](https://github.com/user-attachments/assets/573cd1bd-86e2-44ab-becf-e5aa915e4532)
![image](https://github.com/user-attachments/assets/d6397d01-4da2-4368-9f31-bc0be5461202)
![image](https://github.com/user-attachments/assets/0f81c13d-2c97-4bba-82b3-eb1ec844ac75)

## Lab - Passing parameter to a Tekton Task
```
cd ~/openshift-administration
git pull
cd Day3/tekton
cat task-with-params.yml
oc apply -f task-with-params.yml
tkn task list
tkn task start hello-task-with-params --showlog
```

Expected output
![image](https://github.com/user-attachments/assets/268102d1-7497-43a2-8b99-d188f0797626)
![image](https://github.com/user-attachments/assets/d57e3e00-bf13-4b91-8ae9-a21b764f56d0)

## Lab - Installing Tasks from Tekton Hub to refer that from our TaskRun
```
tkn hub install task git-clone
```
Expected output
![image](https://github.com/user-attachments/assets/a3c71d71-facb-43d6-8db6-d8d97ac02c9a)
![image](https://github.com/user-attachments/assets/12d4da30-da3f-4f2c-96b4-2ec26a22a53f)

## Lab - Clone GitHub repo using Tekton TaskRun

Make sure you are customizing the clone.yml replace 'jegan' with your name.   Also make sure the nfs server IP and the path is updated as per the folder allocated to you before proceeding.

```
cd ~/openshift-administration
git pull
cd Day3/tekton
cat clone.yml
oc apply -f clone.yml
```

Expected output
![image](https://github.com/user-attachments/assets/a004d436-d489-4d18-b06d-53fd2d4b7c47)
![image](https://github.com/user-attachments/assets/861874dc-533c-4ff4-acd1-2ed6fec59eb1)
![image](https://github.com/user-attachments/assets/99d536b2-1c93-47e3-aecc-ecc1a56730f7)
![image](https://github.com/user-attachments/assets/759b3930-b147-4216-a840-71ccdcf976b4)

## Lab - Build a maven project using Tekton TaskRun
```
cd ~/openshift-administration
git pull
cd Day3/tekton
cat maven-java-task.yml
oc create -f maven-java-task.yml
```

Expected output
![image](https://github.com/user-attachments/assets/3c1013b0-8d7a-4f19-9507-c2f225462ee8)
![image](https://github.com/user-attachments/assets/193161b7-f20f-4bf7-81fd-304f0a0cf90b)

As we haven't installed the Maven task, our TaskRun failed as it wasn't able to locate the Maven Task
![image](https://github.com/user-attachments/assets/b6816176-a97b-4151-a519-ac3ef3a771d8)
![image](https://github.com/user-attachments/assets/4126d52a-3506-4beb-98b3-2ad5b30c83e2)

To fix this error, we need to first install the Maven Tekton Task into our Openshift project
```
oc project jegan
tkn hub install task maven
oc create -f maven-java-task.yml
```

Expected output
![image](https://github.com/user-attachments/assets/f317180b-3be9-4655-acb1-1d53eba345dd)
![image](https://github.com/user-attachments/assets/54391e37-2a48-4001-a919-a991558d1518)
![image](https://github.com/user-attachments/assets/bc2d373e-40b4-4fe9-b3f1-ea85440a75a6)
![image](https://github.com/user-attachments/assets/21d23e74-4158-4e44-b7b5-f1c039e0ed92)
![image](https://github.com/user-attachments/assets/dc3db51f-ea7b-4333-ae01-52b05e09bc77)
![image](https://github.com/user-attachments/assets/85a32575-d22b-4d37-a749-d2727976e8e1)
![image](https://github.com/user-attachments/assets/335be265-7829-4164-bb32-0d07dc73dc34)
![image](https://github.com/user-attachments/assets/f803ccc1-ee58-469c-83c7-2bc1f6900216)
![image](https://github.com/user-attachments/assets/66b6d076-d80a-4e8b-8af6-f5fa22976d24)
![image](https://github.com/user-attachments/assets/c2a8e3fd-3ca5-42a8-9948-9aa056313cca)

## Lab - Kindly delete the project after each exercise is completed
```
oc delete project jegan
oc new-project jegan
```

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

