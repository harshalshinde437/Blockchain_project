
# WorldTradeX-WTX : HLF Deployment - Minikube


### Reference Deployment Strategies


#### Prerequisite
 
[![Minikube Version](https://img.shields.io/badge/minikube-v1.32.0+-blue)]() 
[![kubectl Version](https://img.shields.io/badge/kubectl-v1.29.2+-blue)]() 
[![Helm Version](https://img.shields.io/badge/helm_version-v3.14.2+-blue)]() 
[![Nginx Version](https://img.shields.io/badge/nginx_ingress_version-v1.9.0+-blue)]()

##### Keep in mind the following things throughout this example deployment;

  <!-- 1. We'll be using the domain `wtx-dev.net` for external connect ( under Research & Development).  -->
  1. We'll be using the domain `localho.st` for external connect ( under Research & Development). 
  2. Nginx ingress services are exposed via aws Loadbalancer.
  <!-- 4. StorageClass we use `ebs-sc` as default from aws.  -->
  3. StorageClass we use `standard` as default from aws. 
  4. project : `wtx`. 
  5.  Cluster internal  communication we will be using xxx.namespace.svc 

  # Let's get started.

1.  **Start Minikube with docker driver on local system**
	
  ```minikube start --driver=docker ```
	
2.  **Enable the nginx ingress** 
 
 ``` minikube addons enable ingress ```

*NOTE: Check services 80 and 443 are running properly*

3.  **Setup core DNS with hostname and ClusterIP**

- ``` CLUSTER_IP=$(kubectl -n ingress-nginx get svc ingress-nginx-controller -o json | jq -r .spec.clusterIP)```
- ``` echo $CLUSTER_IP ``` 
- Update coreDNS by adding below code in coredns_update.yaml
```
kind: ConfigMap
apiVersion: v1
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        rewrite name regex (.*)\.localho.st host.ingress.internal
        hosts {
          ${CLUSTER_IP} host.ingress.internal
          fallthrough
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
```
-   ``` kubectl apply -f coredns_update.yaml```

5. **Enable the prometheys metrics**

```kubectl  create  -f  https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/bundle.yaml```

6. **Deploy a Filestore server.**

This is where the helper uploads some of the common artifacts like blockfiles, chain code package, collection config etc. Its basically an Nginx deployment with custom nginx rules to support file uploading over curl. 
 
 ```helm install filestore -n filestore helm-charts/filestore/ -f examples/filestore/values.yaml --create-namespace```

Access filestore servers as below within the pods.
 
 `curl http://filestore.filestore.svc/wtx`

7. **Create Kuberenetes seceret**

Create a kubernetes secret with ```user``` and ```password``` as keys for CA's . Secret's are kept out of the helm charts/values for beter security. 
All CA/ICA/TLSCA server username/password are handled in this way. Change the ```namespace``` & ```secret``` as below.
```
kubectl  create  ns  orderer
kubectl  -n  orderer  create  secret  generic  rca-secret  --from-literal=user=rca-admin  --from-literal=password=rcaworldtradexp@SS
kubectl  -n  orderer  create  secret  generic  tlsca-secret  --from-literal=user=tls-admin  --from-literal=password=tlsworldtradexp@SS
kubectl  -n  orderer  create  secret  generic  orderer-secret  --from-literal=user=ica-orderer  --from-literal=password=icaordererSamplePassword

kubectl  create  ns adminorg
kubectl  -n  adminorg  create  secret  generic  adminorg-secret  --from-literal=user=ica-adminorg  --from-literal=password=icaadminorgSamplePassword

kubectl  create  ns farmer
kubectl  -n  farmer  create  secret  generic  farmer-secret  --from-literal=user=ica-farmer  --from-literal=password=icafarmerSamplePassword

kubectl  create  ns buyer
kubectl  -n  buyer  create  secret  generic  buyer-secret  --from-literal=user=ica-buyer  --from-literal=password=icabuyerSamplePassword

kubectl  create  ns shipper
kubectl  -n  shipper  create  secret  generic  shipper-secret  --from-literal=user=ica-shipper  --from-literal=password=icashipperSamplePassword
```

### Deploy ROOTCA,TLSCA  Environments

8. **Deploy ROOT CA**
 ```
helm install root-ca -n orderer helm-charts/fabric-ca -f examples/fabric-ca/root-ca.yaml
```
This will deploy the `root-ca` server for you and the server will be available at `https://root-ca.orderer.svc:7051`. 

To verify the server, you can get into any running pod in the cluster and send a curl request as below;
```
curl https://root-ca.orderer.svc:7051/cainfo --insecure
```
9. **Deploy TLSCA**

```
helm install tls-ca -n orderer helm-charts/fabric-ca -f examples/fabric-ca/tls-ca.yaml
```
This will deploy the `tls-ca` server for you and the server will be available at `https://tls-ca.orderer.svc:7051`. 
You can verify it with the similar way we verified the root-ca end-point above.
```
curl https://tls-ca.orderer.svc:7051/cainfo --insecure
```

10. ### Create Identities
* Note:- Every identity registration job must be executed in the same namespace where the respective CA's are running. And the admin credentials secret name must be supplied to the values file at `Values.ca_secret` if need to change.

**For  ROOT CA**
```
helm install rootca-ops -n orderer helm-charts/fabric-ops/ -f examples/fabric-ops/rootca/rootca-identities.yaml
```  
  **For  TLS CA**
  
```
helm install tlsca-ops -n orderer helm-charts/fabric-ops/ -f examples/fabric-ops/tlsca/tlsca-identities.yaml
```
11.  **Deploy Orderer ICA**
* Now deploy the  ORDER CA  Fabric-ca charts,  but this will be in ICA mode. 
```
helm install ica-orderer -n orderer helm-charts/fabric-ca -f examples/fabric-ca/ica-orderer.yaml
```
12. **Deploy Adminorg ICA**
```
helm install ica-adminorg -n adminorg helm-charts/fabric-ca -f examples/fabric-ca/ica-adminorg.yaml 
```
13. **Create Identities for ICA Orderer & Adminorg**

* **Create identities for ICA-orderer**
```
helm install orderer-ops -n orderer helm-charts/fabric-ops/ -f examples/fabric-ops/orderer/orderer-identities.yaml
```

* **Create identities for ICA-adminorg**   

```
helm install adminorg-ops -n adminorg helm-charts/fabric-ops/ -f examples/fabric-ops/adminorg/identities.yaml
```
14. **Generate Genesisblock & Channel transaction**
```
helm install cryptogen -n orderer helm-charts/fabric-ops/ -f examples/fabric-ops/orderer/orderer-cryptogen.yaml
```
After successful completion of this `cryptogen Job`, you'll see the `Genesisblock` file and `Channel transaction` file in your filestore under your project directory. If your project name is `yourproject`, then your project directory will be created as `/usr/share/nginx/html/wtx`.

### Deploy Orderers and Admin Peers

15. **Deploy Orderers**
```
helm install orderer -n orderer helm-charts/fabric-orderer/ -f examples/fabric-orderer/orderer.yaml
```
16.  **Deploy Peers on Adminorg**
```
helm install peer -n adminorg helm-charts/fabric-peer/ -f  examples/fabric-peer/adminorg/values.yaml
```
After successful deployment of the  Orders and Peers, you will get 5 Orderers and 2 peers in  each namespace. 
Each Peers will have 1 Init container and 2 app containers `(Fabric Peer & CouchDB)`.
Couchdb passwds can be encrypted and store in kube secrets.

17.  **Create Channel**
```
helm  install  channelcreate  -n  adminorg  helm-charts/fabric-ops/  -f  examples/fabric-ops/adminorg/channel-create.yaml
```

18.  **Update Anchor peers for adminorg** 
```
helm  install  updateanchorpeer  -n  adminorg  helm-charts/fabric-ops/  -f  examples/fabric-ops/adminorg/update-anchor-peer.yaml
```
19.  **Package and Deploy Chaincode As External  Service**

* **Important** :- Before you install chaincode , you need to package the chaincode for `CCAAS`  and upload the packaged chaincode file `cc-p2p-asset-ext.tgz` to the filestore under `wtx` directory. 
  * Use following command to copy chaincode file `cc-p2p-asset-ext.tgz` to `wtx` directory

  ```
  kubectl cp examples/chaincode/chaincode-external/cc-p2p-asset-ext.tgz filestore/<filestore-pod_name>:/usr/share/nginx/html/wtx
  ```
  * This collection-config.json required for approvechaincode on org
  ```
  kubectl cp examples/files/collection-config.json filestore/<filestore-pod_name>:/usr/share/nginx/html/wtx
  ```

**Packaging chaincode (optional for this setup)**
```
tar cfz code.tar.gz connection.json
tar cfz cc-p2p-asset-ext.tgz metadata.json code.tar.gz

peer lifecycle chaincode calculatepackageid cc-p2p-asset-ext.tgz
p2p-cc:52d58cb7e17422210455fefd0291205ee2a3b1daaab785f255174c0d0de71ea2
```
This will generate the chaincode packageid , Make a note of it which will use for approvals.

20. **Install chaincode  package on adminorg Peers**
```
helm  install  installchaincode  -n  adminorg  helm-charts/fabric-ops/  -f  examples/fabric-ops/adminorg/install-chaincode.yaml
```
21.  **Deploy the Chaincode as external Service**

Every peer requires its dedicated chaincode for execution. If your organization has, for instance, three peers, it is necessary to perform the installation three times—once for each peer. 
```
helm  install  {{peerName}}  -n  chaincode  helm-charts/fabric-cc  -f  examples/fabric-cc/values.yaml
Ex. PeerName: peer0-adminorg
```

### **Deploy Farmer environment**

22.  **Deploy Farmer ICA**
```
helm install ica-farmer -n farmer helm-charts/fabric-ca -f examples/fabric-ca/ica-farmer.yaml
```
23.  **Create identities for ICA -Farmer**
```
helm install farmer-ops -n farmer helm-charts/fabric-ops/ -f examples/fabric-ops/farmer/identities.yaml
```
24.  **Add Farmer Org to the network**

For  Adding  `Farmer`  org to the network, Comment the `buyer` section from the `Values.organizatons` array in the values file [examples/fabric-ops/adminrorg/configure-org-channel.yaml](./fabric-ops/adminorg/configure-org-channel.yaml) for now and then will do vice versa for `buyer` org.
```
helm  install  configorgchannel  -n  adminorg  helm-charts/fabric-ops/  -f  examples/fabric-ops/adminorg/configure-org-channel.yaml
```
25.  **Deploy Peers on Farmer**
```
helm install peer -n farmer helm-charts/fabric-peer/ -f examples/fabric-peer/farmer/values.yaml
```

26.  **Install ChainCode on farmer Peers.**
```
helm install installchaincode -n farmer helm-charts/fabric-ops/ -f examples/fabric-ops/farmer/install-chaincode.yaml
```

27.  **Update Anchor peers of farmer**
```
helm install updateanchorpeer -n farmer helm-charts/fabric-ops/ -f examples/fabric-ops/farmer/update-anchor-peer.yaml
```
28.    **Deploy the Chaincode as external Service**

To activate the chaincode for farmer peers, follow the steps outlined in **Step 21**, substituting the respective peer name that you intend to launch.


### **Deploy Buyer environment**

29.  **Deploy Buyer ICA**
``` 
helm install ica-buyer -n buyer helm-charts/fabric-ca -f examples/fabric-ca/ica-buyer.yaml
```
30.  **Create identities for ICA-buyer.**
```
helm install buyer-ops -n buyer helm-charts/fabric-ops/ -f examples/fabric-ops/buyer/identities.yaml
```
31.  **Add Buyer org to the network**

For  Adding  `Buyer`  org to the network, Comment  the `farmer` section from the `Values.organizatons` array in the values file [examples/fabric-ops/adminrorg/configure-org-channel.yaml](./fabric-ops/adminorg/configure-org-channel.yaml) and uncomment the `Buyer` section now.
```
helm  upgrade  configorgchannel  -n  adminorg  helm-charts/fabric-ops/  -f  examples/fabric-ops/adminorg/configure-org-channel.yaml
```
32.  **Deploy peers on Buyer**
```
helm install peer -n buyer helm-charts/fabric-peer/ -f examples/fabric-peer/buyer/values.yaml
```
33.  **Install ChainCode on Buyer Peers**
```
helm install installchaincode -n buyer helm-charts/fabric-ops/ -f examples/fabric-ops/buyer/install-chaincode.yaml
```

34.  **Update Anchor peers of Buyer**
```
helm install updateanchorpeer -n buyer helm-charts/fabric-ops/ -f examples/fabric-ops/buyer/update-anchor-peer.yaml
```

35.    **Deploy the Chaincode as external Service**

To activate the chaincode for farmer peers, follow the steps outlined in **Step 21**, substituting the respective peer name that you intend to launch.


### **Deploy Shipper environment**

36.  **Deploy Shipper ICA**
``` 
helm install ica-shipper -n shipper helm-charts/fabric-ca -f examples/fabric-ca/ica-shipper.yaml
```
37.  **Create identities for ICA-shipper.**
```
helm install shipper-ops -n shipper helm-charts/fabric-ops/ -f examples/fabric-ops/shipper/identities.yaml
```
38.  **Add Shipper org to the network**

For  Adding  `Shipper`  org to the network, Comment  the `farmer` section from the `Values.organizatons` array in the values file [examples/fabric-ops/adminrorg/configure-org-channel.yaml](./fabric-ops/adminorg/configure-org-channel.yaml) and uncomment the `Shipper` section now.
```
helm  upgrade  configorgchannel  -n  adminorg  helm-charts/fabric-ops/  -f  examples/fabric-ops/adminorg/configure-org-channel.yaml
```
39.  **Deploy peers on Shipper**
```
helm install peer -n shipper helm-charts/fabric-peer/ -f examples/fabric-peer/shipper/values.yaml
```
40.  **Install ChainCode on Shipper Peers**
```
helm install installchaincode -n shipper helm-charts/fabric-ops/ -f examples/fabric-ops/shipper/install-chaincode.yaml
```

41.  **Update Anchor peers of Shipper**
```
helm install updateanchorpeer -n shipper helm-charts/fabric-ops/ -f examples/fabric-ops/shipper/update-anchor-peer.yaml
```

42.    **Deploy the Chaincode as external Service**

To activate the chaincode for farmer peers, follow the steps outlined in **Step 21**, substituting the respective peer name that you intend to launch.


### Approve & Commit Chain Code

Since Collection-config required for privacy,  ensure the **`require_collection_config: "true"`** in the value.yaml file.Also need to upload a collection config file to the filestore under your `cvs` project directory. 

43. **Approve ChainCode on Adminorg**

Now you have to update the Chaincode package ID in [examples/fabric-ops/adminorg/approve-chaincode.yaml](./fabric-ops/adminorg/approve-chaincode.yaml) which you collected at step 21. 
Below are the required fields for updating with your own chaincode details. 
- cc_name
- cc_version
- cc_package_id
- seq
  
**Note:**  These feilds are important for subsequent changes or upgrades here after.
```
helm  install  approvechaincode  -n  adminorg  helm-charts/fabric-ops/  -f  examples/fabric-ops/adminorg/approve-chaincode.yaml
```
45.  **Approve ChainCode on Farmer**


 Ensure that you have updated the Chaincode package ID  and other variables  in [examples/fabric-ops/farmer/approve-chaincode.yaml](./fabric-ops/farmer/approve-chaincode.yaml), 
```
helm install approvechaincode -n farmer helm-charts/fabric-ops/ -f examples/fabric-ops/farmer/approve-chaincode.yaml
```
46. **Approve ChainCode on Buyer**

 Ensure that you have updated the Chaincode package ID  and other variables  in [examples/fabric-ops/buyer/approve-chaincode.yaml](./fabric-ops/buyer/approve-chaincode.yaml), 
```
helm install approvechaincode -n buyer helm-charts/fabric-ops/ -f examples/fabric-ops/buyer/approve-chaincode.yaml
```
47. **Approve ChainCode on Shipper**

 Ensure that you have updated the Chaincode package ID  and other variables  in [examples/fabric-ops/shipper/approve-chaincode.yaml](./fabric-ops/shipper/approve-chaincode.yaml), 
```
helm install approvechaincode -n shipper helm-charts/fabric-ops/ -f examples/fabric-ops/shipper/approve-chaincode.yaml
```

48. **Commit ChainCode from the Adminorg**

 Ensure that you have updated the Chaincode package ID and other variables  in [examples/fabric-ops/adminorgorg/commit-chaincode.yaml](./fabric-ops/adminorgorg/commit-chaincode.yaml), below are the required fields for updating chaincode details. 
- cc_name
- cc_version
- seq
```
helm  install  commitchaincode  -n  adminorg  helm-charts/fabric-ops/  -f  examples/fabric-ops/adminorg/commit-chaincode.yaml
```
## Appendix 


### Deploy fabric-cli tools. (Optional)

Optionally deploy a `fabric-cli tool` which has the essential fabric tools for advanced troubleshooting. We have packed it as a helm chart which will do dynamic enrollment for your identities. You don't need seperate values file for this, simply use the existing identity registration value file. The CLI pod  have to deploy per organisation/namespace.
```
helm  install  cli-tools  -n  adminorg  helm-charts/fabric-tools/  -f  examples/fabric-ops/adminorg/identities.yaml
```
 Once deployed, exec into the pod and run `bash /scripts/enroll.sh` and watch the output. All identities will be enrolled and new certs will be available in the respective directory. 
 

 #### Sample Query executed with asset-transfer CC
 ```
 #Smaple query
bash-5.1# export CC_NAME=p2p-cc
bash-5.1# export CHANNEL_NAME=wtxchannel
bash-5.1# export ORDERER_NAME=orderer0-orderer.orderer.svc
bash-5.1# export CORE_PEER_CHAINCODEADDRESS=peer0-adminorg-fabric-cc:7052
bash-5.1#
bash-5.1#
bash-5.1# peer chaincode invoke -o $ORDERER_NAME:30000 --ordererTLSHostnameOverride $ORDERER_NAME --tls --cafile $ORDERER_CA -C $CHANNEL_NAME -n $CC_NAME --peerAddresses $CORE_PEER_ADDRESS --tlsRootCertFiles $CORE_PEER_TLS_ROOTCERT_FILE  -c '{"Args":["InitLedger"]}'
2024-01-10 07:13:20.436 UTC 0001 INFO [chaincodeCmd] chaincodeInvokeOrQuery -> Chaincode invoke successful. result: status:200
bash-5.1#
bash-5.1#
bash-5.1# peer chaincode invoke -o $ORDERER_NAME:30000 --ordererTLSHostnameOverride $ORDERER_NAME --tls --cafile $ORDERER_CA -C $CHANNEL_NAME -n $CC_NAME --peerAddresses $CORE_PEER_ADDRESS --tlsRootCertFiles $CORE_PEER_TLS_ROOTCERT_FILE  -c '{"Args":["ReadAsset","asset1"]}'
2024-01-10 07:17:37.499 UTC 0001 INFO [chaincodeCmd] chaincodeInvokeOrQuery -> Chaincode invoke successful. result: status:200 payload:"{\"ID\":\"asset1\",\"color\":\"blue\",\"size\":5,\"owner\":\"Tomoko\",\"appraisedValue\":300}"
```

### Trying for different peer  with sample query.

```
####  Cannot execute from other peer from same org, since no CC for that container. Ex. Below.
bash-5.1# CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/users/peer1-adminorg/tls/server.key
bash-5.1# CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/users/peer1-adminorg/tls/ca.crt
bash-5.1# CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/users/peer1-adminorg/tls/server.crt
bash-5.1# CORE_PEER_ADDRESS=peer1-adminorg:30000
bash-5.1# peer chaincode invoke -o $ORDERER_NAME:30000 --ordererTLSHostnameOverride $ORDERER_NAME --tls --cafile $ORDERER_CA -C $CHANNEL_NAME -n $CC_NAME --peerAddresses $CORE_PEER_ADDRESS --tlsRootCertFiles $CORE_PEER_TLS_ROOTCERT_FILE  -c '{"Args":["ReadAsset","asset1"]}'
Error: endorsement failure during invoke. response: status:500 message:"error in simulation: failed to execute transaction d6a686cd357ca057531f14dab98cf709cbd3821c15541df13d360b55222d9cb9: could not launch chaincode p2p-cc:c9fa0502e64ae4d020e5a99aaea405c2f4a3379567f16d7f28c591e120ac26a9: connection to p2p-cc:c9fa0502e64ae4d020e5a99aaea405c2f4a3379567f16d7f28c591e120ac26a9 failed: error cannot create connection for p2p-cc:c9fa0502e64ae4d020e5a99aaea405c2f4a3379567f16d7f28c591e120ac26a9: error creating grpc connection to peer1-adminorg-fabric-cc.chaincode.svc:9999: failed to create new connection: connection error: desc = \"transport: error while dialing: dial tcp: lookup peer1-adminorg-fabric-cc.chaincode.svc on 172.20.0.10:53: no such host\""

```

## Deploy fabric Explorer.

### Create a Database credentials 
```
kubectl  create  ns hlexplorer
kubectl -n hlexplorer create secret generic pgs-secret --from-literal=user=hppoc --from-literal=password=PGpassword
```

### Deploy the Explorer
```
helm install fabexplorer -n hlexplorer helm-charts/fabric-explore/ -f  examples/fabric-explore/explorer.yaml
```

### Access Explorer using this command (for minikube)
```
minikube service fabexplorer-fabric-explore -n hlexplorer
```