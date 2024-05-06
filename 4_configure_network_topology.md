# 4. Configure Network Topology
Nephio utilizes SR Linux to interconnect clusters. However, since we are using multiple servers, we need to connect them as if they were connected. Therefore, we will be using OVS to connect between SR Linux to each clusters.

## 4.1 Setup Containerlab
> **IMPORTANT:** Perform this in `mgmt` cluster.

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

This will deploy a SR Linux having `e1-1`, `e1-2`, `e1-3` and those are connected to host's `sr-r`, `'sr-e1` and `sr-e1`. Also, we are going to connect each interfaces to a ovs tunnel that is connected to the remote server using VXLAN. Deploy containerlab using codes
```bash
##### -----=[ In mgmt cluster ]=----- ####
$ sudo containerlab deploy --topo topology.yaml
```

So for example, it will be something like:
```
leaf -- e1-1(veth) -- sr-r(veth) -- br-tun-r(ovs) -- vxlan0 -- VXLAN -- vxlan0 -- eth1(ovs)
```

Create OVS bridge in mgmt cluster by:
```bash
##### -----=[ In mgmt cluster ]=----- ####
$ sudo ovs-vsctl add-br br-tun-r
$ sudo ovs-vsctl add-br br-tun-e1
$ sudo ovs-vsctl add-br br-tun-e2

$ sudo ifconfig br-tun-r up
$ sudo ifconfig br-tun-e1 up
$ sudo ifconfig br-tun-e2 up
```

Then prepare to connect vxlans in mgmt cluster by:
```bash
##### -----=[ In mgmt cluster ]=----- ####
#Before copy&paste, change remote_ip address to regional, edge01, edge02 ip address!

$ sudo ovs-vsctl add-port br-tun-r vxlan0 -- set interface vxlan0 type=vxlan options:remote_ip=172.18.0.121 options:dst_port=48317 options:tag=321
$ sudo ovs-vsctl add-port br-tun-r vxlan0 -- set interface vxlan0 type=vxlan options:remote_ip=172.18.0.122 options:dst_port=48318 options:tag=321
$ sudo ovs-vsctl add-port br-tun-r vxlan0 -- set interface vxlan0 type=vxlan options:remote_ip=172.18.0.123 options:dst_port=48319 options:tag=321
```

## 4.2 Setup OVS
> **IMPORTANT:** Perform this in `regional`, `edge01`, `edge02` cluster.

Then in each worker clusters, connect the otherpart by:
```bash
##### -----=[ In regional cluster ]=----- ####
#Before copy&paste, change remote ip to mgmt cluster machine's ip address!

$ sudo ovs-vsctl add-br eth1
$ sudo ifconfig eth1 up
$ sudo ovs-vsctl add-port eth1 vxlan0 -- set interface vxlan0 type=vxlan options:remote_ip=172.18.0.120 options:dst_port=48317 options:tag=321 # change remote ip to mgmt cluster machine's ip
```

```bash
##### -----=[ In edge01 cluster ]=----- ####
#Before copy&paste, change remote ip to mgmt cluster machine's ip address!

$ sudo ovs-vsctl add-br eth1
$ sudo ifconfig eth1 up
$ sudo ovs-vsctl add-port eth1 vxlan0 -- set interface vxlan0 type=vxlan options:remote_ip=172.18.0.120 options:dst_port=48318 options:tag=321 # change remote ip to mgmt cluster machine's ip
```

```bash
##### -----=[ In edge02 cluster ]=----- ####
#Before copy&paste, change remote ip to mgmt cluster machine's ip address!

$ sudo ovs-vsctl add-br eth1
$ sudo ifconfig eth1 up
$ sudo ovs-vsctl add-port eth1 vxlan0 -- set interface vxlan0 type=vxlan options:remote_ip=172.18.0.120 options:dst_port=48319 options:tag=321 # change remote ip to mgmt cluster machine's ip
```

Also, create interfaces for `eth1.2` ~ `eth1.6`. These interfaces will be later connected to n3, n4, n6. Those interfaces will be connected to `eth1` OVS bridge in each worker nodes. So perform:
```bash
##### -----=[ In regional, edge01, edge02 clusters ]=----- ####
$ sudo ip link add link eth1 name "eth1.2" type vlan id 2
$ sudo ip link add link eth1 name "eth1.3" type vlan id 3
$ sudo ip link add link eth1 name "eth1.4" type vlan id 4
$ sudo ip link add link eth1 name "eth1.5" type vlan id 5
$ sudo ip link add link eth1 name "eth1.6" type vlan id 6

$ sudo ifconfig eth1.2 up
$ sudo ifconfig eth1.3 up
$ sudo ifconfig eth1.4 up
$ sudo ifconfig eth1.5 up
$ sudo ifconfig eth1.6 up
```
After then, add-port with ovs-vsctl in mgmt cluster as follows:
```bash
##### -----=[ In mgmt cluster ]=----- ####
$ sudo ovs-vsctl add-port br-tun-r sr-r
$ sudo ovs-vsctl add-port br-tun-e1 sr-e1
$ sudo ovs-vsctl add-port br-tun-e2 sr-e2
```

## 4.3 Apply Nephio Networks
Then apply the network settings to Nephio as follows:
```bash
##### -----=[ In mgmt cluster ]=----- ####
$ kubectl apply -f test-infra/e2e/tests/free5gc/002-network.yaml
$ kubectl apply -f test-infra/e2e/tests/free5gc/002-secret.yaml
```

Also, we have to setup RawTopology as well. An example as follows:
```yaml
#change ip address to mgmt cluster's ip!

apiVersion: topo.nephio.org/v1alpha1
kind: RawTopology
metadata:
  name: nephio
spec:
  nodes:
    srl:
      address: 172.18.0.120:57400 # change here to mgmt ip address!
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
##### -----=[ In mgmt cluster ]=----- ####
$ kubectl create -f topo.yaml
```


<br></br>
---
|Before|Next|
|--|--|
|[ Go to Before Page](3_add_k8s_clusters_to_nephio.md) | [ Go to Next Page ](5_deploy_free5gc_cp.md)|