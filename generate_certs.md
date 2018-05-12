## 身份信息

权限管理是Fabric很重要的一个功能, 有权限管理就要有身份的区别,   身份是根据组织, 成员类型或具体某个成员来区分, 底层是通过数字证书. MSP 数字证书是Fabric 网络实体的身份标识, 实体在通信和交易时使用证书进行签名和认证.

### 工具ctyptogen

Fabric 提供了一个工具-cryptogen 它会根据配置文件生成组织身份信息.
```shell
cryptogen generate --config=./crypto-config.yaml
```
byfn脚本对应的输出
```
/Users/haibbo/code/fabric-samples/first-network/../bin/cryptogen
 
##########################################################
##### Generate certificates using cryptogen tool #########
##########################################################
org1.example.com
org2.example.com
```

### 目录结构

创建的证书信息放在了crypto-config目录, 目录大概的结构如下:
```
~/c/f/first-network git:release ❯❯❯ tree crypto-config -L 3
crypto-config
├── ordererOrganizations
│   └── example.com
│       ├── ca
│       ├── msp
│       ├── orderers
│       ├── tlsca
│       └── users
└── peerOrganizations
    ├── org1.example.com
    │   ├── ca
    │   ├── msp
    │   ├── peers
    │   ├── tlsca
    │   └── users
    └── org2.example.com
        ├── ca
        ├── msp
        ├── peers
        ├── tlsca
        └── users

20 directories, 0 files
```

crypto-config目录会有两个目录，ordererOrganizations和peerOrganizations，它们分别代表排序服务节点和Peer节点的MSP配置信息.
peerOrganizations目录下有两个组织 org1.example.com 和 org2.example.com，子目录结构同 ordererOrganizations 的 example.com.

目录 | 说明
------------ | -------------
ca | 根证书和密钥, pem为证书, sk为密钥
msp | 这个目录会共享给ordder或peer节点, 之后详细介绍
orderers或peers | 代表节点的配置
tlsca | tls ca的证书和密钥
user | 用户信息, 一定有一个管理员, 用户个数可指定

下面看下具体peer节点下都有什么:
```
~/c/f/f/c/p/o/p/peer0.org1.example.com git:release ❯❯❯ tree
.
├── msp
│   ├── admincerts
│   │   └── Admin@org1.example.com-cert.pem
│   ├── cacerts
│   │   └── ca.org1.example.com-cert.pem
│   ├── keystore
│   │   └── 49093a3739e911a5210ec54dd2663534600872228e74b55002f5e40bae9f941c_sk
│   ├── signcerts
│   │   └── peer0.org1.example.com-cert.pem
│   └── tlscacerts
│       └── tlsca.org1.example.com-cert.pem
└── tls
    ├── ca.crt
    ├── server.crt
    └── server.key

7 directories, 8 files

```
MSP 目录结构如下:

目录 | 说明
------------ | -------------
admingcerts | MSP的管理员证书
cacerts | MSP的根证书
keystore| 签名的密钥
signcerts| 签名用的证书
tlscacerts| TLS的根证书

tls目录用来进行TLS连接, 保证链路层的安全, 结构如下 :

目录 | 说明
------------ | -------------
ca.crt | root CA 证书
cacerts | 用来进行TLS连接的证书
keystore| 用来TLS连接的密钥

如果仔细看crypto-config就会发现, 里面很多证书是重复的.应该是为了部署方便. 用下面的命令可以看到哪些证书是冗余的, 冗余文件的可能名字相同,也可能不相同.
```shell
find crypto-config -type f | xargs md5 | sort -k 4
```

### 配置文件

cryptogen使用的配置文件,crypto-config的内容如下:

```
~/c/f/first-network git:release ❯❯❯ cat crypto-config.yaml | grep -v "^ *#" | grep -v "^$"
OrdererOrgs:
  - Name: Orderer
    Domain: example.com
    Specs:
      - Hostname: orderer
PeerOrgs:
  - Name: Org1
    Domain: org1.example.com
    Template:
      Count: 2
    Users:
      Count: 1
  - Name: Org2
    Domain: org2.example.com
    Template:
      Count: 2
    Users:
      Count: 1
```
- Domain 是这个节点的全称
- Template 直接节点的个数, 隐式的hostname是peer[0-max]
- Users 指定普通用户个数, 管理员用户是默认存在的
- 用户和 peer 没有对应关系

### 证书信息

解析在org1.example.com里的user1的证书
```
~/c/f/f/c/p/o/u/U/m/signcerts git:release ❯❯❯ openssl x509  -in  User1@org1.example.com-cert.pem -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            04:17:f6:90:07:bd:ba:af:d2:a5:ae:51:41:b3:42:e0
    Signature Algorithm: **ecdsa-with-SHA256**
        Issuer: C=US, ST=California, L=San Francisco, O=org1.example.com, **CN=ca.org1.example.com**
        Validity
            Not Before: Mar 19 07:29:17 2018 GMT
            Not After : Mar 16 07:29:17 2028 GMT
        Subject: C=US, ST=California, L=San Francisco, **CN=User1@org1.example.com**
        ..............
        X509v3 extensions:
            X509v3 Key Usage: critical
                **Digital Signature**
```
- 签名为ECDSA(不支持RSA和SM2), Hash算法为SHA256
- ca.org1.example.com签发
- 身份是User1@org1.example.com
- Key Usage: Digital Signature

你也可以查看签发用户的root ca,它是自签的.
```
/c/f/first-network git:release ❯❯❯ cd ./crypto config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/msp/cacerts
~/c/f/f/c/p/o/p/p/m/cacerts git:release ❯❯❯ openssl x509  -in  ca.org1.example.com-cert.pem -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            66:d5:e3:7f:a2:fe:a2:d7:16:bf:e4:37:89:2f:1c:b9
    Signature Algorithm: **ecdsa-with-SHA256**
        Issuer: C=US, ST=California, L=San Francisco, **O=org1.example.com, CN=ca.org1.example.com**
        Validity
            Not Before: Mar 19 07:29:17 2018 GMT
            Not After : Mar 16 07:29:17 2028 GMT
        Subject: C=US, ST=California, L=San Francisco, **O=org1.example.com, CN=ca.org1.example.com**
        Subject Public Key Info:
          ....................
        X509v3 extensions:
            X509v3 Key Usage: critical
                **Digital Signature, Key Encipherment, Certificate Sign, CRL Sign**
            X509v3 Extended Key Usage:
                Any Extended Key Usage
```

### 这些证书已经生成, 会如何使用呢?
- copy到peer特定的目录里
  ```shell
       - ../crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com /msp:/etc/hyperledger/fabric/msp
       - ../crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com /tls:/etc/hyperledger/fabric/tls
       # base/docker-compose-base.yaml中 把msp和tls 映射到peer0的特定目录里面
  ```

- 客户端(SDK)需要用户证书, 去操作fabric网络

  ```shell
  - ./crypto-config:/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/
  # docker-compose-cli.yaml cli.volumes中 映射了整个crypto-config目录创世block
  ```

- 创世block

