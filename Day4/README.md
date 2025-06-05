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
oc get route console -n openshift-console -o jsonpath='{.spec.host}'
```


