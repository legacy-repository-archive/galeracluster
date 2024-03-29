# Database Replication

데이터베이스 이중화는 하나의 데이터베이스에서 다른 데이터베이스로 데이터를 복사하는 것을 말한다.     
데이터베이스 이중화 시스템은 모든 노드가 동일한 수준의 정보를 공유하는 분산 데이터베이스다.    
   
## Master And Slave

대다수의 DBMS는 데이터베이스 이중화를 지원한다.   
가장 일반적인 이중화 구조는 `마스터(원본)/슬레이브(복사본)` 관계를 채택하고 있다.  
최근에는 이름상의 문제로 `메인/레플리카` 라는 이름을 사용하고 있다.    

![asynchronousreplication](https://user-images.githubusercontent.com/50267433/164360492-25bade02-af8a-476d-a07c-eaffc65bf90e.png)
    
Main DB 서버는 데이터 업데이트를 기록하고 **해당 업데이트 로그를 네트워크를 통해 Replica DB로 전파한다.**       
Replica DB 서버는 Main DB서버로 부터 업데이트 스트림을 수신하고 변경 사항을 적용한다.       

![synchronousreplication](https://user-images.githubusercontent.com/50267433/164360556-850e1775-727b-4ab6-b25f-d7958769e429.png)

Multi Main(Master) Replication 시스템에서는 아무 노드에나 업데이트 할 수 있으며 네트워크를 통해 다른 데이터베이스 노드로 전파된다.    
단, 변경에 대한 로그가 없으며 업데이트 성공 여부를 알려주는 표시가 전송되지 않는다.        
**갈레라 클러스터는 Multi Main(Master) RDB Cluster로 위와 같은 프로세스를 따른다.**     
  
## Asynchronous and Synchronous Replication(비동기 및 동기 이중화)    
관계를 가진 데이터베이스 서버끼리 트랜잭션을 전파하는 프로토콜에도 여러 종류가 있다.    
     
**Synchronous Replication(동기식 복제)**     
* 빠른 복제 방식을 지원한다.    
* Main DB는 복제본 업데이트를 동기적으로 전파한다.  
  즉, Main DB에서 다른 DB로 전파가 완료되면 커밋이 된다.   
* 모든 노드들은 동기화된 상태로 유지된다.   
 
**Asynchronous Replication(비동기식 복제)**      
* 느린 복제 방식을 지원한다.     
* Main DB는 복제본 업데이트를 비동기적으로 전파한다.     
  즉, Main DB에서 다른 DB로 전파시 비동기 복제를 진행하고 커밋된다.     
* 다른 노드가 동기화가 되지 전에 커밋이 될 수 있으므로 특정 시간동안 일부 노드는 다른 값을 보유할 수 있다.    
  
모든 노드가 동기화 되는 시점에 따라, 이른/느린 으로 구분한다고 생각하면 좋다.        
비동기는 메인 노드가 빨리 커밋되지만 전체적인 동기화 시점은 느리다고 보면 된다.      
  
## Advantages of Synchronous Replication(동기식 복제의 장점)
       
동기 복제는 비동기 복제에 비해 몇 가지 장점이 있다.     

1. 동기 이중화는 고가용성을 제공하고 24/7 서비스를 보증한다.     
   노드가 충돌해도, 동기식으로 복제를하니 데이터 손실이 없고 일관성을 유지한다.  
2. 동기 이중화는 모든 노드에서 트랜잭션을 병렬로 실행하여 성능을 향상시킨다.    
3. 동기 이중화의 인과관계는 전체 클러스터에 걸쳐 인과관계를 보장한다.        

## Disadvantages of Synchronous Replication(동기식 복제의 단점) 

동기식 복제 방식은 일반적으로 하나의 노드씩 접근하여 동기화를 실행한다.     
그리고 이 과정에서 2단계의 커밋 또는 분산 잠금을 사용한다.          
 
1개의 노드당 O개의 작업 수행을 하며 하나의 작업당 t 트랜잭션 처리량을 가진다.       
즉, N개의 노드가 있다면 `N개 노드 개수` X `O개의 작업수` X `t 트랜잭션 처리량`을 가진다.    
                 
```
M = N x O x T   
```  
  
이를 다르게 말하면, 노드 수 증가에따라 트랜잭션 응답 시간과 충돌 및 교착 상태 비율이 기하급수적으로 증가한다.            
이러한 이유로, 대다수의 DBMS는 **데이터베이스 성능, 확장성 및 가용성을 위한 비동기 복제 프로토콜을 남겨 둔다.**              
`MySQL` 및 `Postgre`와 같이 널리 채택된 오픈 소스 데이터베이스 SQL은 **비동기 복제 솔루션만 제공하기도 한다**          
    
## Solving the Issues in Synchronous Replication(동기식 복제 방식 문제 해결)  
       
동기식 복제 방식의 문제를 해결하기 위한 대안은 아래와 같다.       
    
* **Group Communication :** 일관성을 보장하면서 노드 간의 통신을 위한 높은 단계의 gcomm 또는 spread 
* **Write-Set :** 너무 많은 Node coordination을  피하기 위해 단일 기록 집합을 작성하여 노드간의 작업 수를 줄인다.
* **갈레라 클러스터 자체적인 GTID 사용 :** MariaDB GTID 복제 매카니즘은 사용하지 않는다. 
* **데이터베이스 상태 장치 :** 
    * 읽기 트랜잭션은 로컬 노드에서 처리한다.   
    * 쓰기 트랜잭션은 로컬에서 Shadow copy로 수행 되어 인증과 커밋을 위해 다른 노드에 읽기 집합으로 브로드캐스트한다.  
* **트랜잭션 재배열 :** 
    * 다른 노드와 트랜잭션 완료전에 트랜잭션을 재배열한다. 
    * 이로써 성공적인 트랜잭션 인증 테스트의 숫자를 증가시킬 수 있다.
       
Galera Cluster가 사용하는 인증 기반 복제 시스템은 이러한 접근 방식을 기반으로 구축되었다.

# 참고 
* [갈레라 클러스터 레퍼런스](https://galeracluster.com/library/documentation/index.html)
* [갈레라 클러스터 (Galera Cluster): Multi Master Replication](https://rastalion.me/galera-cluster/)
