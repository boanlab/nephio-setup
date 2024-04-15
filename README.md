# nephio-setup
This document demonstrates the installation guide for Nephio R2 and [Free5CP demo](https://docs.nephio.org/docs/guides/user-guides/exercise-1-free5gc/).

## Table of Contents
1. Prerequsites
2. Initialize Nephio
3. Adding K8s Clusters to Nephio
4. Configure Network Topology
5. Deploying Free5gc-cp
6. Deploying UPF, AMF and SMF
7. Deploying UERANSIM

## 1. Prerequsites
### 1.1 Servers
The environment will utilize 4 servers:
> Official Nephio GCP utilizes single server with 8 vCPU and 8GB RAM. However, for our purpose, we will be using extra resources.

|Type|Spec|K8s Cluster Name|IP Address|Pod CIDR|
|--|--|--|--|--|
|Nephio Mgmt|8 vCPU / 32GB RAM | `mgmt` | 10.10.0.120 | 10.120.0.0/16 |
|Regional Cluster|8 vCPU / 8GB RAM | `regional` | 10.10.0.121 | 10.121.0.0/16 |
|Edge01 Cluster|8 vCPU / 8GB RAM | `edge01` | 10.10.0.122 | 10.122.0.0/16 |
|Edge02 Cluster|8 vCPU / 8GB RAM | `edge02` | 10.10.0.123 | 10.123.0.0/16 |

### 1.2 Kubernetes
The official environment provisions Kubernetes with KinD. To have as similar environment as possible, we will be using:
- **Kubernetes**: v1.27.4
- **CRI**: Containerd
- **CNI**: Kindnet

Install Kubernetes with Containerd CRI with:
```bash
sudo apt-get install git -y
git clone https://github.com/boanlab/tools.git
./tools/containerd/install-containerd.sh

sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.27/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.27/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo sysctl -w net.ipv4.ip_forward=1
sudo swapoff -a

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
Setup config.yaml for each clusters. Refer to following yaml files for information:

<details>
  <summary>mgmt_config.yaml</summary>
  
  ``` yaml
    apiVersion: kubeadm.k8s.io/v1beta3
    kind: ClusterConfiguration
    networking:
      podSubnet: "10.120.0.0/16" 
    clusterName: "mgmt"
  ```
</details>

<details>
  <summary>regional_config.yaml</summary>
  
  ``` yaml
    apiVersion: kubeadm.k8s.io/v1beta3
    kind: ClusterConfiguration
    networking:
      podSubnet: "10.121.0.0/16" 
    clusterName: "regional"
  ```
</details>

<details>
  <summary>edge01_config.yaml</summary>
  
  ``` yaml
    apiVersion: kubeadm.k8s.io/v1beta3
    kind: ClusterConfiguration
    networking:
      podSubnet: "10.122.0.0/16" 
    clusterName: "edge01"
  ```
</details>

<details>
  <summary>edge02_config.yaml</summary>
  
  ``` yaml
    apiVersion: kubeadm.k8s.io/v1beta3
    kind: ClusterConfiguration
    networking:
      podSubnet: "10.123.0.0/16" 
    clusterName: "edge02"
  ```
</details>

Initialize kubeadm using:
```bash
sudo kubeadm init --config=config.yaml --upload-certs
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```
> We are going to taint master node so that Pods can be provisioned in the master node as well (single node setting)

Then, install Kindnet as CNI by:
```bash
kubectl create -f https://raw.githubusercontent.com/aojea/kindnet/master/install-kindnet.yaml
kubectl get nodes
NAME   STATUS   ROLES           AGE    VERSION
np-m   Ready    control-plane   7d1h   v1.27.12
```

### 1.4 Install Packages
> Nephio utilizes Ansible and kpt to deploy its packages, so make all 4 machines be able to perform `sudo` without password prompt.
> Install all packages in all 4 machines

Install KPT by:
```bash
wget https://github.com/GoogleContainerTools/kpt/releases/download/v1.0.0-beta.44/kpt_linux_amd64
mv kpt_linux_amd64 kpt
chmod +x kpt
sudo mv ./kpt /usr/bin/
kpt version
```

Install porchctl by:
```bash
wget https://github.com/nephio-project/porch/releases/download/v2.0.0/porchctl_2.0.0_linux_amd64.tar.gz
tar xvfz ./porchctl_2.0.0_linux_amd64.tar.gz 
sudo mv porchctl /usr/bin
porchctl version
```

Install Docker by:
```bash
./tools/containers/install-docker.sh
```

Also, for future network SDN connection across clusters, we will be using OVS. So install OVS by:
```bash
sudo apt-get install openvswitch-switch -y  # for ovs-vsctl
sudo apt-get install net-tools -y  # for ifconfig
```

For worker nodes, install gtp5g which is a Kernel module for 5G:
```bash
##### -----=[ In ALL Worker clusters ]=----- ####
wget https://github.com/free5gc/gtp5g/archive/refs/tags/v0.8.3.tar.gz
tar xvfz v0.8.3.tar.gz
cd gtp5g-0.8.3/
sudo make install
```

For worker nodes, install some more packages by:
```bash
##### -----=[ In ALL Worker clusters ]=----- ####
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/nephio/core/workload-crds@main
kpt fn render workload-crds
kpt live init workload-crds
kpt live apply workload-crds --reconcile-timeout=15m --output=table

kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/infra/capi/multus@main
kpt fn render multus
kpt live init multus
kpt live apply multus --reconcile-timeout=15m --output=table
```

### 1.5 Prepare Nephio
Nephio utilizes `gitea` and `gitea` utilizes 2 local path PVs. Therefore, create a PV with:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: data-gitea-0
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  volumeMode: Filesystem
  hostPath:
    path: /home/boan/nephio/gitea/data-gitea-0
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: data-gitea-postgresql-0
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  volumeMode: Filesystem
  hostPath:
    path: /home/boan/nephio/gitea/data-gitea-postgresql-0
```

## 2. Initialize Nephio
> **IMPORTANT:** Perform this in `mgmt` cluster.

Git clone test-infra which has Ansible playbook that deploys Nephio. The original Nephio's test-infra cannot provision without Kind, so we have removed KinD by force. Change workdir to home directory and start initializing `init.sh`.
```
cd ~
git clone https://github.com/boanlab/nephio-test-infra.git test-infra
export NEPHIO_USER=$USER
export ANSIBLE_CMD_EXTRA_VAR_LIST="k8s.context=kubernetes-admin@mgmt kind.enabled=false host_min_vcpu=4 host_min_cpu_ram=8"
sudo -E ./test-infra/e2e/provision/init.sh
```

The installation sequence normally takes around 20 ~ 30 minutes, keep monitoring namespaces by:
```
watch -n 1 kubectl get ns
```

Once the `gitea/gitea-0` and `gitea/gitea-postgresql-0` starts in Nephio, we need to perform:
```bash
chmod 777 /home/boan/nephio -R
```
Otherwilse, the Nephio will get stuck while installing.

## 3. Adding K8s Clusters to Nephio
### 3.1 Registering Clusters to Nephio
Once step 2 was finished **without any errors**, add `regional` cluster to Nephio by:
```bash
##### -----=[ In mgmt cluster ]=----- ####
kubectl get repositories
porchctl rpkg get --name nephio-workload-cluster
porchctl rpkg clone -n default catalog-infra-capi-b0ae9512aab3de73bbae623a3b554ade57e15596 --repository mgmt regional
porchctl rpkg pull -n default mgmt-08c26219f9879acdefed3469f8c3cf89d5db3868 regional
kpt fn eval --image gcr.io/kpt-fn/set-labels:v0.2.0 regional -- "nephio.org/site-type=regional" "nephio.org/region=us-west1"
porchctl rpkg push -n default mgmt-08c26219f9879acdefed3469f8c3cf89d5db3868 regional
porchctl rpkg propose -n default mgmt-08c26219f9879acdefed3469f8c3cf89d5db3868
porchctl rpkg approve -n default mgmt-08c26219f9879acdefed3469f8c3cf89d5db3868
```

This will create draft `regional` to `gitea`. Then add `edge01` and `edge02` clusters to Nephio by:
```bash
##### -----=[ In mgmt cluster ]=----- ####
kubectl apply -f test-infra/e2e/tests/free5gc/002-edge-clusters.yaml
```

Then after a bit of time, check for repositories registered in Nephio by
```
kubectl get repositories
NAME                        TYPE   CONTENT   DEPLOYMENT   READY   ADDRESS
...
edge01                      git    Package   true         False    http://172.18.0.200:3000/nephio/edge01.git
edge02                      git    Package   true         False    http://172.18.0.200:3000/nephio/edge02.git
mgmt                        git    Package   true         True     http://10.10.0.131:3000/nephio/mgmt.git
mgmt-staging                git    Package   false        True     http://10.10.0.131:3000/nephio/mgmt-staging.git
...
regional                    git    Package   true         False    http://172.18.0.200:3000/nephio/regional.git
```

As you can see, the repositories are directing to wrong gitea address which is `http://172.18.0.200:3000`. We need to modify this manually to our address. You can achieve this by visiting the gitea service. The gitea will be running in `http://10.10.0.131:3000/`, the default login credentials are as it follows:
- **username**: nephio
- **password**: secret

Once logged in, visit repository `nephio/mgmt` and modify the following files:
- `mgmt/regional-repo/repo-porch.yaml`
- `mgmt/edge01-repo/repo-porch.yaml`
- `mgmt/edge02-repo/repo-porch.yaml`

Replace any `172.18.0.200` to the gitea's service IP. An example will be like below:
```yaml
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
    repo: http://10.10.0.131:3000/nephio/regional.git
    secretRef:
      name: regional-access-token-porch
  type: git
```

Once you have committed the change, the porch-system which is running in `mgmt` cluster will notice that the repositories were changed. This might take a little bit of time, in some time later, check the repositories registered in Nephio by:
```bash
$ kubectl get repositories
NAME                        TYPE   CONTENT   DEPLOYMENT   READY   ADDRESS
...
edge01                      git    Package   true         True    http://10.10.0.131:3000/nephio/edge01.git
edge02                      git    Package   true         True    http://10.10.0.131:3000/nephio/edge02.git
mgmt                        git    Package   true         True    http://10.10.0.131:3000/nephio/mgmt.git
mgmt-staging                git    Package   false        True    http://10.10.0.131:3000/nephio/mgmt-staging.git
regional                    git    Package   true         True    http://10.10.0.131:3000/nephio/regional.git
```
If the addresses were changed to the designated gitea service's IP, the `READY` field will be changed to `True`. If this step was successfully performed, the edge and regional clusters can Join without any problem.

### 3.2 Clusters Joining Nephio
In order for regional, edge clusters to join Nephio, they require `secrets` to gain access to `mgmt` cluster's gitea service. The secrets are automatically generated in step 3.1. You can check them by:
```bash
$ kubectl get secrets --all-namespaces
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

```
##### -----=[ In mgmt cluster ]=----- ####
kubectl get secret regional-access-token-configsync -o yaml > regional-secret.yaml
scp regional-secret.yaml boan@10.10.0.121:/home/boan
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
  namespace: config-management-system
  ownerReferences:
...
```
Change the namespace of the token to `config-management-system` from `default`. Perform this for all `edge01` and `edge02` clusters as well. In short, to proceed, we have to have the following secrets:
- `config-management-system/regional-access-token-configsync` in `regional` cluster
- `config-management-system/edge01-access-token-configsync` in `edge01` cluster
- `config-management-system/edge02-access-token-configsync` in `edge02` cluster

Then, install `configsync` and `rootsync` in the clusters joining Nephio by:
```bash
##### -----=[ In ALL Worker clusters ]=----- ####
kpt pkg get --for-deployment https://github.com/nephio-project/catalog.git/nephio/core/configsync@main
kpt fn render configsync
kpt live init configsync
kpt live apply configsync --reconcile-timeout=15m --output=table

kpt pkg get https://github.com/nephio-project/catalog.git/nephio/optional/rootsync@main
```

This will automatically install `configsync`. However, in order for `rootsync` to know the target gitea service that the cluster needs to access, we need to modify `rootsync/rootsync.yaml` Refer to the following rootsync for detailed informaton:

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
    repo: http://10.10.0.120:3000/nephio/regional.git
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
    repo: http://10.10.0.131:3000/nephio/edge01.git
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
    repo: http://10.10.0.131:3000/nephio/edge02.git
    branch: main
    auth: token
    secretRef:
      name: edge02-access-token-configsync
  ```
</details>

Once the rootsync/rootsync files were modified, install rootsync by:
```bash
##### -----=[ In ALL Worker clusters ]=----- ####
kpt live init rootsync
kpt live apply rootsync --reconcile-timeout=15m --output=table
```

This might fail one time, try again. If the problem proceeds, check for the error message and make sure the secrets are registered in respective clusters properly. If you see `root-reconciler` the namespace `config-management-system` like below, this is working as intended.

```$ kubectl get pods -n config-management-system
NAME                                          READY   STATUS    RESTARTS       AGE
config-management-operator-6946b77565-wxwvd   1/1     Running   0              6d20h
reconciler-manager-5b5d8557-wf69f             2/2     Running   0              6d20h
root-reconciler-regional-79949ff68-r5jvs      4/4     Running   87 (78m ago)   6d20h
```

Also, check if `root-reconciler` can actually access the gitea properly by:
```bash
kubectl logs -n config-management-system root-reconciler-regional-79949ff68-r5jvs -c git-sync
INFO: detected pid 1, running init handler
I0415 04:31:40.264048      12 main.go:1101] "level"=1 "msg"="setting up git credential store"
...
I0415 05:50:31.188928      12 main.go:585] "level"=1 "msg"="next sync" "wait_time"=15000000000
I0415 05:50:46.194475      12 cmd.go:48] "level"=5 "msg"="running command" "cwd"="/repo/source/rev" "cmd"="git rev-parse HEAD"
I0415 05:50:46.198248      12 cmd.go:48] "level"=5 "msg"="running command" "cwd"="/repo/source/rev" "cmd"="git ls-remote -q http://10.10.0.131:3000/nephio/regional.git refs/heads/main"
I0415 05:50:46.232708      12 main.go:1065] "level"=1 "msg"="no update required" "rev"="HEAD" "local"="059047c546d8c944a3bca69c1c03e81fb9a52d14" "remote"="059047c546d8c944a3bca69c1c03e81fb9a52d14"
I0415 05:50:46.232790      12 main.go:585] "level"=1 "msg"="next sync" "wait_time"=15000000000
```

> If the message keeps coming out as if the target gitea address is not accessable, perform `curl` to the gitea service. If the gitea is stil down, restart the `metallb-system`'s daemonsets in `mgmt` cluster. There are some frequent occurances with the `metallb-system/speaker` daemonset not serving the gitea properly. In this case, restart the daemonset by:
> ```bash
> ##### -----=[ In mgmt cluster ]=----- ####
> kubectl rollout restart daemonsets -n metallb-system
> ```
> This will solve the issue and make the load balancer IP work again.

If you seen all `edge01`, `edge02` and `regional` clusters having proper git-sync, you can proceed to to the Step 4.

# 4. Configure Network Topology
Nephio utilizes SR Linux to interconnect clusters. However, since we are using multiple servers, we need to connect them as if they were connected. Therefore, we will be using OVS to connect between SR Linux to each clusters.

## 4.1 Setup Containerlab - `mgmt`
Create a network topology file as follows:
```yaml
name: 5g
prefix: net
topology:
  kinds:
    srl:
      type: ixrd3
      image: ghcr.io/nokia/srlinux:22.11.2-116
  nodes:
    leaf:
      kind: srl
      ports:
        - 57400:57400
      mgmt-ipv4: 172.20.20.120
  links:
    - endpoints: ["leaf:e1-1", "host:sr-r"]
    - endpoints: ["leaf:e1-2", "host:sr-e1"]
    - endpoints: ["leaf:e1-3", "host:sr-e2"]
```

This will deploy a SR Linux having `e1-1`, `e1-2`, `e1-3` and those are connected to host's `sr-r`, `'sr-e1` and `sr-e1`. Also, we are going to connect each interfaces to a ovs tunnel that is connected to the remote server using VXLAN. Deploy containerlab using
```bash
##### -----=[ In mgmt cluster ]=----- ####
sudo containerlab deploy --topo topology.yaml
```

So for example, it will be something like:
```
leaf -- e1-1(veth) -- sr-r(veth) -- br-tun-r(ovs) -- vxlan0 -- VXLAN -- vxlan0 -- eth1(ovs)
```

Create OVS bridge in mgmt cluster by:
```bash
##### -----=[ In mgmt cluster ]=----- ####
sudo ovs-vsctl add-br br-tun-r
sudo ovs-vsctl add-br br-tun-e1
sudo ovs-vsctl add-br br-tun-e2

sudo ifconfig br-tun-r up
sudo ifconfig br-tun-e1 up
sudo ifconfig br-tun-e2 up
```

Then prepare to connect vxlans in mgmt cluster by
```bash
##### -----=[ In mgmt cluster ]=----- ####
sudo ovs-vsctl add-port br-tun-r vxlan0 -- set interface vxlan0 type=vxlan options:remote_ip=10.10.0.121 options:df_default=true options:egress_pkt_mark=0 options:in_key=flow options:out_key=flow options:dst_port=48317 options:tag=321
sudo ovs-vsctl add-port br-tun-e1 vxlan1 -- set interface vxlan1 type=vxlan options:remote_ip=10.10.0.122 options:df_default=true options:egress_pkt_mark=0 options:in_key=flow options:out_key=flow options:dst_port=48318 options:tag=321
sudo ovs-vsctl add-port br-tun-e2 vxlan2 -- set interface vxlan2 type=vxlan options:remote_ip=10.10.0.123 options:df_default=true options:egress_pkt_mark=0 options:in_key=flow options:out_key=flow options:dst_port=48319 options:tag=321
```

## 4.2 Setup OVS - `edge01`, `edge02` and `regional`
Then in each worker clusters, connect the otherpart by:
```bash
##### -----=[ In regional cluster ]=----- ####
sudo ovs-vsctl add-br eth1
sudo ifconfig eth1 up
sudo ovs-vsctl add-port eth1 vxlan0 -- set interface vxlan0 type=vxlan options:remote_ip=10.10.0.120 options:df_default=true options:egress_pkt_mark=0 options:in_key=flow options:out_key=flow options:dst_port=48317 options:tag=321
```

```bash
##### -----=[ In edge01 cluster ]=----- ####
sudo ovs-vsctl add-br eth1
sudo ifconfig eth1 up
sudo ovs-vsctl add-port eth1 vxlan0 -- set interface vxlan0 type=vxlan options:remote_ip=10.10.0.120 options:df_default=true options:egress_pkt_mark=0 options:in_key=flow options:out_key=flow options:dst_port=48318 options:tag=321
```

```bash
##### -----=[ In edge02 cluster ]=----- ####
sudo ovs-vsctl add-br eth1
sudo ifconfig eth1 up
sudo ovs-vsctl add-port eth1 vxlan0 -- set interface vxlan0 type=vxlan options:remote_ip=10.10.0.120 options:df_default=true options:egress_pkt_mark=0 options:in_key=flow options:out_key=flow options:dst_port=48319 options:tag=321
```

Also, create interfaces for `eth1.2` ~ `eth1.6`. These interfaces will be later connected to n3, n4, n6. Those interfaces will be connected to `eth1` OVS bridge in each worker nodes. So perform:
```bash
##### -----=[ In ALL Worker clusters ]=----- ####
sudo ip link add eth1.2 type veth peer name eth1.2-br
sudo ip link add eth1.3 type veth peer name eth1.3-br
sudo ip link add eth1.4 type veth peer name eth1.4-br
sudo ip link add eth1.5 type veth peer name eth1.5-br
sudo ip link add eth1.6 type veth peer name eth1.6-br

sudo ovs-vsctl add-port eth1 eth1.2-br
sudo ovs-vsctl add-port eth1 eth1.3-br
sudo ovs-vsctl add-port eth1 eth1.4-br
sudo ovs-vsctl add-port eth1 eth1.5-br
sudo ovs-vsctl add-port eth1 eth1.6-br
```

Till now, we have actually setup the connection. This corresponds to 
```bash
export E2EDIR=${E2EDIR:-$HOME/test-infra/e2e}
export LIBDIR=${LIBDIR:-$HOME/test-infra/e2e/lib}
export TESTDIR=${TESTDIR:-$HOME/test-infra/e2e/tests/free5gc}
./test-infra/e2e/provision/hacks/inter-connect_workers.sh
./test-infra/e2e/provision/hacks/vlan-interfaces.sh
```

## 4.3 Apply Nephio Networks
Then apply the network settings to Nephio by:
```
##### -----=[ In mgmt cluster ]=----- ####
kubectl apply -f test-infra/e2e/tests/free5gc/002-network.yaml
kubectl apply -f test-infra/e2e/tests/free5gc/002-secret.yaml
```

Also, we have to setup RawTopology as well. An example is like below:
```yaml
apiVersion: topo.nephio.org/v1alpha1
kind: RawTopology
metadata:
  name: nephio
spec:
  nodes:
    srl:
      address: 10.10.0.120:57400
      provider: srl.nokia.com
    mgmt:
      provider: host
    regional:
      provider: host
      labels:
        nephio.org/cluster-name: regional
    edge01:
      provider: host
      labels:
        nephio.org/cluster-name: edge01
    edge02:
      provider: host
      labels:
        nephio.org/cluster-name: edge02
  links:
  - endpoints:
    - { nodeName: srl, interfaceName: e1-1}
    - { nodeName: regional, interfaceName: eth1}
  - endpoints:
    - { nodeName: srl, interfaceName: e1-2}
    - { nodeName: edge01, interfaceName: eth1}
  - endpoints:
    - { nodeName: srl, interfaceName: e1-3}
    - { nodeName: edge02, interfaceName: eth1}
```

Be aware that the srl.address shall be provided as the `mgmt` cluster's SR Linux container. Apply this using:
```bash
kubectl create -f topo.yaml
```

# 5. Deploying Free5gc-cp
Now, deploy Free5gc-CP as usual: https://docs.nephio.org/docs/guides/user-guides/exercise-1-free5gc/#step-4-deploy-free5gc-control-plane-functions. The Nephio webui will be running in `10.10.0.132:7007`. 

The regional cluster utilizes host path PV to store data for `mongodb`. Create a new PV in `mgmt` cluster by:
```yaml
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
    path: /home/boan/nephio/mongodb/
```

Then
```bash
##### -----=[ In mgmt cluster ]=----- ####
kubectl create -f mongodb-pv.yaml
```

> Also, just like the `gitea` PVs in `mgmt` cluster, we need to manually `chmod` the local directory. Otherwise, the `mongodb` will not setup.
> ```bash
> sudo chmod 777 -R /home/boan/nephio/mongodb
> ```


Then deploy free5gc operators using the following command:
```bash
##### -----=[ In mgmt cluster ]=----- ####
kubectl apply -f test-infra/e2e/tests/free5gc/004-free5gc-operator.yaml
```

This will deploy 
- `free5gc/free5gc-operator` pods in `edge01`, `edge02` and `regional` clusters.
- `free5gc-cp/free5gc-NFV` pods in `regional` cluster. Ex) `free5gc-ausf`, `nrf`, `nssf`,` pcf`, `udm`, etc.

# 6. Deploying UPF, AMF and SMF
Once Free5gc-CP was setup properly, you can pretty much deploy other UPF, AMF and SMF as usual by:

```
##### -----=[ In mgmt cluster ]=----- ####
kubectl apply -f test-infra/e2e/tests/free5gc/005-edge-free5gc-upf.yaml
kubectl apply -f test-infra/e2e/tests/free5gc/006-regional-free5gc-amf.yaml
kubectl apply -f test-infra/e2e/tests/free5gc/006-regional-free5gc-smf.yaml
```
- TBD

# 7. Deploying UERANSIM
- TBD
