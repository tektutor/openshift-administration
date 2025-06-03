## Day 2

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

# Info - How Calico(BGP) supports to Pod to Pod communication without communication overhead
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
      - Ensure that any server in your network can reach any pod in the OpenShift cluster without NAT or overlays
</pre>

#### Without BGP (Packet gets Encapsulated/De-encapsulated - Leads to Performance issues)
<pre>
Pod A (10.1.1.1) --VXLAN--> Node1 (192.168.1.10) ----> Node2 (192.168.1.20) --Decap--> Pod B (10.1.2.2)
</pre>

#### With BGP (No Encapsulation/De-encapsulation - Better Performance)
<pre>
Pod A (10.1.1.1) -----------------routed IP-----------------> Pod B (10.1.2.2)
</pre>


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

