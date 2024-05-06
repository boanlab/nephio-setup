# 2. Install Nephio

> **IMPORTANT:** Git clone test-infra repo which has Ansible playbook that deploys Nephio. \
> The original Nephio's test-infra cannot provision without Kind, so we removed KinD by force. \
> Change workdir to home directory and start initializing `init.sh`.

### Download init script & Set env

```bash
##### -----=[ In mgmt cluster ]=----- ####

# download test-infra repo
git clone https://github.com/free5gc/nephio-test-infra.git test-infra

# set env for nephio
export NEPHIO_USER=$USER
export ANSIBLE_CMD_EXTRA_VAR_LIST="k8s.context=kubernetes-admin@mgmt kind.enabled=false host_min_vcpu=4 host_min_cpu_ram=8"
```

> **IMPORTANT:** Before running the init.sh script, the IP addresses of installation components, must be changed to  test environment(gitea and nephio-webui). \
> For this purpose, the following three tasks need to be performed. \
> 1 - Fork https://github.com/nephio-project/catalog.git repository to a arbitrary git repository. \
> 2 - Clone repository and change the 172.18.0.200 IP and 172.18.0.0/24 IP range strings. then commit changed codes\
> 3 - Among the codes in the downloaded test-infra directory, change the string written with the https://github.com/nephio-project/catalog.git address to the cloned repository address.

### Change ip address, ranges in catalog (task 2)

```bash
# clone codes from forked repository
git clone https://github.com/[arbitrary_repo_name]/catalog.git

# set env for edit
GITEA_IP_ADDR=$(hostname -I | awk '{print $1}')
SUBNET_IP_RANGE=[subnet_ip_range]
NEPHIO_WEBUI_ADDR=[nephio_webui_addr]

# using sed command, change ip address, ip ranges
sed 's/172.18.0.200/$GITEA_IP_ADDR/g' catalog/distros/gcp/nephio-mgmt/nephio-controllers/app/deployment-token-controller.yaml
sed 's/172.18.0.200/$GITEA_IP_ADDR/g' catalog/distros/sandbox/gitea/service-gitea.yaml
sed 's/172.18.0.200\/20/$SUBNET_IP_RANGE/g' catalog/distros/sandbox/metallb-sandbox-config/ipaddresspool.yaml
sed 's/172.18.0.200/$GITEA_IP_ADDR/g' catalog/distros/sandbox/repo-porch.yaml
sed 's/172.18.0.200/$GITEA_IP_ADDR/g' catalog/distros/sandbox/repository/set-values.yaml
sed 's/172.18.0.200/$GITEA_IP_ADDR/g' catalog/nephio/core/nephio-operator/app/controller/deployment-controller.yaml
sed 's/172.18.0.200/$GITEA_IP_ADDR/g' catalog/nephio/core/nephio-operator/app/controller/deployment-token-controller.yaml
sed 's/172.18.0.200/$GITEA_IP_ADDR/g' catalog/nephio/optional/rootsync/rootsync.yaml
sed 's/172.18.0.200/$GITEA_IP_ADDR/g' catalog/nephio/optional/rootsync/set-values.yaml
sed 's/172.18.0.200/$NEPHIO_WEBUI_ADDR/g' catalog/nephio/optional/webui/service.yaml

# commit changed code
cd catalog
git add .
git commit -m "changed ip address"
git push
cd ..
```

### Change catalog path in test-infra (task 3)

```bash
# using sed command, change repository name
sed 's/free5gc/[forked repository name]/g' ./test-infra/e2e/provision/playbooks/roles/bootstrap/defaults/main.yaml
sed 's/free5gc/[forked repository name]/g' ./test-infra/e2e/provision/playbooks/roles/install/defaults/main.yaml
```

### Run init script

```bash
sudo -E ./test-infra/e2e/provision/init.sh
```

> **IMPORTANT:** After running `init.sh`, we need to change files permission under `nephio` directory. \
> keep monitor starting `gitea/gitea-0` and `gitea/gitea-postgresql-0`, then change permission. \
> Otherwilse, the Nephio will get stuck while installing.

###  keep monitoring starting pods

```bash
# monitor with watch command
watch -n 1 kubectl get pods -n gitea
```

### Once the `gitea/gitea-0` and `gitea/gitea-postgresql-0` starts in Nephio, change permissions:

```bash
# change file permission in PV's hostpath
sudo chmod 777 /home/[User]/nephio -R 
```

<br></br>
---
|Before|  |  |  |  |  |  |Next|
|--|--|--|--|--|--|--|--|
|[ Go to Before Page](1_prerequsites.md) |  |  |  |  |  |  | [ Go to Next Page ](3_add_k8s_clusters_to_nephio.md)|