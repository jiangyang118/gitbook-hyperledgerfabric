# 1、环境
阿里云centos7.4，安装过程可以参考[centos7安装fabric](https://www.jianshu.com/p/cb032c42c909)
```
Docker version 18.09.0, build 4d60db4
docker-compose version 1.23.2, build 1110ad0
go version go1.11.4 linux/amd64
gcc version 4.8.5 20150623 (Red Hat 4.8.5-36) (GCC)
node -v v9.9.0
npm -v 6.5.0
```
# 2、步骤
整体步骤为：
 1. 初始化环境：创建证书并创建创世区块以及通道连接证书文件；
 2. 启动网络；
 3. 客户端安装链码，实例化链码；
 4. 客户端查询，转账操作。

 下面步骤均为一步步操作，如果想查看自动化脚本，请查看[centos7.4运行hyperLedger fabric 1.3.0 first network](https://blog.csdn.net/destiny_aqua/article/details/86577705)
## 2.1、初始化环境
初始化环境包含如下几个部分：
1、下载可执行文件（用于生成证书、创世区块、拉取镜像等）；
2、创建证书；
3、创建创世区块与区块交易文件。
首先在fabric目录下新建fabric-test目录并切换到fabric目录下。
```shell
#后续步骤均在目录/home/yigou/fabric-1.3.0下执行
cd /home/yigou
mkdir fabric-1.3.0
cd fabric-1.3.0
```
### 2.1.1、 下载可执行文件
```shell
#下载生成证书的可执行文件，启动1.3.0表示下载1.3.0版本，可以在fabric目录下通过git branch查看有多少版本
curl -sSL https://goo.gl/6wtTNS | bash -s 1.3.0
```
如果无法访问外网可以直接通过[这里下载](https://download.csdn.net/download/destiny_aqua/10922560)并上传到服务器，上传后如下图所示。
但是建议通过如下命令下载，不过速度比较慢（阿里云服务器大约15分钟）。因为每个服务器的环境不同，他所需的可执行文件也不同。
```bash
wget https://raw.githubusercontent.com/hyperledger/fabric/master/scripts/bootstrap.sh
chmod +x bootstrap.sh
#需要指定版本，否则下载最新版本
./bootstrap.sh 1.3.0
#下载完成后可以看到fabric-sample文件夹
[root@bc1 fabric-samples]# cd bin/
[root@bc1 bin]# ll
total 149656
-rwxrwxr-x 1 1001 1001 18551624 Oct 10 22:40 configtxgen
-rwxrwxr-x 1 1001 1001 19693584 Oct 10 22:41 configtxlator
-rwxrwxr-x 1 1001 1001 11909544 Oct 10 22:40 cryptogen
-rwxrwxr-x 1 1001 1001 19532432 Oct 10 22:40 discover
-rwxrwxr-x 1 1001 1001 17101576 Jan 10 02:04 fabric-ca-client
-rwxrwxr-x 1 1001 1001      810 Oct 10 22:41 get-docker-images.sh
-rwxrwxr-x 1 1001 1001 10724192 Oct 10 22:40 idemixgen
-rwxrwxr-x 1 1001 1001 23890528 Oct 10 22:41 orderer
-rwxrwxr-x 1 1001 1001 31812608 Oct 10 22:41 peer
[root@bc1 bin]# pwd
/home/yigou/fabric-1.3.0/fabric-samples/bin
```
### 2.1.2、 创建证书
创建证书所需要的文件crypto-config.yaml，1.3.0版本之前（包括1.3.0）来自fabric目前下的example目录下的[e2e_cli示例](https://github.com/hyperledger/fabric/tree/release-1.3/examples/e2e_cli)；1.4.0版本来自fabric-sample目录下的[first-network](https://github.com/hyperledger/fabric-samples/tree/release-1.4/first-network)
```shell
#1、当前目录是fabric-samples，进入第一个测试fabric网络
cd first-network/
#2、使用bin/cryptogen工具创建证书文件，生成如下两个组织（显示域名）
#生成org1和org2两个组织：org1.example.com和org2.example.com，放入crypto-config文件夹内。
../bin/cryptogen generate --config=./crypto-config.yaml 
#下面是执行结果
org1.example.com
org2.example.com
[root@bc1 first-network]# cd crypto-config
[root@bc1 crypto-config]# ll
total 8
drwxr-xr-x 3 root root 4096 Jan 21 21:49 ordererOrganizations
drwxr-xr-x 4 root root 4096 Jan 21 21:49 peerOrganizations
```
### 2.1.3、创建创世区块
```shell
#1 设置环境变量并创建channel-artifacts文件夹，目的是将创世区块放入channel-artifacts文件夹内
export FABRIC_CFG_PATH=$PWD
echo $PWD
mkdir channel-artifacts

#2 创建创世区块，参考https://hyperledger-fabric.readthedocs.io/en/release-1.4/commands/configtxgen.html?highlight=configtxgen 该说明讲解查看创世区块中的内容
../bin/configtxgen -profile TwoOrgsOrdererGenesis -outputBlock ./channel-artifacts/genesis.block
#下面是执行结果
2019-01-22 15:27:04.712 CST [common/tools/configtxgen] main -> WARN 001 Omitting the channel ID for configtxgen for output operations is deprecated.  Explicitly passing the channel ID will be required in the future, defaulting to 'testchainid'.
2019-01-22 15:27:04.712 CST [common/tools/configtxgen] main -> INFO 002 Loading configuration
2019-01-22 15:27:04.761 CST [common/tools/configtxgen] doOutputBlock -> INFO 003 Generating genesis block
2019-01-22 15:27:04.761 CST [common/tools/configtxgen] doOutputBlock -> INFO 004 Writing genesis block

#3 设置通道的交易文件
export CHANNEL_NAME=mychannel  && ../bin/configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID $CHANNEL_NAME
#下面是执行结果
2019-01-22 15:28:11.618 CST [common/tools/configtxgen] main -> INFO 001 Loading configuration
2019-01-22 15:28:11.640 CST [common/tools/configtxgen] doOutputChannelCreateTx -> INFO 002 Generating new channel configtx
2019-01-22 15:28:11.642 CST [common/tools/configtxgen] doOutputChannelCreateTx -> INFO 003 Writing new channel tx

#4 生成Org1MSP和Org2MSP的anchor peer（在org1组织下的节点提交交易信息的背书环节使用）
../bin/configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org1MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org1MSP
../bin/configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org2MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org2MSP
#下面是执行结果
2019-01-22 15:32:18.127 CST [common/tools/configtxgen] main -> INFO 001 Loading configuration
2019-01-22 15:32:18.149 CST [common/tools/configtxgen] doOutputAnchorPeersUpdate -> INFO 002 Generating anchor peer update
2019-01-22 15:32:18.149 CST [common/tools/configtxgen] doOutputAnchorPeersUpdate -> INFO 003 Writing anchor peer update
2019-01-22 15:32:35.167 CST [common/tools/configtxgen] main -> INFO 001 Loading configuration
2019-01-22 15:32:35.189 CST [common/tools/configtxgen] doOutputAnchorPeersUpdate -> INFO 002 Generating anchor peer update
2019-01-22 15:32:35.189 CST [common/tools/configtxgen] doOutputAnchorPeersUpdate -> INFO 003 Writing anchor peer update
```
##  2.2、启动网络
fabric网络包含1个orderer和4个peer以及1个client。使用docker-compose运行docker节点。
```shell
#up-启动；down-停止；-d参数表示后台启动，去掉-d参数可以查看日志
docker-compose -f docker-compose-cli.yaml up -d
#查看启动的peer节点和orderer节点的docker，以及相关的ca、couchdb的docker（本示例没有ca和couchdb）
docker ps -a
#下面是执行结果
CONTAINER ID        IMAGE                               COMMAND             CREATED             STATUS              PORTS                                              NAMES
138cc70a4bba        hyperledger/fabric-tools:latest     "/bin/bash"         9 seconds ago       Up 8 seconds                                                           cli
35311089f614        hyperledger/fabric-peer:latest      "peer node start"   10 seconds ago      Up 9 seconds        0.0.0.0:8051->7051/tcp, 0.0.0.0:8053->7053/tcp     peer1.org1.example.com
cdc8f0dedd98        hyperledger/fabric-peer:latest      "peer node start"   10 seconds ago      Up 9 seconds        0.0.0.0:7051->7051/tcp, 0.0.0.0:7053->7053/tcp     peer0.org1.example.com
6a933335588d        hyperledger/fabric-peer:latest      "peer node start"   10 seconds ago      Up 9 seconds        0.0.0.0:9051->7051/tcp, 0.0.0.0:9053->7053/tcp     peer0.org2.example.com
8302425b7379        hyperledger/fabric-orderer:latest   "orderer"           10 seconds ago      Up 9 seconds        0.0.0.0:7050->7050/tcp                             orderer.example.com
3a9cc84b8ecc        hyperledger/fabric-peer:latest      "peer node start"   10 seconds ago      Up 9 seconds        0.0.0.0:10051->7051/tcp, 0.0.0.0:10053->7053/tcp   peer1.org2.example.com
```
##  2.3、客户端安装链码，实例化链码
运行如下代码，启动客户端安装链码，实例化链码。
```bash
docker exec -it cli bash
#下面是执行结果
root@138cc70a4bba:/opt/gopath/src/github.com/hyperledger/fabric/peer#

#1 创建channel channel.tx是2.1小节中生成的交易证书，cli客户端通过装载channel-artifacts文件夹获得
# ca-cert是验证TLS握手协议必须的证书文件，使用前需要设置CHANNEL_NAME
export CHANNEL_NAME=mychannel
peer channel create -o orderer.example.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/channel.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
#下面是执行结果
2019-01-22 07:41:30.487 UTC [main] InitCmd -> WARN 001 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-01-22 07:41:30.566 UTC [main] SetOrdererEnv -> WARN 002 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-01-22 07:41:30.570 UTC [channelCmd] InitCmdFactory -> INFO 003 Endorser and orderer connections initialized
2019-01-22 07:41:30.600 UTC [cli.common] readBlock -> INFO 004 Received block: 0

#2 将peer加入channel中
peer channel join -b mychannel.block
#下面是执行结果
2019-01-22 07:43:24.007 UTC [channelCmd] InitCmdFactory -> INFO 003 Endorser and orderer connections initialized
2019-01-22 07:43:24.049 UTC [channelCmd] executeJoin -> INFO 004 Successfully submitted proposal to join channel

#3 peer0.org2加入channel中
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp CORE_PEER_ADDRESS=peer0.org2.example.com:7051 CORE_PEER_LOCALMSPID="Org2MSP" CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt peer channel join -b mychannel.block
#下面是执行结果
2019-01-22 07:45:13.129 UTC [channelCmd] InitCmdFactory -> INFO 003 Endorser and orderer connections initialized
2019-01-22 07:45:13.170 UTC [channelCmd] executeJoin -> INFO 004 Successfully submitted proposal to join channel

#4 加入anchor peer
peer channel update -o orderer.example.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/Org1MSPanchors.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
#下面是执行结果
2019-01-22 07:45:55.005 UTC [channelCmd] InitCmdFactory -> INFO 003 Endorser and orderer connections initialized
2019-01-22 07:45:55.013 UTC [channelCmd] update -> INFO 004 Successfully submitted channel update

#5 peer0.org2加入anchor peer
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp CORE_PEER_ADDRESS=peer0.org2.example.com:7051 CORE_PEER_LOCALMSPID="Org2MSP" CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt peer channel update -o orderer.example.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/Org2MSPanchors.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
#下面是执行结果
2019-01-22 07:46:55.829 UTC [channelCmd] InitCmdFactory -> INFO 003 Endorser and orderer connections initialized
2019-01-22 07:46:55.838 UTC [channelCmd] update -> INFO 004 Successfully submitted channel update

#6 安装chaincode -P "AND ('Org1MSP.peer','Org2MSP.peer')"背书策略意味着需要两个节点一起才能背书
peer chaincode install -n mycc -v 1.0 -p github.com/chaincode/chaincode_example02/go/
#下面是执行结果
2019-01-22 07:47:47.851 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 003 Using default escc
2019-01-22 07:47:47.851 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 004 Using default vscc
2019-01-22 07:47:50.488 UTC [chaincodeCmd] install -> INFO 005 Installed remotely response:<status:200 payload:"OK" > 

#7 peer0.org2安装chaincode
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp CORE_PEER_ADDRESS=peer0.org2.example.com:7051 CORE_PEER_LOCALMSPID="Org2MSP" CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt peer chaincode install -n mycc -v 1.0 -p github.com/chaincode/chaincode_example02/go/

2019-01-22 07:55:38.252 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 003 Using default escc
2019-01-22 07:55:38.252 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 004 Using default vscc
2019-01-22 07:55:38.491 UTC [chaincodeCmd] install -> INFO 005 Installed remotely response:<status:200 payload:"OK" > 

#8 实例化chaincode
peer chaincode instantiate -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C $CHANNEL_NAME -n mycc -v 1.0 -c '{"Args":["init","a", "100", "b","200"]}' -P "AND ('Org1MSP.peer','Org2MSP.peer')"
#下面是执行结果
2019-01-22 07:49:10.480 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 003 Using default escc
2019-01-22 07:49:10.480 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 004 Using default vscc
```
##  2.4、客户端查询，转账操作
继续在客户端中运行。
```shell
#1 查询a/b分别持有的金额
peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","b"]}'
#下面是执行结果
root@138cc70a4bba:/opt/gopath/src/github.com/hyperledger/fabric/peer# peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
100
root@138cc70a4bba:/opt/gopath/src/github.com/hyperledger/fabric/peer# peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","b"]}'
200

#2 a向b转账10元
peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C $CHANNEL_NAME -n mycc --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt -c '{"Args":["invoke","a","b","10"]}'
#下面是执行结果
2019-01-22 07:56:06.330 UTC [chaincodeCmd] chaincodeInvokeOrQuery -> INFO 003 Chaincode invoke successful. result: status:200 

#3 查询a/b分别持有的金额
peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","b"]}'
#下面是执行结果
90
210
```
