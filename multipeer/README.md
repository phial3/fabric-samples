# Fabric2.4 多机部署

## 1. 编写生成证书文件

### (1)生成证书文件的配置文件：crypto-config.yaml

当我们在./multipeer文件夹下面，我们可以选择touch一个`crypto-config.yaml`，但是其实官方可以给我们模板文件，将模板文件复制过来进行修改即可:

复制模板文件命令：

`cryptogen showtemplate > crypto-config.yaml`

如果这里出现了问题：cryptogen工具不存在啥的，就要配置一下外部的环境变量：

`export PATH=${PWD}/../bin:$PATH`

### (2)进入文件中进行修改:

`vim crypto-config.yaml`

### (3)通过配置文件生成证书文件

`cryptogen generate --config=crypto-config.yaml`

> 注：生成文件可以复制到另外一个虚拟机的同等位置中，也可以通过scp命令将证书文件复制到其他虚拟机中。


## 2. 编写生成通道文件
### (1)生成通道文件的配置文件：`configtx.yaml`

当我们在./multipeer文件夹下面,新建`configtx.yaml`，可以参考官方示例的该文件

### (2)生成创世块文件和通道文件
下面这四句话分别创建:`genesis.block`文件 `channel.tx`文件，以及两个锚节点`Org1MSPanchors.tx` 和`Org2MSPanchors.tx`，这些文件全部都保存在`channel-artifacts`文件夹中

```bash
configtxgen -profile TwoOrgsOrdererGenesis -channelID fabric-channel -outputBlock ./channel-artifacts/genesis.block
 
# 生成用于配置自定义 channel 的交易文件, 名称为:mychannel
export CHANNEL_NAME=mychannel

configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID $CHANNEL_NAME
 
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org1MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org1MSP
 
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org2MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org2MSP
```

> 注：将`channel-artifacts`文件夹复制到另外一虚拟机上，这样两台机子都有相同的创始块文件和通道文件了。

## 3.编写docker-compose文件

该文件是用来启动docker容器。

针对不同的节点，我们在`./multipeer`文件夹下面，需要编写三个不同的docker-compose配置文件

分别是`orderer.yaml` ，`peer.yaml`

> 注：注意：这里的extra_hosts中IP地址需要修改，一定要改成你的网络环境，其他地方可以不用动！！！！！！

![参考地址](https://studygolang.com/articles/26415?fr=sidebar)
## 4. 部署网络
### (1)使用 docker-compose 启动所有的节点环境
```bash
docker-compose -f orderer.yaml up -d
docker-compose -f org1.yaml up -d
```
### (2)使用 fabric-tools 工具创建 channel
```bash
docker exec -it peer0-org1-cli /bin/bash
```
```bash
export CHANNEL_NAME=mychannel

export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt

peer channel create \
    -o orderer.example.com:7050 \
    -c $CHANNEL_NAME \
    -f ./channel-artifacts/channel.tx \
    --tls \
    --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

### (3)逐个将 peer 节点加入到组织中
```bash
# 将 peer0.org1 加入到 channel 中
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt

peer channel join -b mychannel.block

# 将 peer1.org1 加入到 channel 中
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer1.org1.example.com:8051
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer1.org1.example.com/tls/ca.crt

peer channel join -b mychannel.block

# 将 peer0.org2 加入到 channel 中
export  CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp 
export CORE_PEER_ADDRESS=peer0.org2.example.com:9051 
export CORE_PEER_LOCALMSPID="Org2MSP" 
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt 

peer channel join -b mychannel.block

# 将 peer1.org2 加入到 channel 中
export  CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp 
export CORE_PEER_ADDRESS=peer1.org2.example.com:10051 
export CORE_PEER_LOCALMSPID="Org2MSP" 
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer1.org2.example.com/tls/ca.crt 

peer channel join -b mychannel.block
```

### (4)更新组织的 anchor peer
```bash
# 更新 org1 的 anchor peer
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt

peer channel update \
    -o orderer.example.com:7050 \
    -c $CHANNEL_NAME \
    -f ./channel-artifacts/Org1MSPanchors.tx \
    --tls \
    --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

# 更新 org2 的 anchor peer
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp 
export CORE_PEER_ADDRESS=peer0.org2.example.com:9051 
export CORE_PEER_LOCALMSPID="Org2MSP" 
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt 

peer channel update \
    -o orderer.example.com:7050 \
    -c $CHANNEL_NAME \
    -f ./channel-artifacts/Org2MSPanchors.tx \
    --tls \
    --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

## 5.安装 chaincode
### (1)打包 chaincode
```bash
peer lifecycle chaincode package mychaincode.tar.gz \
--path github.com/hyperledger/fabric-samples/chaincode/abstore/go/ \
--lang golang \
--label mycc_1
```
### (2)安装 chaincode
```bash
# 在 peer0.org1 上安装 chaincode
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
peer lifecycle chaincode install mychaincode.tar.gz

# 查看是否安装成功
peer lifecycle chaincode queryinstalled

# 在 peer0.org2 上安装 chaincode
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=peer0.org2.example.com:9051
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
peer lifecycle chaincode install mychaincode.tar.gz

# 查看是否安装成功
peer lifecycle chaincode queryinstalled
```
### (3)组织对新安装的 chaincode 进行投票
```bash
# 在 queryinstalled 时会输出 chaincode 的 package-id，将此 id 替换
export CC_PACKAGE_ID=mycc_1:4622fb602aa60c6368716a70474dc9d9ba2776200f70eccca07a4df4360eaff3

# org1 投票
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt

peer lifecycle chaincode approveformyorg \
--channelID $CHANNEL_NAME \
--name mycc \
--version 1 \
--init-required \
--package-id $CC_PACKAGE_ID \
--sequence 1 \
--tls true \
--cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

# 查看投票情况
peer lifecycle chaincode checkcommitreadiness \
--channelID $CHANNEL_NAME \
--name mycc \
--version 1 \
--init-required \
--sequence 1 \
--tls true \
--cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem \
--output json

# org2 投票
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=peer0.org2.example.com:9051
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt

peer lifecycle chaincode approveformyorg \
--channelID $CHANNEL_NAME \
--name mycc \
--version 1 \
--init-required \
--package-id $CC_PACKAGE_ID \
--sequence 1 \
--tls true \
--cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

# 查看投票情况
peer lifecycle chaincode checkcommitreadiness \
--channelID $CHANNEL_NAME \
--name mycc \
--version 1 \
--init-required \
--sequence 1 \
--tls true \
--cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem \
--output json
```

### (4) submit
```bash
peer lifecycle chaincode commit \
-o orderer.example.com:7050 \
--channelID $CHANNEL_NAME \
--name mycc \
--version 1 \
--sequence 1 \
--init-required \
--tls true \
--cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem \
--peerAddresses peer0.org1.example.com:7051 \
--tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt \
--peerAddresses peer0.org2.example.com:9051 \
--tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt

# 在 peer0.org1 检查是否 commit 成功
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
peer lifecycle chaincode querycommitted \
--channelID mychannel \
--name mycc

# 在 peer0.org2 检查是否 commit 成功
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=peer0.org2.example.com:9051
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
peer lifecycle chaincode querycommitted \
--channelID mychannel \
--name mycc
```

## 6.测试 chaincode
### (1)Invoke
```bash
peer chaincode invoke \
-o orderer.example.com:7050 \
--tls true \
--cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem \
-C mychannel \
-n mycc \
--peerAddresses peer0.org1.example.com:7051 \
--tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt \
--peerAddresses peer0.org2.example.com:9051 \
--tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt \
--isInit \
-c '{"Args":["Init","a","100","b","100"]}'
```

### (2)Query
```bash
peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
```
