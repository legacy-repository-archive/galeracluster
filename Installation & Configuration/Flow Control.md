# Flow Control(흐름제어)   
   
Galera Cluster는 Flow Control이라는 피드백 메커니즘을 사용하여 복제 프로세스를 관리한다.       
Flow Control은 **트랜잭션을 적용할 때 노드가 뒤쳐지는 것을 방지하는 것으로 복제를 일시 중지하고 재개할 수 있다.**          

## 흐름 제어 작동 방식
   
Galera Cluster는 클러스터의 모든 노드에 순서대로 트랜잭션이 복사됨으로써 동기식 복제를 달성한다.          
즉, **트랜잭션이 클러스터를 통해 복제될 때 트랜잭션이 적용되고 커밋이 비동기적으로 발생한다.**     
              
노드는 **클러스터에서 Write-Set 을 수신하고 이를 전역 순서로 구성한다.**              
**적용 및 커밋되지 않은 트랜잭션이 있다면 이를 Received Queue에 보관한다.**        
            
**Received Queue 가 특정 크기에 도달하면 FlowControl을 동작시킨다.**               
노드는 복제를 일시 중지한 다음 수신된 대기열의 WriteSet에 대해서 작업한다.         
Received Queue 내의 데이터량이 줄어들면 노드는 다시 복제를 재개한다.          
    
## 노드 상태 이해  
        
Galera Cluster 는 상태에 따라 여러 형태의 Flow Control을 구현한다.         
Flow Control 구현체는 가상 동기화가 제공하는 논리적인 것과는 대조적으로 시간적 동기화와 일관성을 보장한다.      
         
**Flow Control 에는 기본적으로 네 가지 종류가 있다.**       
* No Flow Control
* Write-Set Caching  
* Catching Up
* Cluster Sync  

## No Flow Control
          
OPEN 또는 PRIMARY 상태일 때 작동하는 FlowControl 이다.     
노드가 OPEN 또는 PRIMARY 상태를 유지하면 클러스터의 일부로 간주하지 않는다.           
이 상태의 노드는 Write-Set을 복제 및 적용 또는 캐시할 수 없다.         
  
## Write-Set Caching  
             
JOINER 및 DONOR 상태일 때 작동하는 FlowControl 이다.       
노드는 **쓰기 세트를 적용할 수 없으며 나중을 위해 캐시해야 한다.**                 
모든 복제를 중지하는 것 외에는 노드를 클러스터와 동기화된 상태로 유지하는 합리적인 방법이 없다.      
       
**Write-Set Cache가 할당된 크기를 초과하지 않도록 복제 속도를 제한할 수 있다.**         
다음 매개변수를 사용하여 Write-Set Cache를 제어할 수 있다.       
    
* **gcs.recv_q_hard_limit :** 최대 쓰기 세트 캐시 크기(바이트).
* **gcs.max_throttle :** 노드가 클러스터에서 허용할 수 있는 일반 복제 속도에 대한 가장 작은 부분이다.
* **gcs.recv_q_soft_limit :** 노드의 평균 복제 속도 추정치.
  
## Catching Up
            
JOINED 상태에 있을 때 작동하는 FlowControl 이다.     
Write-Set을 반영할 수 있고 노드가 클러스터의 반영 속도를 따라잡을 수 있도록 보장한다.                         
**특히, Write-Set Cache가 절대 커지지 않도록 보장한다.**                  
클러스터 전체 복제 속도는 **이 상태에서 Write-Set 을 적용할 수 있는 속도에 의해 제한된다.**            
Write-Set은 일반적인 트랜잭션을 처리 방법보다 몇 배 빠르기에 클러스터 성능에 거의 영향을 미치지 않는다.         
노드의 상태가 JOINED 일때, 버퍼 풀이 비어 있는 맨 처음에 딱 한 번 클러스터 성능에 영향을 미친다.        
  
## Cluster Sync  
    
SYNCED상태에 있을 때 작동하는 FlowControl 이다.                   
**Cluster Sync는 슬레이브 대기열을 최소로 유지하려고 시도한다.**           
다음 매개변수를 사용하여 노드가 이를 처리하는 방법을 구성할 수 있다.  
   
* **gcs.fc_limit :** 흐름 제어가 작동하는 지점을 결정하는 데 사용된다.
* **gcs.fc_factor :** 흐름 제어가 해제되는 지점을 결정하는 데 사용된다.  

# 노드 상태의 변경 사항
    
노드 상태 장치의 Galera Cluster의 다른 레이어에서 다양한 상태 변경을 처리한다.   
다음은 최상위 레이어에서 발생하는 노드 상태 변경 사항이다.  
   
![galerafsm](https://user-images.githubusercontent.com/50267433/165318335-72d1b84a-a826-4f57-a581-204baab1f6b2.png)
  
1. 노드가 시작되고 기본 구성 요소에 대한 연결이 설정된다.   
2. 노드가 상태 전송 요청에 성공하면 Write-Set을 캐시한다.    
3. 노드는 상태 스냅샷을 전송 받는다.       
   이제 모든 클러스터 데이터가 있고 캐시된 WriteSet을 적용하기 시작한다.      
   여기서 노드는 FlowControl이 Receive Queue의 궁극적인 감소를 보장할 수 있도록 한다.     
4. 노드가 클러스터 따라잡기를 완료한다.     
   이제 Receive Queue는 비어 있다.          
   노드는 MySQL 상태 변수 wsrep_ready 를 값으로 설정한다.   
   이제 노드에서 트랜잭션을 처리할 수 있다.           
5. 노드는 상태 이전 요청을 수신한다.  
   FlowControl이 Donor로 이양된다.    
   노드는 적용할 수 없는 모든 WriteSet을 캐시한다.(GCache)     
6. 노드는 Joiner Node 로 상태 전송을 완료한다.      
  
