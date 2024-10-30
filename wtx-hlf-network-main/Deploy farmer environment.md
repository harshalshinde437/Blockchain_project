Deploy farmer environment

	1. Deploy farmer ICA
		-	helm install ica-farmer -n farmer helm-charts/fabric-ca -f examples/fabric-ca/ica-farmer.yaml
		-	helm install ica-buyer -n buyer helm-charts/fabric-ca -f examples/fabric-ca/ica-buyer.yaml
		-	helm install ica-shipper -n shipper helm-charts/fabric-ca -f examples/fabric-ca/ica-shipper.yaml
	
	2. Create identities for ICA -farmer
		-	helm install farmer-ops -n farmer helm-charts/fabric-ops/ -f examples/fabric-ops/farmer/identities.yaml
		-	helm install buyer-ops -n buyer helm-charts/fabric-ops/ -f examples/fabric-ops/buyer/identities.yaml
		-	helm install shipper-ops -n shipper helm-charts/fabric-ops/ -f examples/fabric-ops/shipper/identities.yaml
	
	3.	Add farmer Org to the network
		For Adding farmer org to the network, Comment the receiver section from the Values.organizatons array in the values file examples/fabric-ops/adminrorg/configure-org-channel.yaml for now and then will do vice versa for receiver org.
		-	helm  install  configorgchannel  -n  adminorg  helm-charts/fabric-ops/  -f  examples/fabric-ops/adminorg/configure-org-channel.yaml
	
	4.	Deploy Peers on farmer
		-	helm install peer -n farmer helm-charts/fabric-peer/ -f examples/fabric-peer/farmer/values.yaml
		-	helm install peer -n buyer helm-charts/fabric-peer/ -f examples/fabric-peer/buyer/values.yaml
		-	helm install peer -n shipper helm-charts/fabric-peer/ -f examples/fabric-peer/shipper/values.yaml
	
	5.	Install ChainCode on farmer Peers.
		-	helm install installchaincode -n farmer helm-charts/fabric-ops/ -f examples/fabric-ops/farmer/install-chaincode.yaml

	6.	Update Anchor peers of farmer
		-	helm install updateanchorpeer -n farmer helm-charts/fabric-ops/ -f examples/fabric-ops/farmer/update-anchor-peer.yaml

	7.	Approve chaincode defination
		- helm install approvechaincode -n farmer helm-charts/fabric-ops/ -f examples/fabric-ops/farmer/approve-chaincode.yaml

	8.	Commit Chaincode
		- helm  install  commitchaincode  -n  adminorg  helm-charts/fabric-ops/  -f  examples/fabric-ops/adminorg/commit-chaincode.yaml

	10.  **Deploy the Chaincode as external Service**

		Every peer requires its dedicated chaincode for execution. If your organization has, for instance, three peers, it is necessary to perform the installation three times—once for each peer. 
		```
		helm  install  {{peerName}}  -n  chaincode  helm-charts/fabric-cc  -f  examples/fabric-cc/values.yaml
		Ex. PeerName: peer0-adminorg
		```