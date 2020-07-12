HyperLedger Multi-Host 구성하기
===============================
   
* 개요

* 구성 목표
   * node별 container 구성
   * node 정보  
   
* 인프라 준비작업
  * host name 설정
  * docker swarm 설정
  * overlay network 설정
  
* artifacts,MSP(channel-artifacts, crypto-config) 준비작업

* git repogitory clone 
   * Oderer yaml 파일 설정
   * peer yaml 파일 
   * yaml 파일 deploy
   
* create channel, join channel, update anchor peer

* chain code 
   * package 하기
   * 설치
   * 설치 및 package id 확인
   * 설치 승인
   * 승인 상태 확인
   * commit 하기

* utilz
   * 사용하지 않는 volume 삭제

----

개요
----
> Hyperledger의 first-network 예제는 Single-Host에서 동작 되도록 구성되어 실 서비스 적용에 적합하지 않은 구조입니다. 실 서비스 적용이 가능하도록 Multi-Host로 구성하는 방법을 알아 보도록 하겠습니다.

----

구성
----
### node별 Container 구성
![구성도](https://user-images.githubusercontent.com/15353753/86920336-96da4a00-c164-11ea-8624-dd8d9cdf151d.png)

### 노드 정보
| node name | ip | container|
| :-----:| :-----------: | :------------:|
| node1    |192.168.249.11  | orderer |
| node2    |192.168.249.12  | orderer2, peer0, couchdb, cli |
| node3    |192.168.249.13  | orderer3, peer1, couchdb, cli |
| node4    |192.168.249.14  | peer3, couchdb, cli |
| node5    |192.168.249.15  | peer4, couchdb, cli |

준비 작업
----
### Host name 설정
host name이 중복되지 않도록 설정합니다.

```sh 
#명령어
$hostnamectl set-hostname [newname]
```
```sh 
#192.168.249.11
$hostnamectl set-hostname node1
```
```sh 
#192.168.249.12
$hostnamectl set-hostname node2
```
```sh 
#192.168.249.13
$hostnamectl set-hostname node3
```
```sh 
#192.168.249.14
$hostnamectl set-hostname node4
```
```sh 
#192.168.249.15
$hostnamectl set-hostname node5
```

- swarm 설정을 위한 포트를 개방 합니다.
```sh 
   $ sudo firewall-cmd --add-port=2377/tcp --permanent
   $ sudo firewall-cmd --add-port=7946/tcp --permanent
   $ sudo firewall-cmd --add-port=7946/udp --permanent
   $ sudo firewall-cmd --add-port=4789/udp --permanent   
```

- host의 date를 동기화 시킵니다

```sh 
   $ date -s 'yyyy-mm-dd hh:mm:ss'   
```   

### Docker Swarm 설정
- port 해방
```sh 
   $ sudo firewall-cmd --add-port=2377/tcp --permanent
   $ sudo firewall-cmd --add-port=7946/tcp --permanent
   $ sudo firewall-cmd --add-port=7946/udp --permanent
   $ sudo firewall-cmd --add-port=4789/udp --permanent
   $ sudo firewall-cmd --reload
```

- initialize swarm
```sh 
#node1
$docker swarm init
```
- swarm manager 등록
```sh 
#node1
$docker swarm join-token manager
```
![enroll](https://user-images.githubusercontent.com/15353753/87030247-4c1c0900-c21c-11ea-8176-6158f550f2aa.png)
```sh 
#node2,node3,node4,node5
$docker swarm join --token SWMTKN-1-39iqem217lt0fv4t1b7qs130jtynxw22bcpz5tch3dhcrr6svj-462ko5j185o3an0iumy7sqyy9 192.168.249.11:2377
```
- join 확인
```sh 
#node1
$docker node ls
```
![node ls](https://user-images.githubusercontent.com/15353753/87030750-057ade80-c21d-11ea-941f-700288214381.png)
- overlay network 생성
```sh 
#node1
$docker network create --attachable --driver overlay fabric
$docker network ls
```
![net](https://user-images.githubusercontent.com/15353753/87032148-1cbacb80-c21f-11ea-9704-d0de4d713127.png)

**인프라 준비 완료**

----

artifacts,MSP(channel-artifacts, crypto-config) 준비작업
----
> first-network의 byfn.sh를 실행하여 channel-artifacts, crypto-config를 생성하도록 하겠습니다.

> 샘플에서는 5개 Orderer Container가 실행됩니다. 그러나 우리는 3개의 Orderer Container만 사용 할 것이므로 orderer4, orderer5를 제거하기 위해 fabric-samples가 설치 되어있는 폴더로 이동합니다.
```sh 
#node1
$pwd
/home/root/prj/fabric/fabric-samples-master/first-network
```
- channel-artifacts 생성을 위한 설정
   - configtx.yaml 파일을 열어 아래와 같이 Profiles 섹션을 수정합니다.
```sh 
#node1
$vi configtx.yaml
.
.
.

    SampleMultiNodeEtcdRaft:
        <<: *ChannelDefaults
        Capabilities:
            <<: *ChannelCapabilities
        Orderer:
            <<: *OrdererDefaults
            OrdererType: etcdraft
            EtcdRaft:
                Consenters:
                - Host: orderer.example.com
                  Port: 7050
                  ClientTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/tls/server.crt
                  ServerTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/tls/server.crt
                - Host: orderer2.example.com
                  Port: 8050
                  ClientTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer2.example.com/tls/server.crt
                  ServerTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer2.example.com/tls/server.crt
                - Host: orderer3.example.com
                  Port: 9050
                  ClientTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer3.example.com/tls/server.crt
                  ServerTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer3.example.com/tls/server.crt
                #- Host: orderer4.example.com
                 # Port: 10050
                 # ClientTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer4.example.com/tls/server.crt
                 # ServerTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer4.example.com/tls/server.crt
                #- Host: orderer5.example.com
                 # Port: 11050
                 # ClientTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer5.example.com/tls/server.crt
                 # ServerTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer5.example.com/tls/server.crt
            Addresses:
                - orderer.example.com:7050
                - orderer2.example.com:8050
                - orderer3.example.com:9050
                #- orderer4.example.com:10050
                #- orderer5.example.com:11050
.
.
.
```
- crypto-config 생성을 위한 설정
   - crypto-config.yaml 파일을 열어 OrdererOrgs 섹션을 수정합니다.
 ```sh 
#node1
$vi crypto-config.yaml
.
.
.
# ---------------------------------------------------------------------------
# "OrdererOrgs" - Definition of organizations managing orderer nodes
# ---------------------------------------------------------------------------
OrdererOrgs:
  # ---------------------------------------------------------------------------
  # Orderer
  # ---------------------------------------------------------------------------
  - Name: Orderer
    Domain: example.com
    # ---------------------------------------------------------------------------
    # "Specs" - See PeerOrgs below for complete description
    # ---------------------------------------------------------------------------
    Specs:
      - Hostname: orderer
      - Hostname: orderer2
      - Hostname: orderer3
      #- Hostname: orderer4
      #- Hostname: orderer5
.
.
.
```    
- byfn.sh 실행
```sh 
#node1
$./byfn.sh down
$./byfn.sh generate
```
![gen](https://user-images.githubusercontent.com/15353753/87034141-550fd900-c222-11ea-936a-b213a3f6eb19.png)
   * **channel-artifacts, crypto-config**의 폴더가 생성 되었습니다.
   * 생성된 두 폴더는 node1의 경로와 동일한 경로로 node2,node3,node4,node5에 복사를 합니다.
   
**artifacts, msp 준비 완료**

----

git repogitory clone 
----
- git clone으로 yaml 파일을 내려 받습니다.
```sh 
#node1
$ git clone https://github.com/thexen/HyperLedger.git
```
![image](https://user-images.githubusercontent.com/15353753/87144393-a8496080-c2e2-11ea-92e1-4aed216d7746.png)

orderer yaml 파일 설정
----
* orderer.yaml, orderer2.yaml, orderer3.yaml 파일을 열어 volumes 부분을 적당히 수정 합니다.
* **channel-artifacts, crypto-config**이 생성된 path를 입력 합니다. 

```sh 
#node1
$vi docker-compose-orderer.yaml
version: '3'

volumes:
  orderer.example.com:

networks:
  fabric:
    external:
      name: fabric

.
.
.
  volumes:
      - /home/root/prj/fabric/fabric-samples-master/first-network/channel-artifacts/genesis.block:/var/hyperledger/orderer/orderer.genesis.block
      - /home/root/prj/fabric/fabric-samples-master/first-network/crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/msp/:/var/hyperledger/orderer/msp/
      - /home/root/prj/fabric/fabric-samples-master/first-network/crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/tls/:/var/hyperledger/orderer/tls/
.
.
.
```
![image](https://user-images.githubusercontent.com/15353753/87145566-7933ee80-c2e4-11ea-9987-64992b9cd118.png)

peer yaml 파일 설정
----
- peer0-org1.yaml, peer1-org1.yaml, peer0-org2.yaml, peer1-org2.yaml 파일을 열어 volumes 부분을 적당히 수정 합니다.
- service peer0 와 peer0-cli 의 volumes를 orderer와 동일하게 설정합니다
```sh 
#node1
$ vi docker-compse-peer0-org1.yaml
.
.
.

############################################################################################
#Peer: Peer0
############################################################################################
  peer0:
    image: hyperledger/fabric-peer:2.1
.
.
.
    volumes:
    - /var/run/:/host/var/run/
    - /home/root/prj/fabric/fabric-samples-master/first-network/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/msp/:/etc/hyperledger/fabric/msp/
    - /home/root/prj/fabric/fabric-samples-master/first-network/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/:/etc/hyperledger/fabric/tls/
    - peer0.org1.example.com:/var/hyperledger/production
.
.
.
############################################################################################
#cli:
############################################################################################
  peer0-cli:
.
.
.
    volumes:
        - /var/run/:/host/var/run/
        - /home/root/prj/fabric/fabric-samples-master/chaincode/:/opt/gopath/src/github.com/hyperledger/fabric-samples/chaincode
        - /home/root/prj/fabric/fabric-samples-master/first-network/crypto-config:/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/
        - /home/root/prj/fabric/fabric-samples-master/first-network/scripts:/opt/gopath/src/github.com/hyperledger/fabric/peer/scripts/
        - /home/root/prj/fabric/fabric-samples-master/first-network/channel-artifacts:/opt/gopath/src/github.com/hyperledger/fabric/peer/channel-artifacts
.
.
.
```

deploy
----
 - docker-compse 파일이 준비가 다 되었습니다.
 - 이제 container를 올려 보도록 하겠습니다.
 - docker stack deploy -c [yaml 파일 이름] fabric 을 사용하면 yaml 파일에서 설정한 node에 container가 실행됩니다.
 ```sh 
#node1
$ docker stack deploy -c docker-compose-orderer.yaml fabric
$ docker stack deploy -c docker-compose-orderer2.yaml fabric
$ docker stack deploy -c docker-compose-orderer3.yaml fabric
$ docker stack deploy -c docker-compose-peer0-org1.yaml fabric
$ docker stack deploy -c docker-compose-peer1-org1.yaml fabric
$ docker stack deploy -c docker-compose-peer0-org2.yaml fabric
$ docker stack deploy -c docker-compose-peer1-org2.yaml fabric
```
- 실행 확인
```sh 
#node1
$ dokcer service ls
```
- 아래 그림과 같이 보이면 성공적으로 container를 실행 한 것입니다.
 ![image](https://user-images.githubusercontent.com/15353753/87147275-4c350b00-c2e7-11ea-873a-b6a9700a14f3.png)
 
----
 
create channel
----
1. node2로 접속합니다.
2. cli의 container id를 구한 후 container를 실행합니다
3. mychannel을 생성합니다.
```sh 
#node2
$ dokcer ps
$ dokcer exec -it caefcfbaca7b bash
```

![image](https://user-images.githubusercontent.com/15353753/87237997-e73cfa80-c437-11ea-9494-0638a1fb5751.png)

```sh 
#node2: cli
$ peer channel create -o orderer.example.com:7050 -c mychannel -f ./channel-artifacts/channel.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```
![image](https://user-images.githubusercontent.com/15353753/87238058-9ed20c80-c438-11ea-8fa5-30c500132052.png)

* 생성된 mychannel.block 파일을 node3, node4, node5의 cli로 복사해야 합니다. 그럴려면 node2:cli(peer0-cli)에 있는 mychannel.block을 host로 복사 한 후 host에서 node3, node4, node5의 cli로 다시 복사 합니다.
>container에서 host로 복사하는 방법: docker cp [from-containerid]:/opt/gopath/src/github.com/hyperledger/fabric/peer/mychannel.block mychannel.block

```sh 
#node2
$ docker cp caefcfbaca7:/opt/gopath/src/github.com/hyperledger/fabric/peer/mychannel.block mychannel.block
```

![image](https://user-images.githubusercontent.com/15353753/87238112-51a26a80-c439-11ea-826b-b729f9587159.png)

* host로 복사된 mychannel.block 파일을 node3, node4, node5의 host로 복사 한 후 cli로 다시 복사 합니다
* node3, node4, node5의 host로 복사 하였다고 가정하고 다음 명령어를 실행 합니다.
```sh 
#node3
$ docker cp mychannel.block [to-containerid]:/opt/gopath/src/github.com/hyperledger/fabric/peer/mychannel.block
```

![image](https://user-images.githubusercontent.com/15353753/87238177-3552fd80-c43a-11ea-85ab-e9d61d937bc9.png)

```sh 
#node4
$ docker cp mychannel.block [to-containerid]:/opt/gopath/src/github.com/hyperledger/fabric/peer/mychannel.block
```
```sh 
#node5
$ docker cp mychannel.block [to-containerid]:/opt/gopath/src/github.com/hyperledger/fabric/peer/mychannel.block
```

* mychannel.block 파일은 /opt/gopath/src/github.com/hyperledger/fabric/peer/mychannel.block 에 있습니다.

join channel
----
```sh 
#node2: cli
$ peer channel join -b mychannel.block
```

![image](https://user-images.githubusercontent.com/15353753/87238214-b9a58080-c43a-11ea-9b71-78868930dfd5.png)

```sh 
#node3: cli
$ peer channel join -b mychannel.block
```
```sh 
#node4: cli
$ peer channel join -b mychannel.block
```
```sh 
#node4: cli
$ peer channel join -b mychannel.block
```
> Error: genesis block file not found open mychannel.block: no such file or directory라 출력 된다면 
> node2:cli(peer0-cli)의 mychannel.black를 node3,node4,node5의 cli로 복사하면 해결 될 것입니다.

anchor peer 설정
----
```sh 
#node2:cli
$ peer channel update -o orderer.example.com:7050 -c mychannel -f ./channel-artifacts/Org1MSPanchors.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

![image](https://user-images.githubusercontent.com/15353753/87238243-1a34bd80-c43b-11ea-8def-d791374e3f26.png)

```sh 
#node4:cli
$peer channel update -o orderer.example.com:7050 -c mychannel -f ./channel-artifacts/Org2MSPanchors.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

----

chain code
----
1. package
* node2로 접속합니다.
* cli의 container id를 구한 후 container를 실행합니다

> scripts/utils.sh 의 packageChaincode() 참고

> peer lifecycle chaincode package ${ChainCodeName}.tar.gz --path ${CC_SRC_PATH} --lang ${CC_RUNTIME_LANGUAGE} --label ${ChainCodeName}_${VERSION}

```sh 
#node2: cli
$peer lifecycle chaincode package mycc.tar.gz --path github.com/hyperledger/fabric-samples/chaincode/abstore/go/ --lang golang --label mycc_1
```

![image](https://user-images.githubusercontent.com/15353753/87238308-e6a66300-c43b-11ea-9994-61fb9b3254a2.png)

* 생성된 package 파일 mycc.tar.gz를 node3:cli, node4:cli, node5:cli의 working_dir로 복사 하거나 node3:cli, node4:cli, node5:cli에서도 package 합니다.

2. install
> scripts/utils.sh 의 installChaincode() 참고
 
> peer lifecycle chaincode install ${ChainCodeName}.tar.gz

```sh 
#node2:cli, node3:cli, node4:cli, node5:cli
$peer lifecycle chaincode install mycc.tar.gz 
 ```
 
 ![image](https://user-images.githubusercontent.com/15353753/87238364-a1366580-c43c-11ea-9f00-d2aa1d57e04e.png)
 
3. install 확인 및 package id 획득
```sh 
#node2:cli, node3:cli, node4:cli, node5:cli
$peer lifecycle chaincode queryinstalled 
 ```

![image](https://user-images.githubusercontent.com/15353753/87238437-70a2fb80-c43d-11ea-8791-edc2cc277930.png)

4. Approve for org(승인)
>  scripts/utils.sh 의 approveForMyOrg(() 참고

> peer lifecycle chaincode approveformyorg --tls --cafile $ORDERER_CA --channelID $CHANNEL_NAME --name ${ChainCodeName} --version ${VERSION} --init-required --package-id ${PACKAGE_ID} --sequence ${VERSION} --waitForEvent 

* anchor peer에서 승인합니다.
```sh 
#node2:cli, node4:cli
$peer lifecycle chaincode approveformyorg --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --channelID mychannel --name mycc --version 1 --init-required --package-id mycc_1:045aa77006e2ef1efeb7d91763914cf7453718cffd56088536f43276cfc1843e --sequence 1 --waitForEvent 
 ```
 
 ![image](https://user-images.githubusercontent.com/15353753/87238479-d7c0b000-c43d-11ea-8db4-be900fa57532.png)
 
5. Approve 상태 확인
> scripts/utils.sh 의 checkCommitReadiness() 참고
 
> peer lifecycle chaincode checkcommitreadiness --channelID $CHANNEL_NAME --name ${ChainCodeName} $PEER_CONN_PARMS --version ${VERSION} --sequence ${VERSION} --output json --init-required
 ```sh 
#node2:cli, node3:cli, node4:cli, node5:cli
$peer lifecycle chaincode checkcommitreadiness --channelID mychannel --name mycc --version 1 --sequence 1 --output json --init-required
 ```
 
 ![image](https://user-images.githubusercontent.com/15353753/87238497-03439a80-c43e-11ea-8291-b021578dd78a.png)
 
6. commit
> scripts/utils.sh 의 commitChaincodeDefinition() 참고
 
> peer lifecycle chaincode querycommitted --channelID $CHANNEL_NAME --name mycc
```sh 
#node2:cli, node3:cli, node4:cli, node5:cli
$peer lifecycle chaincode commit -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --channelID mychannel --name mycc --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:9051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt --version 1 --sequence 1 --init-required
```

![image](https://user-images.githubusercontent.com/15353753/87238529-56b5e880-c43e-11ea-8899-385ac2a62366.png)
![image](https://user-images.githubusercontent.com/15353753/87238537-7d741f00-c43e-11ea-827d-930db803473d.png)

* 정상적으로 commit이 되었다면 docker container가 추가되어 실행 된 것을 확인 할 수 있을 겁니다.

----

utilz
----
* 사용하지 않는 volume 삭제
```sh 
docker volume rm $(docker volume ls -qf dangling=true)
```
