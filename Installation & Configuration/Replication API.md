# Replication API 
 
동기식 복제 시스템은 일반적으로 즉시 복제를 사용한다.      
클러스터의 노드는 **단일 트랜잭션을 통해 복제본을 업데이트하여 다른 모든 노드와 동기화한다.**      
    
* 이는 트랜잭션이 커밋될 때 모든 노드가 동일한 값을 갖는다는 것을 의미한다.        
* 이 프로세스는 Group Communication 을 통한 Write-Set 복제를 사용하여 발생한다.     

![replicationapi](https://user-images.githubusercontent.com/50267433/165106959-e801fd3f-ca33-4654-9502-a3dbb48f0cd7.png)

Galera Cluster의 내부 아키텍처는 4가지 구성 요소를 중심으로 이루어진다.      
   
**데이터베이스 관리 시스템(DBMS)**     
* 데이터베이스 서버
* Galera Cluster는 MySQL, MariaDB 또는 Percona XtraDB를 사용할 수 있다.    
     
**wsrep API**     
* 데이터베이스용 복제 플러그인 인터페이스다.  
* 2가지 요소로 구성된다.
    * wsrep Hooks: 쓰기 데이터 복제를 위해 데이터베이스 서버 엔진과 통합된다.
    * dlopen(): wsrep 후크에서 wsrep 공급자를 사용할 수 있도록 한다.
      
**Galera Replication Plugin**        
* wsrep API의 구현체로, Write-Set 복제 기능을 제공한다.      
  
**Group Communication Plugins**    
* Galera Cluster에서 사용 가능한 Group Communication 플러그인들이 있다.   
* 예시: gcomm 및 Spread   
  
## wsrep API

![ReflicationAPI](https://user-images.githubusercontent.com/50267433/165448416-60772e85-8536-4e1f-9f3d-a5da61356ec9.png)

wsrep API 는 데이터베이스용 **복제 플러그인 인터페이스다.**    
애플리케이션 콜백 및 복제 플러그인 호출 세트를 정의한다.   
   
wsrep API는 데이터베이스 서버를 상태로 간주하는 복제 모델을 사용한다.(상태는 데이터베이스의 내용을 나타낸다.)    
**데이터베이스가 사용 중이고 클라이언트가 데이터베이스 콘텐츠를 수정하면 상태가 변경된다.**          
wsrep API는 **데이터베이스 상태 변경을 일련의 원자적 변경 또는 트랜잭션으로 나타낸다.**      
      
데이터베이스 클러스터에서 모든 노드는 항상 동일한 상태를 갖는다.            
동일한 직렬 순서로 상태 변경을 복제하고 적용하여 서로 동기화한다.        
   
**보다 기술적인 관점에서 Galera Cluster는 다음과 같은 방식으로 상태 변경을 처리한다.**         
* 클러스터에 속한 어느 노드에서 데이터베이스 상태 변경이 발생한다.     
* 데이터베이스에서 wsrep hook는 변경 사항을 Write-Set 으로 변환한다.    
* `dlopen()` 호출한 다음 wsrep hook에서 wsrep provide 기능을 사용할 수 있다.   
* Galera Replication Pulgin 은 쓰기 데이터 인증 및 클러스터 복제 처리를 진행한다.    
* 업데이트 내용을 GCache 라는 영역에 저장한다.   

클러스터의 각 노드에 대해 응용 프로그램 프로세스는 우선 순위가 높은 트랜잭션에 의해 발생한다.   
  
## Global Transaction ID
 
클러스터 전체에서 동일한 상태를 유지하기 위해        
wsrep API는 GTID(Global Transaction ID)를 사용한다.        
GTID 를 이용해 상태 변경을 식별하고 마지막 상태 변경과 비교하여 현재 상태를 식별할 수 있다.      

```
45eec521-2f34-11e0-0800-2a36050b826b:94530586304
```  
  
GTID(Global Transaction ID)는 다음 구성 요소로 구성된다.         
* State UUID : 상태에 대한 고유 식별자와 상태가 겪는 변경 순서를 나타낸다.       
* Ordinal Sequence Number : seqno는 시퀀스의 변경 위치를 나타내는 데 사용되는 64비트 부호 있는 정수다.   
     
글로벌 트랜잭션 ID를 사용하면 애플리케이션 상태를 비교하고 상태 변경 순서를 설정할 수 있다.      
이를 사용하여 변경 사항이 적용되었는지 여부와 변경 사항이 주어진 상태에 적용 가능한지 여부를 확인할 수 있다.   

## Galera Replication Plugin
   
Galera Replication Plugin은 wsrep API를 구현한다.         
즉, wsrep 공급자로 작동한다.        
  
보다 기술적인 관점에서 볼 때, Galera Replication Plugin은 다음과 같은 구성 요소로 구성됩니다.   

* Certification Layer: 
    * 쓰기 데이터를 준비하고 인증 검사를 수행하므로 쓰기 데이터가 적용될 수 있다.
* Replication Layer: 
    * 복제 프로토콜을 관리하고 전체 주문 기능을 제공한다.
* Group Communication Framework: 
    * Galera Cluster에 연결하는 다양한 그룹 통신 시스템을 위한 플러그인 아키텍처를 제공한다.

## Group Communication Plugins  

그룹 통신 프레임워크는 다양한 `gcomm` 시스템을 위한 플러그인 아키텍처를 제공한다.       
Galera Cluster는 가상 동기화 QoS 를 구현하는 독점 그룹 통신 시스템 계층 위에 구축된다.         
가상 동기화는 데이터 전달 및 클러스터 멤버십 서비스를 통합하여 메시지 전달 의미 체계에 대한 명확한 형식을 제공한다.     
       
가상 동기화는 일관성을 보장하지만 원활한 다중 마스터 작업에 필요한 시간적 동기화를 보장하지는 않는다.           
이 문제를 해결하기 위해 Galera Cluster는 자체 런타임 구성 가능 시간 흐름 제어를 구현한다.         
**흐름 제어는 노드 동기화를 1초 미만으로 유지한다.**        
    
그룹 커뮤니케이션 프레임워크는 또한 여러 소스에서 온 메시지의 전체 순서를 제공한다.       
이를 사용 하여 다중 마스터 클러스터에서 글로벌 트랜잭션 ID 를 생성 한다.      
     
전송 수준에서 Galera Cluster는 대칭 무방향 그래프다.       
모든 데이터베이스 노드는 TCP 연결 을 통해 서로 연결된다.        
기본적으로 TCP 는 메시지 복제와 클러스터 구성원 서비스 모두에 사용된다.      
그러나 LAN 에서 복제를 위해 UDP 멀티캐스트를 사용할 수도 있다.   
