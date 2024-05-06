# 1. Prerequsites
## 1.1 Servers
The environment will utilize 4 servers:
> Official Nephio GCP utilizes single server with 8 vCPU and 8GB RAM. However, for our purpose, we will be using extra resources. 
> In the case of the IP address, you must configure it by changing it to the IP of the installation environment.

|Type|Spec|K8s Cluster Name|IP Address|Pod CIDR|
|--|--|--|--|--|
|Nephio Mgmt|8 vCPU / 32GB RAM | `mgmt` | 172.18.0.3 | 10.120.0.0/16 |
|Regional Cluster|8 vCPU / 8GB RAM | `regional` | 172.18.0.4 | 10.121.0.0/16 |
|Edge01 Cluster|8 vCPU / 8GB RAM | `edge01` | 172.18.0.5 | 10.122.0.0/16 |
|Edge02 Cluster|8 vCPU / 8GB RAM | `edge02` | 172.18.0.6 | 10.123.0.0/16 |

When installing in GCP, VPC settings are required, and the configured VPC network must be applied to the instance.
To set up a VPC network, proceed in the following order.
```
1. VPC Network > CREATE VPC NETWORK
2. In New Subnet, set the region to the same region as the instance and IP Range to 172.18.0.0/24, then press the create button.
```

After creating a VPC network, apply the VPC Network to the instance through the following procedure.
```
1. Create Instance > Advanced Options > Networking > Network Interface
2. Click 'Edit Network Interface', and select the created vpc network.
3. Click Boot Disk and change it to Ubuntu.
4. Click Create to create an instance.
```

## 1.2 Install kubernetes
The official environment provisions Kubernetes with KinD. To have as similar environment as possible, we will be using:
- **Kubernetes**: v1.27.12
- **CRI**: Containerd
- **CNI**: Kindnet

 

Install Kubernetes with Containerd CRI with:
```bash
##### -----=[ In ALL clusters ]=----- ####
$ sudo apt-get install git -y
$ git clone https://github.com/boanlab/tools.git
$ ./tools/containers/install-containerd.sh

$ sudo apt-get update
$ sudo apt-get install -y apt-transport-https ca-certificates curl gpg
$ curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.27/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
$ echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.27/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

$ sudo sysctl -w net.ipv4.ip_forward=1
$ sudo swapoff -a

$ sudo apt-get update
$ sudo apt-get install -y kubelet kubeadm kubectl
$ sudo apt-mark hold kubelet kubeadm kubectl
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

Initialize kubeadm as follows:
```bash
##### -----=[ In ALL clusters ]=----- ####
$ sudo modprobe br_netfilter
$ sudo bash -c 'echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables'
$ sudo kubeadm init --config=config.yaml --upload-certs
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
$ kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```
> We are going to taint master node so that Pods can be provisioned in the master node as well (single node setting)

Then, install Kindnet CNI as follows:
```bash
##### -----=[ In ALL clusters ]=----- ####
$ kubectl create -f https://raw.githubusercontent.com/aojea/kindnet/master/install-kindnet.yaml
$ kubectl get nodes
NAME   STATUS   ROLES           AGE    VERSION
np-m   Ready    control-plane   7d1h   v1.27.12
```

## 1.3 Install Packages
> Nephio utilizes Ansible and kpt to deploy its packages, so make all 4 machines be able to perform `sudo` without password prompt.
> Install all packages in all 4 machines

Install KPT as follows:
```bash
##### -----=[ In ALL clusters ]=----- ####
$ wget https://github.com/GoogleContainerTools/kpt/releases/download/v1.0.0-beta.44/kpt_linux_amd64
$ mv kpt_linux_amd64 kpt
$ chmod +x kpt
$ sudo mv ./kpt /usr/bin/
$ kpt version
```

Install porchctl as follows:
```bash
##### -----=[ In ALL clusters ]=----- ####
$ wget https://github.com/nephio-project/porch/releases/download/v2.0.0/porchctl_2.0.0_linux_amd64.tar.gz
$ tar xvfz ./porchctl_2.0.0_linux_amd64.tar.gz 
$ sudo mv porchctl /usr/bin
$ porchctl version
```

Install Docker as follows:
```bash
##### -----=[ In ALL clusters ]=----- ####
$ ./tools/containers/install-docker.sh
```

Also, for future network SDN connection across clusters, we will be using OVS. So install OVS as follows:
```bash
##### -----=[ In ALL clusters ]=----- ####
$ sudo apt-get install openvswitch-switch -y  # for ovs-vsctl
$ sudo apt-get install net-tools -y  # for ifconfig
```

For worker clusters, install gtp5g which is a Kernel module for 5G:
```bash
##### -----=[ In regional, edge01, edge02 clusters ]=----- ####
$ wget https://github.com/free5gc/gtp5g/archive/refs/tags/v0.8.3.tar.gz
$ tar xvfz v0.8.3.tar.gz
$ cd gtp5g-0.8.3/
$ sudo apt-get install gcc gcc-12 make
$ sudo make
$ sudo make install
$ lsmod | grep gtp
```

## 1.4 Prepare Nephio
Nephio utilizes `gitea` and `gitea` utilizes 2 local path PVs. Therefore, Create pv in `mgmt` cluster as follows:
```yaml
# Change hostPath to install env user path

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
    path: /home/boan/nephio/gitea/data-gitea-0 #change here
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
    path: /home/boan/nephio/gitea/data-gitea-postgresql-0 #change here
```
After save codes, apply it
```bash
##### -----=[ In mgmt cluster ]=----- ####
$ kubectl apply -f gitea-pv.yaml
```
<br></br>
---
|Index|Next|
|--|--|
|[ Go to Index Page](README.md) |  [ Go to Next Page ](2_install_nephio.md)|