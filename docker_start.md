## 启动Docker

### Docker Compose
Compose 是 Docker 官方的开源项目，负责实现对 Docker 容器集群的快速编排. 在fabric网络中需要多个节点协同才能完成区块链应用, 它允许用户通过一个单独的 docker-compose.yaml 模板文件来定义一组相关联的应用容器为一个项目.

### Docker-compose-cli.yaml
打开脚本bybn.sh, 会看到如下的启动命令:
```
docker-compose -f docker-compose-cli.yaml up -d
```
打开docker-compose-cli.yaml:
```shell
.........
services: 
  orderer.example.com:  # 服务名称
    extends:
      file:   base/docker-compose-base.yaml # include别处的配置文件
      service: orderer.example.com
    container_name: orderer.example.com # container名词, 有些命令需要使用到它
........
```
很多细节需要去看base/docker-compose-base.yaml.

### Service Orderer
```shell
..........
services:
  orderer.example.com:
    container_name: orderer.example.com
    image: hyperledger/fabric-orderer:$IMAGE_TAG #使用的docker image
    environment:
      - ORDERER_GENERAL_LOGLEVEL=debug   ## 输出日志级别
      - ORDERER_GENERAL_LISTENADDRESS=0.0.0.0 ## 服务监听的地址
      - ORDERER_GENERAL_GENESISMETHOD=file ## 创世区块提供的方式
      - ORDERER_GENERAL_GENESISFILE=/var/hyperledger/orderer/orderer.genesis.block ## 创世区块的路径
      - ORDERER_GENERAL_LOCALMSPID=OrdererMSP ## MSP的ID
      - ORDERER_GENERAL_LOCALMSPDIR=/var/hyperledger/orderer/msp ## MSP文件路径
      - ORDERER_GENERAL_TLS_ENABLED=false ## 是否开启TLS, false的情况下面三个可以不用设置.
      - ORDERER_GENERAL_TLS_PRIVATEKEY=/var/hyperledger/orderer/tls/server.key ## 签名私钥的位置
      - ORDERER_GENERAL_TLS_CERTIFICATE=/var/hyperledger/orderer/tls/server.crt ## 指定身份证书位置
      - ORDERER_GENERAL_TLS_ROOTCAS=[/var/hyperledger/orderer/tls/ca.crt] ## 指定信任的root CA的证书位置
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric ## 工作目录
    command: orderer ## 节点执行的命令
    volumes: ## 路径挂载, 可以方便的配置节点, 或保留节点存储的信息
      - ../channel-artifacts/genesis.block:/var/hyperledger/orderer/
          orderer.genesis.block ## 把生成的创世区块映射到一具体的目录里
      - ../crypto-config/ordererOrganizations/example.com/orderers/
          orderer.example.com/msp:/var/hyperledger/orderer/msp ## 传入orderer的MSP的文件
      - ../crypto-config/ordererOrganizations/example.com/orderers/        orderer.example.com/tls/:/var/hyperledger/orderer/tls ## 传入TLS证书
      - orderer.example.com:/var/hyperledger/production/orderer ## 接受节点生成的block等信息, 但是这边的写法有问题, 需要更改.
    ports: ##端口映射
      - 7050:7050  ## 把本地的7050 端口映射到orderer节点的7050端口.
```
### Service peer
```shell
peer0.org1.example.com:
    container_name: peer0.org1.example.com
    extends:
      file: peer-base.yaml  # 引用别处的配置, 和orderer类似就不做介绍了
      service: peer-base
    environment:
      - ORE_VM_ENDPOINT=unix:///host/var/run/docker.sock ## Docker的服务地址
      - CORE_PEER_ID=peer0.org1.example.com # Peer ID, 必须唯一
      - CORE_PEER_ADDRESS=peer0.org1.example.com:7051 # 服务地址
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org1.example.com:7051 ## 对组织外发包的地址
      - CORE_PEER_LOCALMSPID=Org1MSP # 所受组织MSP的ID
    volumes:
        - /var/run/:/host/var/run/ ## docker
        - ../crypto-config/peerOrganizations/org1.example.com/peers/ peer0.org1.example.com/msp:/etc/hyperledger/fabric/msp ## 传入生成的MSP 证书
        - ../crypto-config/peerOrganizations/org1.example.com/peers/ peer0.org1.example.com/tls:/etc/hyperledger/fabric/tls ## 传入生成的TLS证书
        - peer0.org1.example.com:/var/hyperledger/production # 接受节点生成的block等信息, 但是这边的写法有问题, 需要更改.
    ports:
      - 7051:7051
      - 7053:7053
```
### 客户端节点
在BYFN中, cli节点是一个客户端, 它负责了之后的所有的操作, 具体的命令在脚本 script/script.sh中.  

之后的测试中, 我会用我的主机代替cli节点把操作分开来执行(需设置好环境变量和hosts文件).



