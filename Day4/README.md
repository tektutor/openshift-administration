## Day 4

## Red Hat OpenShift Multi-Cluster setup

```
oc get ns open-cluster-management
oc get ns open-cluster-management-hub
```

Create a managedcluster.yml
<pre>
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  name: rps-ocp4-multicluster
  labels:
    name: rps-ocp4-multicluster
spec:
  hubAcceptsClient: true  
</pre>

Apply it
```
oc apply -f managed-cluster.yaml
oc get secret -n rps-ocp4-multicluster
oc get secret rps-ocp4-multicluster-import -n rps-ocp4-multicluster -o jsonpath='{.data.import\.yaml}' | base64 -d > import.yaml
```



We need to register all the managed cluster to the Multi-cluster Hub
```
# Install CRDs
kubectl apply -f https://github.com/open-cluster-management-io/registration-operator/releases/latest/download/klusterlet.crd.yaml

# Install registration operator
kubectl apply -f https://github.com/open-cluster-management-io/registration-operator/releases/latest/download/registration-operator.yaml

oc apply -f import.yaml
```

We can get the multi-cluster hub route
```
oc get subscription multiclusterhub-operator -n open-cluster-management -o yaml


oc get installplan -n open-cluster-management
oc describe installplan <installplan-name> -n open-cluster-management
oc patch installplan <installplan-name> -n open-cluster-management --type merge -p '{"spec":{"approved":true}}'
oc get pods -n open-cluster-management
oc get subscription multiclusterhub-operator -n open-cluster-management
oc describe subscription multiclusterhub-operator -n open-cluster-management
oc get pods -n openshift-operator-lifecycle-manager

oc get csv -n open-cluster-management
oc describe multiclusterhub multiclusterhub -n open-cluster-management
oc scale deployment multiclusterhub-operator -n open-cluster-management --replicas=0
oc get multiclusterhub multiclusterhub -n open-cluster-management -o json > mch.json
oc delete application --all -n open-cluster-management
oc delete placementrule --all -n open-cluster-management
oc delete policy --all -A
oc delete managedcluster --all
oc delete multiclusterhub multiclusterhub -n open-cluster-management

oc get validatingwebhookconfigurations | grep multiclusterhub
oc delete validatingwebhookconfiguration multiclusterhub-operator-validating-webhook
oc patch validatingwebhookconfiguration multiclusterhub-operator-validating-webhook \
  --type='json' \
  -p='[{"op": "replace", "path": "/webhooks/0/failurePolicy", "value":"Ignore"}]'
oc get multiclusterhub multiclusterhub -n open-cluster-management -o json > mch.json
# Edit: remove finalizers and save
oc replace -f mch.json
oc delete multiclusterhub multiclusterhub -n open-cluster-management --force --grace-period=0
oc delete namespace open-cluster-management --force --grace-period=0
oc delete ns open-cluster-management-hub --force --grace-period=0
oc get crds | grep open-cluster-management | awk '{print $1}' | xargs oc delete crd


oc config get-contexts
oc config use-context managed-cluster-1
oc whoami
oc get nodes

oc get route multicloud-console -n open-cluster-management -o jsonpath='{.spec.host}'

oc get route console -n openshift-console -o jsonpath='{.spec.host}'
```

## Lab - LDAP Integration with Openshift

Create a secret for LDAP Bind Credential
```
oc create secret generic ldap-secret \
  --from-literal=bindPassword='YOUR_LDAP_PASSWORD' \
  -n openshift-config
```

Create a LDAP Identity Provider Configuration
<pre>
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: my_ldap_provider
    mappingMethod: claim
    type: LDAP
    ldap:
      url: ldaps://ost.com:636/ou=users,dc=ost,dc=com?uid
      bindDN: "cn=admin,dc=example,dc=com"
      bindPassword:
        name: ldap-secret
      insecure: false
      attributes:
        id:
        - uid
        name:
        - cn
        email:
        - mail
        preferredUsername:
        - uid
</pre>

Apply 
```
oc apply -f ldap-oauth.yaml
oc login --username=user01@ost.com --password=rps@123 --server=https://api.ocp4.rps.com:6443
```


