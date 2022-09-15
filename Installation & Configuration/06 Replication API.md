# Replication API 
 
동기식 복제 시스템은, 단일 트랜잭션을 통해 다른 모든 노드와 동기화한다.       
이는 트랜잭션이 커밋될 때 모든 노드가 동일한 값을 갖는다는 것을 의미한다.          
이 프로세스는 `Group Replication` 을 통한 `Write-Set` 복제를 사용하여 발생한다.       

**Group Communication**
```
MySQL 5.7.17 버전에 도입된 새로운 복제 방식이며,   
기존 MySQL 복제 프레임워크를 기반으로 구현되어 내부 적으로 
Row 포맷의 바이너리 로그, 릴레리 로그, GTID를 이용한다.

트랜잭션 처리가 다른데, 모드 노드에서 쓰기가 가능하며 다른 노드와 읽기/쓰기가 가능한 구조다.(멀티 마스터)     
```  

## Galera Cluster 내부 아키텍처 

Galera Cluster의 내부 아키텍처는 4가지 구성 요소를 중심으로 이루어진다.       
  
![replicationapi](https://user-images.githubusercontent.com/50267433/165106959-e801fd3f-ca33-4654-9502-a3dbb48f0cd7.png)
      
* **데이터베이스 관리 시스템(DBMS)**     
  * 데이터베이스 서버
  * Galera Cluster는 MySQL, MariaDB 또는 Percona XtraDB를 사용할 수 있다.         
* **wsrep API**     
  * 데이터베이스용 복제 플러그인 인터페이스(2가지 요소로 구성)
      * **wsrep Hooks :** 쓰기 데이터 복제를 위해 데이터베이스 서버 엔진과 통합된다.
      * **dlopen() :** wsrep 후크에서 wsrep 공급자를 사용할 수 있도록 한다.
* **Galera Replication Plugin**        
  * wsrep API의 구현체로, Write-Set(gcache) 복제 기능을 제공한다.        
* **Group Communication Plugins**    
  * Galera Cluster에서 사용 가능한 Group Communication 플러그인들이 있다.   
  * 예시: gcomm 및 Spread   
  
## wsrep API

![ReflicationAPI](https://user-images.githubusercontent.com/50267433/165448416-60772e85-8536-4e1f-9f3d-a5da61356ec9.png)

**wsrep API** 는 데이터베이스용 **복제 플러그인 인터페이스다.**      
애플리케이션 콜백 및 복제 플러그인 호출 세트를 정의한다.       
   
wsrep API는 데이터베이스 서버를 상태로 간주하는 복제 모델을 사용한다.      
**데이터베이스가 사용 중이고 클라이언트가 데이터베이스 콘텐츠를 수정하면 상태가 변경된다**             
wsrep API는 데이터베이스 상태 변경을 일련의 원자적 변경 또는 트랜잭션으로 나타낸다.
      
데이터베이스 클러스터에서 모든 노드는 항상 동일한 상태를 갖는다.            
동일한 직렬 순서로 상태 변경을 복제하고 적용하여 서로 동기화한다.           
     
**보다 기술적인 관점에서 Galera Cluster는 다음과 같은 방식으로 상태 변경을 처리한다.**         
* 클러스터에 속한 임의의 노드에서 데이터베이스 상태 변경이 발생한다.     
* `wsrep hook`는 변경 사항을 Write-Set 으로 변환한다.    
* `dlopen()` 호출 후 `wsrep hook`에서 **wsrep provide 기능을 사용할 수 있다.**.     
* Galera Replication Pulgin 은 write-set 인증 및 클러스터 동기화 처리를 진행한다.    

클러스터의 각 노드에 대해 응용 프로그램 프로세스는 우선 순위가 높은 트랜잭션에 의해 발생한다.   
  
## Global Transaction ID
 
클러스터 전체에서 동일한 상태를 유지하기 위해        
wsrep API는 GTID(Global Transaction ID)를 사용한다.        
GTID 를 이용해 상태 변경을 식별하고 마지막 상태 변경과 비교하여 현재 상태를 식별할 수 있다.      

```
45eec521-2f34-11e0-0800-2a36050b826b:94530586304
```  
  
GTID(Global Transaction ID)는 다음 구성 요소로 구성된다.         
* **State UUID :** 상태에 대한 고유 식별자와 상태가 겪는 변경 순서를 나타낸다.       
* **Ordinal Sequence Number :** seqno는 시퀀스의 변경 위치를 나타내는 데 사용되는 64비트 부호 있는 정수다.   
         
글로벌 트랜잭션 ID를 사용하면 애플리케이션 상태를 비교하고 상태 변경 순서를 설정할 수 있다.         
이를 사용하여 변경 사항이 적용되었는지 여부와 변경 사항이 주어진 상태에 적용 가능한지 여부를 확인할 수 있다.    

## Galera Replication Plugin
     
Galera Replication Plugin은 wsrep API 의 구현체로 **wsrep Provider 역할을 맡고 있다.**      
기술적인 관점으로, Galera Replication Plugin은 다음과 같은 구성 요소로 구성된다.           
* **Certification Layer :** Write-Set을 준비하고 인증 검사를 수행하므로 Write-Set이 적용될 수 있다.(인증기반복제)   
* **Replication Layer :** 복제 프로토콜을 관리하고 복제와 관련된 명령을 제공한다.      
* **Group Communication Framework :** 다양한 Group Communication System을 위한 플러그인 아키텍처를 제공한다.       
   
## Group Communication Plugins  

Galera Cluster는 **`가상 동기화 QoS(QualityOfSystem)`를 구현하는 독점 그룹 통신 시스템 계층 위에 구축된다.**      
가상 동기화(QoS)는 데이터 전송 및 클러스터 멤버십 서비스를 통합하여 메시지 전달 의미에 대한 명확한 형식을 제공한다.          
  
가상 동기화는 일관성을 보장하지만 원활한 다중 마스터 작업에 필요한 일시적 동기화를 보장하지는 않는다.        
이를 해결하기 위해 Galera Cluster는 자체 런타임 구성 가능한 일시적 흐름 제어를 구현한다.        
**흐름 제어는 노드를 몇 초 만에 동기화한다.**      
    
Group Communication Framework 또한 여러 소스에서 온 메시지의 전체 순서를 제공한다.             
이를 사용 하여 Multi Master(Main) Cluster 에서 Global Transaction ID 를 생성한다.            
(GTID를 통해, 노드가 어느시점까지 동기화를 했는지 확인하고 순차대로 동기화 작업 진행)   
        
**전송 수준에서 Galera Cluster의 네트워크 구조는 무방향 대칭 그래프다.**            
모든 데이터베이스 노드는 TCP Connection 을 통해 서로 연결되어 있는 상태다.            
   
일반적으로 TCP는 `메시지 복제`와 `클러스터 구성원 서비스` 모두에서 사용된다.       
그러나 LAN 에서 복제를 위해 UDP 멀티캐스트를 사용할 수도 있다.       
