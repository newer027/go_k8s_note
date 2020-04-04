# 企业级区块链hyperledger fabric的完整实施

- 前提：准备 master，10.10.70.137(peer0)，10.10.121.27(peer)三台 ubuntu 16.04 虚拟。

0.在三台虚机中安装1.13版本的 go 环境，设置环境变量
```bash
export GOROOT=/usr/local/go
export GOPATH=$HOME/Projects/golang
export PATH=$GOPATH/bin:$GOROOT/bin:$PATH
```


1.在 master 创建工程的目录，设置环境变量。
```bash
cd /home/ubuntu/hlf-deployment
export PATH=${PWD}/bin:${PWD}:$PATH
export FABRIC_CFG_PATH=${PWD}/fabric-config
```

2.在 master 生成加密产物。
```bash
cryptogen generate --config=./fabric-config/crypto-config.yaml
```

3.在 master 生成创世区块。
```bash
configtxgen -profile OneOrgsOrdererGenesis -outputBlock ./network-config/orderer.block
```

4.在 master 创建设置单个组织的通道的产物。
```bash
configtxgen -profile OneOrgsChannel -outputCreateChannelTx ./network-config/channel.tx -channelID mychannel
```

5.在 master 创建设置组织的MSP的产物。
```bash
configtxgen -profile OneOrgsChannel -outputAnchorPeersUpdate ./network-config/Org1MSPanchors.tx -channelID mychannel -asOrg Org1MSP
```

6.在 master 设置 ca yaml 中秘钥的文件名。
```bash
ls crypto-config/peerOrganizations/org1.example.com/ca #获取CA文件名 
vim deployment/docker-compose-kafka.yml  #修改CA文件名，FABRIC_CA_SERVER_CA_KEYFILE 80eb9dfa677b46db171517410afbed2cea3bf7b2dff136000a27b0ca468dee83_sk
```

7.在 master 启动 kafka 和 zookeeper 的 docker 容器。
```bash
docker-compose -f deployment/docker-compose-kafka.yml up -d
#传递文件夹crypto-config 到peer0, peer1
```

8.在 peer0: 10.10.70.137 中启动 peer0 和 cli0 容器。
```bash
docker-compose -f deployment/docker-compose-peer0.yml up -d
docker-compose -f deployment/docker-compose-cli0.yml up -d
```

9.在 peer1: 10.10.121.27 中启动 peer1 和 cli1 容器。
```bash
docker-compose -f deployment/docker-compose-peer1.yml up -d
docker-compose -f deployment/docker-compose-cli1.yml up -d
```

10.在 peer0: 10.10.70.137 设置通道，加入通道。
```bash
docker exec -e "CORE_PEER_LOCALMSPID=Org1MSP" -e "CORE_PEER_MSPCONFIGPATH=/var/hyperledger/users/Admin@org1.example.com/msp" peer0.org1.example.com peer channel create -o orderer0.example.com:7050 -c mychannel -f /var/hyperledger/configs/channel.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

# docker exec cli peer channel create -o orderer0.example.com:7050 -c mychannel -f /var/hyperledger/configs/channel.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

docker exec -e "CORE_PEER_LOCALMSPID=Org1MSP" -e "CORE_PEER_MSPCONFIGPATH=/var/hyperledger/users/Admin@org1.example.com/msp" peer0.org1.example.com peer channel join -b mychannel.block --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

# docker-compose -f deployment1911/docker-compose-cli0.yml exec cli peer channel join -b mychannel.block
```

11.在 peer0: 10.10.70.137 从 docker 容器内部获取包含通道信息的数据产物。
```bash
docker cp peer0.org1.example.com:/opt/gopath/src/github.com/hyperledger/fabric/peer/mychannel.block .
# docker cp cli:/opt/gopath/src/github.com/hyperledger/fabric/peer/mychannel.block .
scp -r mychannel.block ubuntu@10.10.121.27:/home/ubuntu/deployment
```

12.在 peer1: 10.10.121.27 中导入数据产物，加入前期设定的通道。
```bash
docker cp mychannel.block peer1.org1.example.com:/opt/gopath/src/github.com/hyperledger/fabric/peer/mychannel.block
# docker cp mychannel.block cli:/opt/gopath/src/github.com/hyperledger/fabric/peer/mychannel.block
rm mychannel.block

docker exec -e "CORE_PEER_LOCALMSPID=Org1MSP" -e "CORE_PEER_MSPCONFIGPATH=/var/hyperledger/users/Admin@org1.example.com/msp" peer1.org1.example.com peer channel join -b mychannel.block
# docker-compose -f deployment/docker-compose-cli1.yml exec cli peer channel join -b mychannel.block

# docker-compose -f deployment/docker-compose-cli1.yml exec cli peer channel update -o orderer0.example.com:7050 -c mychannel -f /var/hyperledger/configs/Org1MSPanchors.tx  --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

13.在 peer0: 10.10.70.137 安装链代码。
```bash
docker exec -it cli peer chaincode install -n marble -p github.com/chaincode -v 1.0
```

14.在 peer1: 10.10.121.27 安装链代码。
```bash
docker exec -it cli peer chaincode install -n marble -p github.com/chaincode -v 1.0
```

15.在 peer0: 10.10.70.137 启动链代码。
```bash
docker exec -it cli peer chaincode instantiate -o orderer0.example.com:7050 -C mychannel -n marble github.com/chaincode -v 1.0 -c '{"Args":["init"]}' --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```


16.在 peer0: 10.10.70.137 测试调用链代码。
```bash
docker exec -it cli peer chaincode invoke -o orderer0.example.com:7050 -n marble -c '{"Args":["initStringSha", "2040"]}' -C mychannel --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer0.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```


