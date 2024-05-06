# 6. Deploying UPF, AMF and SMF
Once Free5gc-CP was setup properly, you can pretty much deploy other `UPF`, `AMF` and `SMF` as usual by:
```bash
##### -----=[ In mgmt cluster ]=----- ####
$ kubectl apply -f test-infra/e2e/tests/free5gc/005-edge-free5gc-upf.yaml
$ kubectl apply -f test-infra/e2e/tests/free5gc/006-regional-free5gc-amf.yaml
$ kubectl apply -f test-infra/e2e/tests/free5gc/006-regional-free5gc-smf.yaml
``` 

This take a bit more time, in `regional` check `amf`. In step 6, SMF in `regional` wil connect to UPF in `edge1` and `edge02`. Therefore, check SMF's log in `regional` by:

```bash
##### -----=[ In regional cluster ]=----- ####
$ kubectl logs -n free5gc-cp -l name=smf-regional
...
2024-04-28T19:26:17Z [INFO][SMF][App] Received PFCP Association Setup Accepted Response from UPF[172.1.0.254]
2024-04-28T19:26:17Z [INFO][SMF][App] Sending PFCP Association Request to UPF[172.1.2.254]
2024-04-28T19:26:17Z [INFO][LIB][PFCP] Remove Request Transaction [2]
2024-04-28T19:26:17Z [INFO][SMF][App] Received PFCP Association Setup Accepted Response from UPF[172.1.2.254]
```
If both UPFs were successfully connected, this means that the N4 connection was successful.



<br></br>
---
|Before|  |  |  |  |  |  |Next|
|--|--|--|--|--|--|--|--|
|[ Go to Before Page](5_deploy_free5gc_cp.md) |  |  |  |  |  |  | [ Go to Next Page ](7_deploy_ueransim.md)|