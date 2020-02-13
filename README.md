#Testing Hyperledger Fabric 2.0.0 on Kubernetes

launch minikube on local machine

setup an NFS server on local machine. On my machine, the server is at _/nfs/fabric_
```bash
sudo apt install nfs-kernel-server nfs-common cifs-utils
sudo mkdir -p /nfs/fabric
sudo chown -R nobody:nogroup /nfs
sudo chmod -R 777 /nfs

sudo nano /etc/exports
# add (the ip is the one you get from minikube ip)
/nfs/fabric 10.0.2.15(rw,sync,no_subtree_check)

sudo exportfs -a
sudo service nfs-kernel-server restart
sudo service nfs-kernel-server status
```

in k8s folder:

```bash
kubectl apply -f fabric-pv.yaml
kubectl apply -f fabric-pvc.yaml
kubectl apply -f fabric-tools.yaml

# wait that the fabric-tools pod is created

kubectl exec -it fabric-tools -- mkdir /fabric/config
kubectl exec -it fabric-tools -- mkdir /go/src/fabcar

kubectl cp ../config/configtx.yaml fabric-tools:/fabric/config/
kubectl cp ../config/crypto-config-orderer.yaml fabric-tools:/fabric/config/
kubectl cp ../config/crypto-config-org1.yaml fabric-tools:/fabric/config/
kubectl cp ../config/crypto-config-org2.yaml fabric-tools:/fabric/config/
kubectl cp ../chaincode/fabcar fabric-tools:/go/src/
kubectl cp ../chaincode/marbles fabric-tools:/go/src/

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

export ORDERER_CA=$DIR_CRYPTO_MATERIAL/ordererOrganizations/example-com/orderers/orderer-example-com/msp/tlscacerts/tlsca-example-com-cert.pem
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

#
# Package & Install chaincode
#
peer lifecycle chaincode package fabcar-new-lifecycle.pak --label fabcar --path fabcar --lang golang
peer lifecycle chaincode install fabcar-new-lifecycle.pak
peer lifecycle chaincode queryinstalled
peer lifecycle chaincode getinstalledpackage --package-id
peer lifecycle chaincode approveformyorg  -o orderer-example-com:7050 --tls --cafile $ORDERER_CA --channelID mychannel --name fabcar --version 1.0 --init-required --package-id fabcar:3f9decd6c10775c19cc25ffd48f767d3aae2f3ea21013f5faea7475e13fd5448 --sequence 1 --signature-policy "OR ('Org1MSP.peer','Org2MSP.peer')"
peer lifecycle chaincode checkcommitreadiness -o orderer-example-com:7050 --channelID mychannel --tls --cafile $ORDERER_CA --name fabcar --version 1.0 --init-required --sequence 1 --output json
peer lifecycle chaincode commit -o orderer-example-com:7050 --channelID mychannel --name fabcar --version 1.0 --sequence 1 --init-required --tls --cafile $ORDERER_CA

peer lifecycle chaincode package marbles.tar.gz --label marbles --path marbles --lang golang
peer lifecycle chaincode install marbles.tar.gz
peer lifecycle chaincode queryinstalled
peer lifecycle chaincode approveformyorg  -o orderer-example-com:7050 --tls --cafile $ORDERER_CA --channelID mychannel --name marbles --version 1 --package-id marbles:d29fb491edce809fb531aa7ed933bf3995bb8a863ed7f0ac65a17990757152e8 --sequence 1 --signature-policy "OR ('Org1MSP.peer','Org2MSP.peer')"
peer lifecycle chaincode checkcommitreadiness -o orderer-example-com:7050 --channelID mychannel --tls --cafile $ORDERER_CA --name marbles --version 1 --sequence 1 --signature-policy "OR ('Org1MSP.peer','Org2MSP.peer')" --output json 
peer lifecycle chaincode commit -o orderer-example-com:7050 --channelID mychannel --name marbles --version 1 --sequence 1 --signature-policy "OR ('Org1MSP.peer','Org2MSP.peer')" --tls --cafile $ORDERER_CA --peerAddresses peer0-org1-example-com:7051 --peerAddresses peer0-org2-example-com:9051 --tlsRootCertFiles $PEER0_ORG1_CA --tlsRootCertFiles $PEER0_ORG2_CA
peer lifecycle chaincode querycommitted --channelID mychannel

peer chaincode invoke -o orderer-example-com:7050 --tls --cafile $ORDERER_CA -C mychannel -n marbles -c '{"Args":["initMarble","marble1","blue","35","tom"]}' --waitForEvent









# ORG2

export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_MSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER0_ORG2_CA
export CORE_PEER_TLS_CERT_FILE=$DIR_CRYPTO_MATERIAL/peerOrganizations/org2-example-com/peers/peer0-org2-example-com/tls/server.crt
export CORE_PEER_TLS_KEY_FILE=$DIR_CRYPTO_MATERIAL/peerOrganizations/org2-example-com/peers/peer0-org2-example-com/tls/server.key
export CORE_PEER_MSPCONFIGPATH=$DIR_CRYPTO_MATERIAL/peerOrganizations/org2-example-com/users/Admin@org2-example-com/msp
#export CORE_PEER_MSPCONFIGPATH=$DIR_CRYPTO_MATERIAL/peerOrganizations/org2-example-com/peers/peer0-org2-example-com/msp/
export CORE_PEER_ADDRESS=peer0-org2-example-com:9051

```
