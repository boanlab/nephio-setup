# 1. Prerequsites

## 1.1 Prepare Servers

The test environment contains 4 servers:

> Official Nephio GCP utilizes a single server with 8 vCPU and 8GB of RAM. \
> However, we need more resources to properly set up Nephio and Free5gc directly on servers.

> Note that you need to configure the following IP addresses depending on your environment.

|Type|Spec|K8s Cluster Name|IP Address|Pod CIDR|
|--|--|--|--|--|
|Nephio Mgmt|8 vCPU / 32GB RAM | `mgmt` | 172.18.0.3 | 10.120.0.0/24 |
|Regional Cluster|8 vCPU / 8GB RAM | `regional` | 172.18.0.4 | 10.121.0.0/24 |
|Edge01 Cluster|8 vCPU / 8GB RAM | `edge01` | 172.18.0.5 | 10.122.0.0/24 |
|Edge02 Cluster|8 vCPU / 8GB RAM | `edge02` | 172.18.0.6 | 10.123.0.0/24 |

## 1.2 Install kubernetes

We use the following versions to set up Nephio and Free5gc.

- **Kubernetes**: v1.27.12
- **CRI**: Containerd
- **CNI**: Kindnet

### Install Kubernetes with Containerd
```bash
##### -----=[ In ALL clusters ]=----- ####

# update repo
$ sudo apt-get update

# add GPG key
$ sudo apt-get install -y curl ca-certificates gnupg
$ sudo install -m 0755 -d /etc/apt/keyrings
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
$ sudo chmod a+r /etc/apt/keyrings/docker.gpg

# add Docker repository
$ echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# update the Docker repo
$ sudo apt-get update

# install containerd
$ sudo apt-get install -y containerd.io

# set up the default config file
$ sudo mkdir -p /etc/containerd
$ sudo containerd config default | sudo tee /etc/containerd/config.toml
$ sudo sed -i "s/SystemdCgroup = false/SystemdCgroup = true/g" /etc/containerd/config.toml
$ sudo systemctl restart containerd

# add the key for kubernetes repo
$ curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# add sources.list.d
$ echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# update repo
$ sudo apt-get update

# enable ipv4.ip_forward
$ sudo sysctl -w net.ipv4.ip_forward=1

# turn off swap filesystem
$ sudo swapoff -a

# install kubernetes
$ sudo apt-get install -y kubelet kubeadm kubectl

# exclude kubernetes packages from updates
$ sudo apt-mark hold kubelet kubeadm kubectl
```

### Setup config.yaml for each cluster

Refer to the following yaml files.

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

### Initialize kubeadm

```bash
##### -----=[ In ALL clusters ]=----- ####

# enable br_netfilter
$ sudo modprobe br_netfilter
$ sudo bash -c 'echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables'

# initialize kubeadm
$ sudo kubeadm init --config=config.yaml --upload-certs

# make kubectl work for non-root user
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config

# disable master isolation
$ kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### Install Kindnet CNI

```bash
##### -----=[ In ALL clusters ]=----- ####

$ kubectl create -f https://raw.githubusercontent.com/aojea/kindnet/master/install-kindnet.yaml

$ kubectl get nodes
NAME   STATUS   ROLES           AGE    VERSION
np-m   Ready    control-plane   7d1h   v1.27.12
```

## 1.3 Install Packages

Nephio utilizes Ansible and kpt to deploy its packages.

### Install KPT

```bash
##### -----=[ In ALL clusters ]=----- ####

$ wget https://github.com/GoogleContainerTools/kpt/releases/download/v1.0.0-beta.44/kpt_linux_amd64
$ mv kpt_linux_amd64 kpt
$ chmod +x kpt
$ sudo mv ./kpt /usr/bin/
$ kpt version
```

### Install porchctl

```bash
##### -----=[ In ALL clusters ]=----- ####

$ wget https://github.com/nephio-project/porch/releases/download/v2.0.0/porchctl_2.0.0_linux_amd64.tar.gz
$ tar xvfz ./porchctl_2.0.0_linux_amd64.tar.gz 
$ sudo mv porchctl /usr/bin
$ porchctl version
```

### Install Docker

```bash
##### -----=[ In ALL clusters ]=----- ####

# add GPG key
$ sudo apt-get install -y ca-certificates gnupg
$ sudo install -m 0755 -d /etc/apt/keyrings
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
$ sudo chmod a+r /etc/apt/keyrings/docker.gpg

# add docker repository
$ echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# update the docker repo
$ sudo apt-get update

# install docker
$ sudo apt-get install -y docker-ce

# add user to docker
$ sudo usermod -aG docker $USER

# bypass to run docker command
$ sudo chmod 666 /var/run/docker.sock
```

### Install Open vSwitch

```bash
##### -----=[ In ALL clusters ]=----- ####

# install Open vSwitch
$ sudo apt-get install -y openvswitch-switch

# install networking tools (e.g., ifconfig)
$ sudo apt-get install -y net-tools
```

### Install gtp5g, a Kernel module for 5G

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

Nephio utilizes `gitea` that needs 2 local path PVs; thus, create PV in `mgmt` cluster.

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
    path: /home/[USER]/nephio/gitea/data-gitea-0 # change here
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
    path: /home/[USER]/nephio/gitea/data-gitea-postgresql-0 # change here
```

### Save the above YAML as 'gitea-pv.yaml' and apply it

```bash
##### -----=[ In mgmt cluster ]=----- ####

# create local paths
$ mkdir -p ~/nephio/gitea/data-gitea-0
$ mkdir -p ~/nephio/gitea/data-gitea-postgresql-0

# modify gitea-pv.yaml
$ sed -i 's/[USER]/$USER/g' gitea-pv.yaml

# apply the YAML file to create 2 local path PVs
$ kubectl apply -f gitea-pv.yaml
```
<br></br>
---
|Index|  |  |  |  |  |  |Next|
|--|--|--|--|--|--|--|--|
|[ Go to Index Page](README.md) |  |  |  |  |  |  | [ Go to Next Page ](2_install_nephio.md)|