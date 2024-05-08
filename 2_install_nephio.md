# 2. Install Nephio

> **IMPORTANT:** Git clone test-infra repo which has Ansible playbook that deploys Nephio. \
> The original Nephio's test-infra does have a option that disables Kind installation, however this does not work properly as of May 2024. So we removed KinD installation from the Ansible playbook. \
> Change workdir to home directory and start initializing `init.sh`.

### Download init script & Set Environment Variables

```bash
##### -----=[ In mgmt cluster ]=----- ####

# download test-infra repo
git clone https://github.com/5gsec/nephio-test-infra.git test-infra

# set env for nephio
export NEPHIO_USER=$USER
export ANSIBLE_CMD_EXTRA_VAR_LIST="k8s.context=kubernetes-admin@mgmt kind.enabled=false host_min_vcpu=4 host_min_cpu_ram=8"
```

Then run `init.sh` to setup Nephio R2 with optional packages as well.

> **IMPORTANT:** Before running the init.sh script, the IP addresses of installation components, must be changed to  test environment(gitea and nephio-webui). \
> For this purpose, the following three tasks need to be performed. \
> 1 - Fork https://github.com/nephio-project/catalog.git repository to a arbitrary git repository. \
> 2 - Clone repository and change the 172.18.0.200 IP and 172.18.0.0/24 IP range strings. then commit changed codes\
> 3 - Among the codes in the downloaded test-infra directory, change the string written with the https://github.com/nephio-project/catalog.git address to the cloned repository address.

### Change IP address, ranges in catalog (task 2)

```bash
# clone codes from forked repository
git clone https://github.com/[Github_user_ID]/catalog.git

# set env for edit
GITEA_IP_ADDR=$(hostname -I | awk '{print $1}')
SUBNET_IP_RANGE=[subnet_IP_address_range]
NEPHIO_WEBUI_ADDR=[nephio_webui_address]

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
sed 's/free5gc/[Github_user_ID]/g' ./test-infra/e2e/provision/playbooks/roles/bootstrap/defaults/main.yaml
sed 's/free5gc/[Github_user_ID]/g' ./test-infra/e2e/provision/playbooks/roles/install/defaults/main.yaml
```

### Run init script

```bash
sudo -E ./test-infra/e2e/provision/init.sh
```

> **IMPORTANT:** If the `test-infra` directory was named as something else, the `init.sh` will attempt to download the original `test-infra` from nephio-project. Which will eventually install KinD. Therefore, keep the cloned directory of `nephio-test-infra` git as `test-infra`.

> **IMPORTANT:** While running init.sh, `kpt` will install package `gitea` which utilizes the PersistentVolumes created in the step 1.4. Please keep an eye on the namespace `gitea` to make sure `gitea/gitea-0` and `gitea/gitea-postgress-0` is running properly using the PersistentVolumes. If they happen to be in the status `CrashLoopBackOff`, please check if the directories designated for the PersistentVolumes are set with the correct file permissions.

### Monitoring Gitea Pods
To keep an eye on the Namespace `gitea`, please open another terminal and use the following command to check if the Pods are up and running properly:

```bash
watch -n 1 kubectl get pods -n gitea
```

### Setting Proper Permissions
Once the `gitea/gitea-0` and `gitea/gitea-postgresql-0` starts in the `gitea` namespace, set the permission of the local path using:

```bash
# change file permission in PV's hostpath
sudo chmod 777 ~/nephio -R 
```

<br></br>
---
|Before|Next|
|--|--|
|[ Go to Before Page](1_prerequsites.md) | [ Go to Next Page ](3_add_k8s_clusters_to_nephio.md)|