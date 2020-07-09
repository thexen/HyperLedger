HyperLedger Multi-Host 구성하기
===============================
   
* 개요
* 구성도
* NODE 정보  
* 준비작업
  * host name 설정
  * docker swarm 설정
  * overlay network 설정

  
개요
----
> Hyperledger의 first-network 예제는 Single-Host에서 동작 되도록 구성되어 실 서비스 적용에 적합하지 않은 구조입니다. 실 서비스 적용이 가능하도록 Multi-Host로 구성하는 방법을 알아 보도록 하겠습니다.

구성도
----
![구성도](https://user-images.githubusercontent.com/15353753/86920336-96da4a00-c164-11ea-8624-dd8d9cdf151d.png)

NODE 정보
----
| NODE  | ip | container|
| :-----| :-----------: | :------------|
| node1    |192.168.249.11  | orderer |
| node2    |192.168.249.12  | orderer2, peer0, couchdb, cli |
| node3    |192.168.249.13  | orderer3, peer1, couchdb, cli |
| node4    |192.168.249.14  | peer3, couchdb, cli |
| node5    |192.168.249.15  | peer4, couchdb, cli |

<pre>
<code>
1. node1 - 192.168.249.11
2. node2 - 192.168.249.12
3. node3 - 192.168.249.13
4 .node4 - 192.168.249.14
5. node5 - 192.168.249.15
</code>
</pre>
