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
oc get secret cluster-name-import -n rps-ocp4-multicluster -o jsonpath='{.data.import\.yaml}' | base64 -d > import.yaml
```

On every managed Openshift cluster environment, we need to run this
```
oc apply -f import.yaml
```




