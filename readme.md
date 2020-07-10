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
* chain code 

  
개요
----
> Hyperledger의 first-network 예제는 Single-Host에서 동작 되도록 구성되어 실 서비스 적용에 적합하지 않은 구조입니다. 실 서비스 적용이 가능하도록 Multi-Host로 구성하는 방법을 알아 보도록 하겠습니다.

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

### Docker Swarm 설정
- initialize swarm
```sh 
#node1
$docker swarm init
```
- swarm manager 등록
```sh 
#node1
$ocker swarm join-token manager
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
$ docker stack deploy -c docker-compse-peer0-org1.yaml fabric
$ docker stack deploy -c docker-compse-peer1-org1.yaml fabric
$ docker stack deploy -c docker-compse-peer0-org2.yaml fabric
$ docker stack deploy -c docker-compse-peer1-org2.yaml fabric
```
- 실행 확인
```sh 
#node1
$ dokcer service ls
```
- 아래 그림과 같이 보이면 성공적으로 container를 실행 한 것입니다.
 ![image](https://user-images.githubusercontent.com/15353753/87147275-4c350b00-c2e7-11ea-873a-b6a9700a14f3.png)
 
chain code
----
1. package
* node2로 접속합니다.
* cli의 container id를 구한 후 container를 실행합니다
```sh 
#node2
$ dokcer ps
```
 
<pre>
<code>
1. 192.168.249.11 - node1
2. 192.168.249.12 - node2
3. 192.168.249.13 - node3
4  192.168.249.14 - node4
5. 192.168.249.15 - node5
</code>
</pre>

# 11
# 22
이것을 **강조**합니다.


``` c
int val = 10;
printf(%s,"Hello, World!");
```

```javascript 
function test() { 
 console.log("hello world!"); 
} 
```

- [x] 체크박스
- [ ] 체크박스


   $ sudo firewall-cmd --add-port=2377/tcp --permanent
   $ sudo firewall-cmd --add-port=7946/tcp --permanent
   $ sudo firewall-cmd --add-port=7946/udp --permanent
   $ sudo firewall-cmd --add-port=4789/udp –permanent
   $ sudo firewall-cmd --reload
