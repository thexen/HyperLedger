HyperLedger Multi-Host 구성하기
===============================
   
* 개요
* 구성 목표
* NODE 정보  
* 준비작업
  * host name 설정
  * docker swarm 설정
  * overlay network 설정

  
개요
----
> Hyperledger의 first-network 예제는 Single-Host에서 동작 되도록 구성되어 실 서비스 적용에 적합하지 않은 구조입니다. 실 서비스 적용이 가능하도록 Multi-Host로 구성하는 방법을 알아 보도록 하겠습니다.

구성
----
### 구성 Container
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
![node ls](https://user-images.githubusercontent.com/15353753/87030750-057ade80-c21d-11ea-941f-700288214381.png)

```sh 
#node2,node3,node4,node5
$docker swarm join --token SWMTKN-1-39iqem217lt0fv4t1b7qs130jtynxw22bcpz5tch3dhcrr6svj-462ko5j185o3an0iumy7sqyy9 192.168.249.11:2377
```
- join 확인
```sh 
#node1
$docker node ls
```
![enroll](https://user-images.githubusercontent.com/15353753/87030247-4c1c0900-c21c-11ea-8176-6158f550f2aa.png)

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
