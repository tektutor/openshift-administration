![image](https://github.com/user-attachments/assets/8daa9f72-0795-4094-98bc-d35222fb8036)## Day 2

## Lab - Creating a project and deploy nginx web server
```
oc new-project jegan
oc create deploy nginx --image=bitnami/nginx:latest --replicas=3 -o yaml --dry-run=client
oc create deploy nginx --image=bitnami/nginx:latest --replicas=3 -o yaml --dry-run=client > nginx-deploy.yml
```

Make sure you edit the nginx-deploy.yml and add the imagePullPolicy: IfNotPresent
![image](https://github.com/user-attachments/assets/7037dbfa-5905-44d5-aa01-f77c6d94aa12)

Now you may deploy nginx in your project
```
oc apply -f nginx-deploy.yml
oc get deploy
oc expose deploy/nginx --port=8080 
```

# Info - How Calico(BGP) supports Pod to Pod communication without communication overhead ?
<pre>
- Calico is a CNI plugin that supports BGP natively
- Calico advertises Pod CIDRs over BGP
- Each node announces the IPs of its local pods to the BGP peers
- Calico network fabric creates a DaemonSet, so that it deploys one Calico agent pod per node
- Calico agent
  - hosts about 4 components
    - Felix 
      - Main networking agent for Calico
      - Programs routes, iptables rules, and IP sets
      - Handles policy enforcement (NetworkPolicies)
      - Coordinates with the BGP daemon to manage advertised routes
      - does not speak BGP directly, but it programs the kernel routes that are advertised by the BGP agent
    - Bird ( BGP Daemon - BGP Speaker )
      - handles BGP Peering
      - The actual BGP speaker
      - Communicates with upstream routers or other Calico nodes via BGP
      - Advertises:
        - The Pod CIDR for the local node
        - Service IPs or host routes, if configured
        - Listens for BGP updates from other nodes and peers
        - BIRD is the default BGP implementation, optionally can configure GoBGP instead for lighter, faster large-scale use
    - Typha (optional)
      - Not a per-node component, but helps scale Felix in large clusters
      - Acts as a proxy between Felix agents and etcd or the Kubernetes API server
      - Reduces the number of direct API connections
      - may see calico-typha running as a Deployment in larger OpenShift + Calico setups
    - CNI Plugin(calico-cni)
      - Manages Pod networking lifecycle
      - Allocates Pod IPs from IP pools.
      - Ensures the Pod is connected to the node’s network
      - This isn’t directly part of BGP, but it works with Felix to assign routable IPs
- Benefits of Calico
  - No overlay network needed – pure L3 routing.
  - Direct routing = better performance, lower latency.
  - Useful in multi-cluster or hybrid cloud environments.
  - Pod-to-Pod Communication across Clusters (Hybrid/Multicloud)
    - In multi-cluster OpenShift setups, or between OpenShift and other Kubernetes clusters, BGP can help:
      - Cluster A advertises its Pod CIDRs to the network
      - Cluster B advertises its Pod CIDRs to the network
      - Traffic between pods in both clusters routes through the shared network, using BGP-learned paths
      - This avoids the need for VPNs or tunnel encapsulation
  - Integrating with Existing Network Infrastructure
    - BGP is used to:
      - Inform your data center routers or leaf switches about pod IPs
      - Ensures any server in your network can reach any pod in the OpenShift cluster without NAT or overlays
</pre>

#### Without BGP (Packet gets Encapsulated/De-encapsulated - Leads to Performance issues)
<pre>
Pod A (10.1.1.1) --VXLAN--> Node1 (192.168.1.10) ----> Node2 (192.168.1.20) --Decap--> Pod B (10.1.2.2)
</pre>

#### With BGP (No Encapsulation/De-encapsulation - Better Performance)
<pre>
Pod A (10.1.1.1) -----------------routed IP-----------------> Pod B (10.1.2.2)
</pre>

## Info - Red Hat Openshift ovn-kubernetes Network Fabric Overview
<pre>
- OVN-Kubernetes is the default network plugin used starting from Red Hat OpenShift 4.3 onwards 
- OVN-Kubernetes replaces the legacy OpenShift SDN (Software Defined Network)
- It provides a highly scalable, policy-aware, and overlay-capable SDN using OVN (Open Virtual Network), built on Open vSwitch (OVS)
- OVN-Kubernetes is an independent open-source project used by Red Hat Openshift
- OVN-Kubernetes supports two types of architecture
  - default mode ( centralized control plane architecture )
  - interconnect mode ( distributed control plane architecture )
- Red Hat Openshift supports the distributed control plane architecture i.e interconnect mode
- In the ovn-kubernetes distributed control plane architecture
  - Two types of Pods are supported
    1. OVNKube Control Plane Pod
       - Containers
         - kube-rbac-proxy
         - ovnkube-cluster-manager
    2. OVNKube Node Pod
       - Containers
         - ovn-controller
         - ovn-acl-logging
         - kube-rbac-proxy-mode
         - kube-rbac-proxy-ovn-metrics
         - northd
         - nbdb
         - sddb
         - ovnkube-controller
  
Logical Switches
- Represent nodes or namespaces in the Kubernetes cluster
- Each node(master/worker) has a single logic switch

Logical Switch Ports
- Represent pod interfaces
- Each pod gets a logical port that connects it to the logical switch

Logical Routers
- Represent the cluster's logical routing infrastructure, often just one logical router per cluster
- Routes traffic between logical switches (i.e., between pods on different nodes).

Logical Router Ports
- Connect logical switches to the logical router

ACLs (Access Control Lists)
- Used to enforce NetworkPolicies in Kubernetes
- Define rules to allow or deny traffic to and from pods

Load Balancers
- Represent Kubernetes Services
- Used to implement service IPs and ClusterIP-based routing

Address Sets and Port Groups
- Support advanced policy and service definitions
- Help with grouping IPs or ports for easier policy management

Static Routes
- Used by the logical router to route traffic across logical switches.

When a pod is created, the CNI plugin (OVN-Kubernetes) creates a veth pair
- One end is placed inside the pod's network namespace (this is the pod’s eth0)
- The other end remains in the host’s root namespace and is connected to the Open vSwitch (OVS) bridge
</pre>

## Roles and responsibilities - component wise
ovnkube-cluster-manager
- allocates a unique subnet to the node hosting the pod
kube-rbac-proxy
- secures access to metrics and health endpoints
- Enforces Kubernetes Role Bases Access Control
- Enables Prometheus to scrape metrics securely
- TLS Termination - provides encrypted access to metrics or health data

ovnkube-controller
- translates the logical network into actual openflow rules
- controls how packets are forwarded, filtered or modified on that node
- reads Southbound DB generated by ovn-northd
- writes flow rules to the Open vSwitch(OVS) instance running on the node 
- connects pods to logical switches
- Handles encapsulation/deencapsulation for inter-node traffic(Geneve tunneling)
- Implements Kubernetes Services like ClusterIP, Node Port, etc.,
- Enforces Network Policies using ACLs
- When Pod is created
  - detects its logical switch port
  - configures the host OVS bridge to connect the pod via veth pair to the OVN logical switch
- implements Routing and NAT

northd
- is the central component of OVN-Kubernetes network fabric
- acts as a logical network compiler
- sits between the Northbound database and Southbound database
- translates high-level network intent into low-level implementation details
- reads logical constucts like logical switches, routers,ACLs from nbdb
- compiles and writes concrete logical flows into the Southbound database
- these flows are then consumed by ovn-controller

ovn-controller
- programs Open vSwitch (OVS) on the node
  - to connect pods to the network
  - applies NAT, routing and ACLs
  - Handles Geneve encapsulation for inter-node-traffic
  
#### OpenShift OVN-Kubernetes Workflow
<pre>
- When a pod is created, the ovnkube-cluster-manager allocates a unique subnet to the node hosting the pod
- The node's ovn-controller programs the local OVS to set up the necessary networking, 
  including logical switches and routers
- Network policies are enforced locally by the ovn-controller, which programs the OVS with 
  appropriate OpenFlow rules
- Services are exposed through local routers, and DNS entries are updated to reflect the new services.
- For communication between pods on different nodes, the local OVS instances handle traffic routing, 
  leveraging the distributed OVN architecture
</pre>

## Lab - Finding OVN-Kubernetes Components that are running with a master node
Let's get inside the master-1 node
```
oc debug node/master01.ocp4.rps.com
```
![image](https://github.com/user-attachments/assets/5003059e-d480-4b9a-b4c2-ec5d0c905358)

List all ovn-kubernetes network pods
```
crictl pods | grep ovn
```
![image](https://github.com/user-attachments/assets/bc749c35-53db-4f0c-92ee-c2a78a89f941)

Find all the containers running within the ovnkube-node pod
```
crictl ps -p <pod-id> -o json | jq NB-DB-r '.containers[] | "\(.metadata.name)"'
crictl ps -p <pod-id> -o json | jq -r '.containers[]'
```
![image](https://github.com/user-attachments/assets/b2fbe440-dfd6-4600-be9e-64dc19ed3c58)
![image](https://github.com/user-attachments/assets/b28defe0-90d8-432b-ae79-8a6b0f33dde6)

Find all the containers running within the ovnkube-control plane pod
```
crictl ps -p <pod-id> -o json | jq -r '.containers[] | "\(.metadata.name)"'
crictl ps -p <pod-id> -o json | jq -r '.containers[]' 
```
![image](https://github.com/user-attachments/assets/13eb3002-fbce-4b71-9cd1-e023d4bc1ab3)
![image](https://github.com/user-attachments/assets/28b3cca3-0709-4b14-a82c-599167337582)

## Lab - Find all pods running in openshift cluster related to ovn-kubernetes network fabric
```
oc get pods -n openshift-ovn-kubernetes
```

List all containers of ovnkube-control-plane pod in master node
```
oc get pod/ovnkube-control-plane-6bf48b856d-2xm7t -n openshift-ovn-kubernetes -o json | jq -r '[.spec.containers[].name]'
```

List all containers of ovnkube-node pod in master node
```
oc get pod ovnkube-node-lctc4 -n openshift-ovn-kubernetes | jq -r '[.spec.containers[].name]'
```

List all containers of ovnkube-node pod in worker node
```
oc get pod ovnkube-node-lctc4 -n openshift-ovn-kubernetes | jq -r '[.spec.containers[].name]'
```

![image](https://github.com/user-attachments/assets/4bc07de3-097a-4bc5-8d18-a5e35fdbb956)
![image](https://github.com/user-attachments/assets/51ef3d8e-1ea6-4cdc-ac6a-a39822b961f3)

## Lab - Create a pod in privileged mode without permissive security context
```
cd ~
git clone https://github.com/tektutor/openshift-administration.git
cd openshift-administration/Day2/security-context-constraint
cat pod-with-root-without-using-scc.yml
oc apply -f pod-with-root-without-using-scc.yml
```

Expected output
![image](https://github.com/user-attachments/assets/6d1c6f4b-2af8-49be-a82b-0ff0844e57da)
![image](https://github.com/user-attachments/assets/3c6f33c6-b4bb-4d84-8102-e0bcbd8a19a5)
![image](https://github.com/user-attachments/assets/6d0b4c15-761c-4c1d-aa7e-c3526c950943)
![image](https://github.com/user-attachments/assets/e5ca5ea7-0a4f-442c-a1d5-7a685e7e484d)

Since the developer account doesn't have a service account with sufficient permission to run a Pod in privileged mode, the pod execution is denied by Openshift as expected.

## Lab - Let's create a similar pod with required permissions
Let's create a custom security context contraint
```
cd ~/openshift-administration
git pull
cd Day2/security-context-constraint
oc login -u developer -p developer
oc whoami
cat custom-scc.yml
oc apply -f custom-scc.yml

oc login -u kubeadmin
oc apply -f custom-scc.yml

oc create sa privileged-sa
oc apply -f custom-scc.yml

oc login -u developer
cat pod-with-custom-scc.yml
oc apply -f pod-with-custom-scc.yml
oc create deploy nginx --image=bitnami/nginx:latest
oc get po
oc get po -w
oc get pod nginx-85458687c9-8pjlg -o jsonpath='{.spec.serviceAccountName}'
oc adm policy who-can use scc privileged
oc adm policy add-scc-to-user privileged -z default -n jegan system:serviceaccount:jegan:default
cat pod-with-custom-scc.yml
oc whoami
oc apply -f pod-with-custom-scc.yml
oc get po
```
Expected output
![image](https://github.com/user-attachments/assets/dbad1a34-32c9-4f3e-8454-6f893526adbc)
![image](https://github.com/user-attachments/assets/ae21cc9b-f636-4c05-a646-0f735d79f62f)
![image](https://github.com/user-attachments/assets/96269c29-d3dd-41c4-ab09-9bf809bde921)
![image](https://github.com/user-attachments/assets/9d6b95c6-8e48-4077-b122-fe672a5f4998)
![image](https://github.com/user-attachments/assets/682920ac-5036-4f63-a11c-56c36f875f0c)
![image](https://github.com/user-attachments/assets/1d7dd177-898e-410f-98d9-75c595c91e58)
![image](https://github.com/user-attachments/assets/a8578f69-00dd-4cbd-b7ea-2e247c20cb3d)
![image](https://github.com/user-attachments/assets/821dca56-2763-4810-9870-60cf0349e278)

Ideally, developer account should not given such privileged access, hence let's revoke the permission
```
oc login -u kubeadmin
oc project jegan
oc delete -f pod-with-custom-scc.yml
oc whoami
oc adm policy remove-scc-from-user privileged -z default -n jegan
oc login -u developer
oc apply -f pod-with-custom-scc.yml
```
Expected output
![image](https://github.com/user-attachments/assets/4e7096b2-bd0b-498c-b043-e2e07f8a88fa)
![image](https://github.com/user-attachments/assets/961bfb05-72cf-4045-8192-ecd7b7ec9f6e)

## Info - PodSecurityPolicies(PSP)
<pre>
- PodSecurityPolicies work in Kubernetes prior to v1.25 but not in Openshift
- PodSecurityPolicies were deprecated in Kubernetes v1.21 and removed in v1.25
- Using Security Context Contraint(SCC) is ideal or alternatively in Openshift we can use Pod Security Admission(PSA)
</pre>

## Info - Pod Security Admssion(PSA) Overview
<pre>
- is built-in admission controller that enforces Kubernetes Pod Security standards on a per namespace basis
- It supports 3 levels of security
  1. privileged - No restrictions(most permissive)
  2. baseline - Basic safety, allows common usecases
  3. restricted - Strictest, follows principle of least privilege
- this can be set using label pod-security.kubernetes.io/enforce: restricted
</pre>  

## Lab - Let's apply the PSA security in our project namespace
```
oc login -u kubeadmin

oc whoami

oc label namespace jegan \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest
 
oc project jegan
cat pod-violating-psa.yml
oc apply -f pod-violating-psa.yml
oc get po

cat pod-following-psa-bestpractice.yml
oc apply -f pod-following-psa-bestpractice.yml
oc get po
```

Expected output
![image](https://github.com/user-attachments/assets/103d22ce-e348-484c-93b4-41df2a8b0a37)
![image](https://github.com/user-attachments/assets/7098d52d-7406-4ff5-9b12-bff4bccb12fa)
![image](https://github.com/user-attachments/assets/fa8e0c5d-18dd-4adf-ac0d-9b1f2cad279c)
![image](https://github.com/user-attachments/assets/13ac977b-64cf-428d-be1e-9e1d7cdfd68f)
![image](https://github.com/user-attachments/assets/45747bcb-a026-4864-8a1e-7ca226130efe)

Trying changing the pod-security.kubernetes.io/enforce=privileged and later to pod-security.kubernetes.io/enforce=baseline and observe the behaviour.


## Lab - Deploying Ceph strorage into Openshift
<pre>
- Need to install Openshift Data Foundation Operator a.k.a (ODF Operator)
- Ceph Operator Components
  - rook-ceph-operator
    - runs as a Deployment usually in the openshift-storage namespace
    - responsible for managing the lifecycle of Ceph components (e.g., creating OSDs, MONs, MGRs, etc.)
    - typically runs on a control plane or infra node
- Ceph Cluster Components
  - MON (Monitor Pods)
    - Usually 3 instances for HA (can be 5 in large clusters)
    - Store cluster state and help with consensus
    - Scheduled on different nodes to ensure fault tolerance
  - OSD (Object Storage Daemon) Pods
    - One or more per storage node, depending on how many storage devices are available
    - Responsible for data storage, replication, recovery.
    - Deployed only on nodes with attached block devices 
    - usually labeled with cluster.ocs.openshift.io/openshift-storage=''
  - MGR (Manager) Pods
    - Provide metrics and manage cluster state
    - One active manager, possibly one standby
    - Usually co-located with MONs or other components
  - MDS (Metadata Server) Pods (Only if CephFS is used)
    - Handle file system metadata.
    - 2 or more pods for HA
  - RGW (RADOS Gateway) Pods 
    - Only if object storage is used via S3 interface
    - Provide S3-compatible object storage interface
    - Usually runs on any worker or storage nodes
- Supporting Components
  - CSI Drivers (CephFS and RBD)
    - Enable dynamic provisioning of persistent volumes
    - Deployed as DaemonSets across all relevant worker nodes
  - rook-ceph-crashcollector
    - One per node with Ceph components
    - Collects crash data for debugging
  - rook-ceph-agent / rook-discover
    - DaemonSets used during initial discovery and device mapping
    - Usually run on all nodes, but mainly used during setup or when adding storage nodes
</pre>

Run the below command to see the pods in openshift-storage namespace
```
oc get pods -n openshift-storage -o wide
```

Check if the rook-ceph-operator, rook-ceph-osd-*, rook-ceph-mon-*, csi-ceph-* pods are running
```
oc get pods -n openshift-storage
```
![image](https://github.com/user-attachments/assets/3d554f92-7f19-4a21-97f5-e765366edced)


## Info - Creating and Managing Users,Groups in OpenShift
<pre>
- Authentication in OpenShift is managed by authentication operator
- authentication operator runs an oauth-server
- oauth-server is where the users obtain oauth access token to authenticate into the API
- oauth server can be configured to use identify providers such as htpasswd, LDAP, GitLab, etc.,  
</pre>

## Info - Installing htpasswd in Ubuntu 
```
sudo apt install -y apache2-utils
```

## Lab - Creating and Managing Users,Groups in OpenShift 
```
htpasswd -cBb /tmp/htpasswd tektutor rps@123
cat /tmp/htpasswd
htpasswd -Bb /tmp/htpasswd tektutor-developer1 rps@123
cat /tmp/htpasswd
oc login -u kubeadmin -p https://
oc create secret generic htpasswd-secret --from-file htpasswd=/tmp/htpasswd -n openshift-config
oc get oauth cluster -o yaml > oauth.yml
```

Edit the oauth.yml
<pre>
- htpasswd:
    fileData:
      name: htpasswd-secret
  mappingMethod: claim
  name: cusers
  type: HTPasswd
</pre>

```
oc replace -f oauth.yml
oc get pods -n openshift-authentication
oc login -u tektutor-admin -p rps@123
oc get pods
oc get svc
oc get nodes
oc whoami
oc login -u kubeadmin -p
oc adm policy add-cluster-role-to-user cluster-admin tektutor-admin
oc login -u tektutor-admin -p rps@123
oc get pods
oc get svc
oc get nodes
oc get users
oc get identify
oc login -u tektutor-developer1 -p rps@123
oc login -u tektutor-admin -p rps@123
oc get users
oc extract secret/htpassword-secret -n openshift-config --to /tmp --confirm
cat /tmp/htpasswd
htpasswd -b /tmp/htpasswd tektutor-developer2 rps@123
oc set data data secret/htpasswd-secret --from-file htpasswd=/tmp/htpasswd -n openshift-config
oc login -u tektutor-developer2 -p rps@123

# Change password
oc login -u kubeadmin -p
oc extract secret/htpasswd-secret -n openshift-config --to /tmp --confirm
htpasswd -b /tmp/htpasswd tektutor-developer1 Rps@123
cat /tmp/htpasswd
oc set data secret/htpasswd-secret --from-file htpasswd=/tmp/htpasswd -n openshift-config
oc get pods -n openshift-authentication
oc login -u tektutor-developer1 Rps@123

# Delete User
oc login -u tektutor-admin -p rps@123
oc extract secret/htpasswd-secret -n openshift-config --to /tmp --confirm
htpasswd -D /tmp/htpasswd tektutor-developer2
cat /tmp/htpasswd
oc set data secret/htpasswd-secret --from-file htpasswd=/tmp/htpasswd -n openshift-config
oc delete identify cusers:tektutor-developer2
oc get identify
oc delete user tektutor-developer2
oc get users
oc login -u tektutor-developer2 -p rps@123
```

