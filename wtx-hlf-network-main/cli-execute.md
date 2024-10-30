peer chaincode invoke -o $ORDERER_NAME:30000 --ordererTLSHostnameOverride $ORDERER_NAME --tls --cafile $ORDERER_CA -C $CHANNEL_NAME -n $CC_NAME --peerAddresses $CORE_PEER_ADDRESS --tlsRootCertFiles $CORE_PEER_TLS_ROOTCERT_FILE  -c '{"Args":["InitLedger"]}'

export CORE_PEER_CHAINCODEADDRESS=peer0-adminorg-fabric-cc:7052
export ORDERER_NAME=orderer0-orderer.orderer.svc
export ORDERER_NAME=ica-orderer.orderer.svc
export CORE_PEER_ADDRESS=peer1-adminorg.localho.st
export CC_NAME=p2p-cc
export CHANNEL_NAME=wtxchannel

peer chaincode invoke -o $ORDERER_NAME:30000 --ordererTLSHostnameOverride $ORDERER_NAME --tls --cafile $ORDERER_CA -C $CHANNEL_NAME -n $CC_NAME --peerAddresses peer0-adminorg:30000 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/users/peer0-adminorg/tls/ca.crt --peerAddresses peer1-adminorg:30000 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/users/peer1-adminorg/tls/ca.crt  -c '{"Args":["InitLedger"]}'

peer1-farmer.localho.st


peer chaincode invoke -o orderer.localho.st --ordererTLSHostnameOverride orderer.localho.st --tls --cafile $ORDERER_CA -C $CHANNEL_NAME -n $CC_NAME --peerAddresses peer0-adminorg.localho.st:30000 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/users/peer0-adminorg/tls/ca.crt --peerAddresses peer1-adminorg.localho.st:30000 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/users/peer1-adminorg/tls/ca.crt  -c '{"Args":["InitLedger"]}'


peer chaincode invoke --isInit --tls true --cafile /organizations/ordererOrganizations/localho.st/orderers/orderer.localho.st/msp/tlscacerts/tlsca.localho.st-cert.pem -C cctest2 -n nslcc07 --peerAddresses peer1-org1:7051 --tlsRootCertFiles /organizations/peerOrganizations/org1.localho.st/peers/peer1.org1.localho.st/tls/ca.crt --peerAddresses peer0-org2:7051 --tlsRootCertFiles /organizations/peerOrganizations/org2.localho.st/peers/peer0.org2.localho.st/tls/ca.crt -c '{"Args":["initLedger"]}' --waitForEvent

osnadmin channel list -o orderer:9443 --ca-file /organizations/ordererOrganizations/localho.st/orderers/orderer.localho.st/msp/tlscacerts/tlsca.localho.st-cert.pem --client-cert /organizations/ordererOrganizations/localho.st/msp/signcerts/cert.pem --client-key /organizations/ordererOrganizations/localho.st/msp/keystore/2fb02d6909d272dd08a7970deb61dafe1fb5a59dea13ede765afedceba279708_sk --no-status


peer chaincode invoke --tls true -o $ORDERER_NAME:30000 --ordererTLSHostnameOverride $ORDERER_NAME --tls --cafile $ORDERER_CA -C wtxchannel -n p2p-cc --peerAddresses $CORE_PEER_ADDRESS --tlsRootCertFiles $CORE_PEER_TLS_ROOTCERT_FILE --peerAddresses peer1-adminorg:30000 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/users/peer1-adminorg/tls/ca.crt  -c '{"Args":["InitLedger"]}'









rm ../code.tar.gz cc-p2p-asset-ext.tgz
export CHAINCODE_NAME=p2p-cc
export CHAINCODE_LABEL=p2p-cc
cat << METADATA-EOF > "metadata.json"
{
    "type": "ccaas",
    "label": "${CHAINCODE_LABEL}"
}
METADATA-EOF

cat > "connection.json" <<CONN_EOF
{
  "address": "${CHAINCODE_NAME}:7052",
  "dial_timeout": "10s",
  "tls_required": false
}
CONN_EOF