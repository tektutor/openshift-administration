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
BGP (Border Gateway Protocol) can be used in OpenShift to support Pod-to-Pod communication, especially in environments that require high performance, external network integration, or where Kubernetes networking needs to extend beyond traditional overlay networks.

How BGP Supports Pod-to-Pod Communication in OpenShift
In OpenShift, native Pod-to-Pod communication is typically handled via SDN (Software Defined Networking), like OVN-Kubernetes or the older OpenShift SDN. However, BGP becomes relevant when integrating with external routers or non-Kubernetes nodes, particularly in hybrid or bare-metal environments.

Here’s how BGP helps:

1. Using MetalLB for BGP in OpenShift
MetalLB is a load-balancer implementation for bare-metal Kubernetes clusters.

It supports BGP mode, where it peers with external routers and announces service IPs via BGP.

While MetalLB is more commonly used for Service IPs (like LoadBalancer services), it can also help expose Pod IPs directly or indirectly, enabling external routing to them.

Workflow:
Pods are assigned IPs from a routable range.

MetalLB (or a similar BGP speaker) advertises these IPs to the upstream network.

External routers know how to reach the Pod IPs directly.

Pod-to-Pod traffic, even across different clusters or data centers, can be routed via standard IP routing with no encapsulation overhead (unlike VXLAN or Geneve).

2. Using Calico with BGP
Calico is a CNI plugin that supports BGP natively.

When used in OpenShift (usually in custom or bare-metal deployments), Calico advertises Pod CIDRs over BGP.

Each node announces the IPs of its local pods to the BGP peers (usually ToR switches or routers).

Benefits:
No overlay network needed – pure L3 routing.

Direct routing = better performance, lower latency.

Useful in multi-cluster or hybrid cloud environments.

3. Pod-to-Pod Communication across Clusters (Hybrid/Multicloud)
In multi-cluster OpenShift setups, or between OpenShift and other Kubernetes clusters, BGP can help:

Cluster A advertises Pod CIDRs to the network.

Cluster B does the same.

Traffic between pods in both clusters routes through the shared network, using BGP-learned paths.

This avoids the need for VPNs or tunnel encapsulation.

4. Integrating with Existing Network Infrastructure
BGP is used to:

Inform your data center routers or leaf switches about pod IPs.

Ensure that any server in your network can reach any pod in the OpenShift cluster without NAT or overlays.

Make OpenShift part of a fabric-wide L3 routing domain.
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

