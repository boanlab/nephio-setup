# 5. Deploying Free5gc-cp
Now, deploy Free5gc-CP as usual: https://docs.nephio.org/docs/guides/user-guides/exercise-1-free5gc/#step-4-deploy-free5gc-control-plane-functions. The Nephio webui will be running in `172.18.0.132:7007` (for example). 

### The regional cluster utilizes host path PV to store data for `mongodb`. Create a new PV in `regional` cluster by:
```yaml

# change home directory Path to your username!
apiVersion: v1
kind: PersistentVolume
metadata:
  name: data-mongodb-0
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  volumeMode: Filesystem
  hostPath:
    path: /home/[User]/nephio/mongodb/
```

### Then apply pv files as follows:

```bash
##### -----=[ In regional cluster ]=----- ####
kubectl apply -f mongodb-pv.yaml
```

### Also, just like the `gitea` PVs in `mgmt` cluster, we need to manually `chmod` the local directory. Otherwise, the `mongodb` will not setup.

```bash
##### -----=[ In regional clusters ]=----- ####
# change home directory Path to your username!

sudo chmod 777 -R /home/[User]/nephio/mongodb
 ```

### Deploy free5gc operators

```bash
##### -----=[ In mgmt cluster ]=----- ####
kubectl apply -f test-infra/e2e/tests/free5gc/004-free5gc-operator.yaml
```

### Install other packages

```bash
##### -----=[ In regional, edge01, edge02 clusters ]=----- ####
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/nephio/core/workload-crds@main
kpt fn render workload-crds
kpt live init workload-crds
kpt live apply workload-crds --reconcile-timeout=15m --output=table

kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/infra/capi/multus@main
kpt fn render multus
kpt live init multus
kpt live apply multus --reconcile-timeout=15m --output=table

kubectl rollout restart deployment -n free5gc
```

### This will deploy 

1. `free5gc/free5gc-operator` pods in `edge01`, `edge02` and `regional` clusters.
2. `free5gc-cp/free5gc-NFV` pods in `regional` cluster. Ex) `free5gc-ausf`, `nrf`, `nssf`,`pcf`, `udm`, etc.


<br></br>
---
|Before|Next|
|--|--|
|[ Go to Before Page](4_configure_network_topology.md) | [ Go to Next Page ](6_deploy_upf_amf_smf.md)|