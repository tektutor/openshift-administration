## Day 2

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
