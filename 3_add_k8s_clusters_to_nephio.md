# 3. Adding K8s Clusters to Nephio

## 3.1 Registering Clusters to Nephio

### Add `regional` cluster to Nephio

```bash
##### -----=[ In mgmt cluster ]=----- ####

# add regional cluster to nephio
kubectl get repositories
porchctl rpkg get --name nephio-workload-cluster
porchctl rpkg clone -n default catalog-infra-capi-b0ae9512aab3de73bbae623a3b554ade57e15596 --repository mgmt regional
porchctl rpkg pull -n default mgmt-08c26219f9879acdefed3469f8c3cf89d5db3868 regional
kpt fn eval --image gcr.io/kpt-fn/set-labels:v0.2.0 regional -- "nephio.org/site-type=regional" "nephio.org/region=us-west1"
porchctl rpkg push -n default mgmt-08c26219f9879acdefed3469f8c3cf89d5db3868 regional
porchctl rpkg propose -n default mgmt-08c26219f9879acdefed3469f8c3cf89d5db3868
porchctl rpkg approve -n default mgmt-08c26219f9879acdefed3469f8c3cf89d5db3868
```

### Add `edge01` and `edge02` clusters to Nephio:

```bash
##### -----=[ In mgmt cluster ]=----- ####

#add edge01, edge02 cluster to nephio using 002-edge-clusters.yaml file
kubectl apply -f test-infra/e2e/tests/free5gc/002-edge-clusters.yaml
```

### Check for repositories registered in Nephio

```bash
##### -----=[ In mgmt cluster ]=----- ####

#check repositories resource
kubectl get repositories

NAME                        TYPE   CONTENT   DEPLOYMENT   READY   ADDRESS
...
edge01                      git    Package   true         False    http://172.18.0.200:3000/nephio/edge01.git
edge02                      git    Package   true         False    http://172.18.0.200:3000/nephio/edge02.git
mgmt                        git    Package   true         True     http://172.18.0.200:3000/nephio/mgmt.git
mgmt-staging                git    Package   false        True     http://172.18.0.200:3000/nephio/mgmt-staging.git
...
regional                    git    Package   true         False    http://172.18.0.200:3000/nephio/regional.git
```

<br></br>
> **IMPORTANT:** As you can see, the repositories are directing to wrong gitea service port which is `http://[ip_addr]:3000`. \
> We need to modify port. You can achieve this by visiting the gitea service. \
> the default login credentials are `nephio` / `secret`

### check gitea service address

```bash
kubectl get svc -n gitea

NAME                  TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                       AGE
gitea                 LoadBalancer   10.100.64.97     172.18.0.200   22:30941/TCP,3000:30104/TCP   7d3h
gitea-memcached       ClusterIP      10.100.218.39    <none>         11211/TCP                     7d3h
gitea-postgresql      ClusterIP      10.104.192.111   <none>         5432/TCP                      7d3h
gitea-postgresql-hl   ClusterIP      None             <none>         5432/TCP                      7d3h
```

Once logged in, visit repository `nephio/mgmt` and modify the following files:
- `mgmt/regional-repo/repo-porch.yaml`
- `mgmt/edge01-repo/repo-porch.yaml`
- `mgmt/edge02-repo/repo-porch.yaml`

### Replace any `172.18.0.200` to the gitea's service port. An example will be like below:

```yaml
# change gitea repo port!

apiVersion: config.porch.kpt.dev/v1alpha1
kind: Repository
metadata: # kpt-merge: default/example-cluster-name
  name: regional
  namespace: default
  annotations:
    internal.kpt.dev/upstream-identifier: config.porch.kpt.dev|Repository|default|example-cluster-name
    nephio.org/cluster-name: regional
spec:
  content: Package
  deployment: true
  git:
    branch: main
    directory: /
    repo: http://172.18.0.200:3000/nephio/regional.git # change here to gitea repo service port
    secretRef:
      name: regional-access-token-porch
  type: git
```

### Once you have committed the change, the porch-system which is running in `mgmt` cluster will notice that the repositories were changed. This might take a little bit of time, in some time later, check the repositories registered in Nephio by:

```bash
##### -----=[ In mgmt cluster ]=----- ####
kubectl get repositories

NAME                        TYPE   CONTENT   DEPLOYMENT   READY   ADDRESS
...
edge01                      git    Package   true         True    http://172.18.0.200:3000/nephio/edge01.git
edge02                      git    Package   true         True    http://172.18.0.200:3000/nephio/edge02.git
mgmt                        git    Package   true         True    http://172.18.0.200:3000/nephio/mgmt.git
mgmt-staging                git    Package   false        True    http://172.18.0.200:3000/nephio/mgmt-staging.git
regional                    git    Package   true         True    http://172.18.0.200:3000/nephio/regional.git
```
### If the addresses were changed to the designated gitea service's IP, the `READY` field will be changed to `True`. If this step was successfully performed, the edge and regional clusters can Join without any problem.

## 3.2 Clusters Joining Nephio
### In order for regional, edge clusters to join Nephio, they require `secrets` to gain access to `mgmt` cluster's gitea service. The secrets are automatically generated in step 3.1. You can check them by:

```bash
##### -----=[ In mgmt cluster ]=----- ####
kubectl get secrets --all-namespaces

NAMESPACE                           NAME                                              TYPE                       DATA   AGE
...
default                             edge01-access-token-configsync                    kubernetes.io/basic-auth   3      6d21h
default                             edge01-access-token-porch                         kubernetes.io/basic-auth   3      6d21h
default                             edge02-access-token-configsync                    kubernetes.io/basic-auth   3      6d21h
default                             edge02-access-token-porch                         kubernetes.io/basic-auth   3      6d21h
default                             regional-access-token-configsync                  kubernetes.io/basic-auth   3      6d22h
default                             regional-access-token-porch                       kubernetes.io/basic-auth   3      6d22h
...
```

Look for secrets:
- edge01-access-token-configsync
- edge02-access-token-configsync
- regional-access-token-configsync

The clusters need to have those secrets registered in their clusters. This means, for example, if `regional` cluster was to gain access to `mgmt`'s gitea service, the `regional` shall also have the `regional-access-token-configsync`. This can be achieved by

```bash
##### -----=[ In mgmt cluster ]=----- ####
# Before copy&paste, change username, ip address, homePath

kubectl get secret regional-access-token-configsync -o yaml > regional-secret.yaml
scp regional-secret.yaml [User]@[cluster_ip_address]:/home/[User] # change mgmt cluster machine's username, homepath, ip address 
```

This will export the secret to regional cluster as `regional-secret.yaml`. In the destination cluster, in this case `regional`, modify a bit of the secret.

```yaml
apiVersion: v1
data:
  password: OTE2YjNlZDlhZWQ5M2E5NTNjYjk1NTI1MmQ4YzBjN2QzMDk2Mzk3NA==
  token: OTE2YjNlZDlhZWQ5M2E5NTNjYjk1NTI1MmQ4YzBjN2QzMDk2Mzk3NA==
  username: bmVwaGlv
kind: Secret
...
  creationTimestamp: "2024-04-08T07:01:41Z"
  name: regional-access-token-configsync
  namespace: config-management-system # change here, default -> config-management-system
  ownerReferences:
...
```

Change the namespace of the token to `config-management-system` from `default`. Perform this for all `edge01` and `edge02` clusters as well. In short, to proceed, we have to have the following secrets:
- `config-management-system/regional-access-token-configsync` in `regional` cluster
- `config-management-system/edge01-access-token-configsync` in `edge01` cluster
- `config-management-system/edge02-access-token-configsync` in `edge02` cluster

### But before apply secrets, install `configsync` in the clusters joining Nephio as follows:

```bash
##### -----=[ In regional, edge01, edge02 clusters ]=----- ####
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/nephio/core/configsync@main
kpt fn render configsync
kpt live init configsync
kpt live apply configsync --reconcile-timeout=15m --output=table

kpt pkg get https://github.com/nephio-project/catalog.git/nephio/optional/rootsync@main
```

### This will automatically install `configsync`. and next, apply secrets in clusters

```bash
##### -----=[ In regional, edge01, edge02 clusters ]=----- ####
kubectl apply -f regional-secret.yaml  # edge01-secret.yaml, edge02-secret.yaml in other clusters
```

### After apply secrets, we need to install `rootsync` in the clusters joining Nephio. 
### in order for `rootsync` to know the target gitea service that the cluster needs to access, we need to modify `rootsync/rootsync.yaml` Refer to the following rootsync for detailed informaton:

<details>
  <summary>regional rootsync.yaml</summary>
  
``` yaml
apiVersion: configsync.gke.io/v1beta1
kind: RootSync
metadata: # kpt-merge: config-management-system/regional
  name: regional
  namespace: config-management-system
  annotations:
    internal.kpt.dev/upstream-identifier: 'configsync.gke.io|RootSync|config-management-system|example-cluster-name'
spec:
  sourceFormat: unstructured
  git:
    repo: http://172.18.0.200:3000/nephio/regional.git # change gitea repo ip address
    branch: main
    auth: token
    secretRef:
      name: regional-access-token-configsync
  ```
</details>

<details>
  <summary>edge01 rootsync.yaml</summary>
  
``` yaml
apiVersion: configsync.gke.io/v1beta1
kind: RootSync
metadata: # kpt-merge: config-management-system/edge01
  name: edge01
  namespace: config-management-system
  annotations:
    internal.kpt.dev/upstream-identifier: 'configsync.gke.io|RootSync|config-management-system|example-cluster-name'
spec:
  sourceFormat: unstructured
  git:
    repo: http://172.18.0.200:3000/nephio/edge01.git # change gitea repo ip address
    branch: main
    auth: token
    secretRef:
      name: edge01-access-token-configsync
  ```
</details>

<details>
  <summary>edge02 rootsync.yaml</summary>
  
``` yaml
apiVersion: configsync.gke.io/v1beta1
kind: RootSync
metadata: # kpt-merge: config-management-system/edge02
  name: edge02
  namespace: config-management-system
  annotations:
    internal.kpt.dev/upstream-identifier: 'configsync.gke.io|RootSync|config-management-system|example-cluster-name'
spec:
  sourceFormat: unstructured
  git:
    repo: http://172.18.0.200:3000/nephio/edge02.git # change gitea repo ip address
    branch: main
    auth: token
    secretRef:
      name: edge02-access-token-configsync
  ```
</details>

### Once the rootsync/rootsync files were modified, install rootsync by:

```bash
##### -----=[ In regional, edge01, edge02 clusters ]=----- ####
kpt live init rootsync
kpt live apply rootsync --reconcile-timeout=15m --output=table
```

### This might fail one time, try again. If the problem proceeds, check for the error message and make sure the secrets are registered in respective clusters properly. If you see `root-reconciler` the namespace `config-management-system` like below, this is working as intended.

``` bash
##### -----=[ In regional, edge01, edge02 clusters ]=----- ####
kubectl get pods -n config-management-system
NAME                                          READY   STATUS    RESTARTS       AGE
config-management-operator-6946b77565-wxwvd   1/1     Running   0              6d20h
reconciler-manager-5b5d8557-wf69f             2/2     Running   0              6d20h
root-reconciler-regional-79949ff68-r5jvs      4/4     Running   87 (78m ago)   6d20h
```

### Also, check if `root-reconciler` can actually access the gitea properly by:

```bash
##### -----=[ In regional, edge01, edge02 clusters ]=----- ####
kubectl logs -n config-management-system root-reconciler-regional-79949ff68-r5jvs -c git-sync
INFO: detected pid 1, running init handler
I0415 04:31:40.264048 12 main.go:1101] "level"=1 "msg"="setting up git credential store"
...
I0415 05:50:31.188928 12 main.go:585] "level"=1 "msg"="next sync" "wait_time"=15000000000
I0415 05:50:46.194475 12 cmd.go:48] "level"=5 "msg"="running command" "cwd"="/repo/source/rev" "cmd"="git rev-parse HEAD"
I0415 05:50:46.198248 12 cmd.go:48] "level"=5 "msg"="running command" "cwd"="/repo/source/rev" "cmd"="git ls-remote -q http://172.18.0.200:3000/nephio/regional.git refs/heads/main"
I0415 05:50:46.232708 12 main.go:1065] "level"=1 "msg"="no update required" "rev"="HEAD" "local"="059047c546d8c944a3bca69c1c03e81fb9a52d14" "remote"="059047c546d8c944a3bca69c1c03e81fb9a52d14"
I0415 05:50:46.232790 12 main.go:585] "level"=1 "msg"="next sync" "wait_time"=15000000000
```

> If the message keeps coming out as if the target gitea address is not accessable, perform `curl` to the gitea service. \
> If the gitea is stil down, restart the `metallb-system`'s daemonsets in `mgmt` cluster. There are some frequent occurances with the `metallb-system/speaker` daemonset not serving the gitea properly. \
> In this case, restart the daemonset by:

```bash
##### -----=[ In mgmt cluster ]=----- ####
kubectl rollout restart daemonsets -n metallb-system
```

> This will solve the issue and make the load balancer IP work again.

<br></br>
---
|Before|Next|
|--|--|
|[ Go to Before Page](2_install_nephio.md) | [ Go to Next Page ](4_configure_network_topology.md)|