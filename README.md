launch minikube on local machine

setup an NFS server on local machine. On my machine, the server is at _/nfs/fabric_

in k8s folder:

```bash
# if on mac, use the -mac version
kubectl apply -f fabric-pv.yaml
# if on mac, use the -mac version
kubectl apply -f fabric-pvc.yaml

kubectl apply -f fabric-tools.yaml

# wait that the fabric-tools pod is created

kubectl exec -it fabric-tools -- mkdir /fabric/config

kubectl cp ../config/configtx.yaml fabric-tools:/fabric/config/
kubectl cp ../config/crypto-config-orderer.yaml fabric-tools:/fabric/config/
kubectl cp ../config/crypto-config-org1.yaml fabric-tools:/fabric/config/
kubectl cp ../config/crypto-config-org2.yaml fabric-tools:/fabric/config/

```

in the fabric-tools pod (enter it with `kubectl exec -it fabric-tools -- /bin/bash`)

```bash
cryptogen generate --config /fabric/config/crypto-config-orderer.yaml
cryptogen generate --config /fabric/config/crypto-config-org1.yaml
cryptogen generate --config /fabric/config/crypto-config-org2.yaml
cp -r crypto-config /fabric/
cp /fabric/config/configtx.yaml /fabric/
cd /fabric/
configtxgen -profile TwoOrgsOrdererGenesis -channelID system-channel -outputBlock genesis.block

# create anchor peer update tx's
export FABRIC_CFG_PATH=/fabric
export CHANNEL_NAME=mychannel
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./Org1MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org1MSP
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./Org2MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org2MSP

```

in k8s folder:

```bash
kubectl apply -f orderer-deploy.yaml
kubectl apply -f orderer-svc.yaml
kubectl apply -f peer0-org1-deploy.yaml
kubectl apply -f peer0-org1-svc.yaml
kubectl apply -f peer0-org2-deploy.yaml
kubectl apply -f peer0-org2-svc.yaml

# https://github.com/kubernetes/minikube/issues/1568#issuecomment-308674617
minikube ssh
(in minikube) $ sudo ip link set docker0 promisc on
```

in the fabric-tools pod (enter it with `kubectl exec -it fabric-tools -- /bin/bash`)

```bash
#
# Create channel creation tx
#

export FABRIC_CFG_PATH=/fabric
export CHANNEL_NAME=mychannel
cd /fabric
configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ${CHANNEL_NAME}.tx -channelID ${CHANNEL_NAME}

chmod a+rx /fabric/* -R

#
# setup env variables
#

export CORE_PEER_TLS_ENABLED=true
export DIR_CRYPTO_MATERIAL="/fabric/crypto-config"
export CHANNEL_NAME=mychannel

export ORDERER_CA=$DIR_CRYPTO_MATERIAL/ordererOrganizations/example-com/orderers/orderer-example-com/msp/tlscacerts/tlsca.example-com-cert.pem
export PEER0_ORG1_CA=$DIR_CRYPTO_MATERIAL/peerOrganizations/org1-example-com/peers/peer0-org1-example-com/tls/ca.crt
export PEER0_ORG2_CA=$DIR_CRYPTO_MATERIAL/peerOrganizations/org2-example-com/peers/peer0-org2-example-com/tls/ca.crt

export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_MSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER0_ORG1_CA
export CORE_PEER_TLS_CERT_FILE=$DIR_CRYPTO_MATERIAL/peerOrganizations/org1-example-com/peers/peer0-org1-example-com/tls/server.crt
export CORE_PEER_TLS_KEY_FILE=$DIR_CRYPTO_MATERIAL/peerOrganizations/org1-example-com/peers/peer0-org1-example-com/tls/server.key
export CORE_PEER_MSPCONFIGPATH=$DIR_CRYPTO_MATERIAL/peerOrganizations/org1-example-com/users/Admin@org1-example-com/msp
#export CORE_PEER_MSPCONFIGPATH=$DIR_CRYPTO_MATERIAL/peerOrganizations/org1-example-com/peers/peer0-org1-example-com/msp/
export CORE_PEER_ADDRESS=peer0-org1-example-com:7051

export FABRIC_CFG_PATH="/etc/hyperledger/fabric"

export CORE_PEER_ADDRESSAUTODETECT="true"

#
# Create channel
#

cd /fabric

peer channel create -o orderer-example-com:7050 -c $CHANNEL_NAME -f /fabric/${CHANNEL_NAME}.tx --tls $CORE_PEER_TLS_ENABLED --cafile $ORDERER_CA

#
# Join channel
#

cd /fabric

peer channel join -b $CHANNEL_NAME.block


```
