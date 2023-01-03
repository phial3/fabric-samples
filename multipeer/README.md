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
下面这四句话分别创建:`genesis.block`文件 `channel.tx`文件 以及两个锚节点`Org1MSPanchors.tx` 和`Org2MSPanchors.tx`，这些文件全部都保存在`channel-artifacts`文件夹中

```bash
configtxgen -profile TwoOrgsOrdererGenesis -channelID fabric-channel -outputBlock ./channel-artifacts/genesis.block
 
configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID mychannel
 
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org1MSPanchors.tx -channelID mychannel -asOrg Org1MSP
 
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org2MSPanchors.tx -channelID mychannel -asOrg Org2MSP
```

> 注：将`channel-artifacts`文件夹复制到另外一虚拟机上，这样两台机子都有相同的创始块文件和通道文件了。

## 3.编写docker-compose文件

该文件是用来启动docker容器。

针对不同的节点，我们在`./multipeer`文件夹下面，需要编写三个不同的docker-compose配置文件

分别是`orderer.yaml` ，`org1.yaml` ，`org2.yaml`

> 注：注意：这里的extra_hosts中IP地址需要修改，一定要改成你的网络环境，其他地方可以不用动！！！！！！

