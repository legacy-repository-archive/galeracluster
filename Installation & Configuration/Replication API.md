# Replication API 
 
동기식 복제 시스템은 일반적으로 즉시 복제를 사용한다.      
클러스터의 노드는 **단일 트랜잭션을 통해 복제본을 업데이트하여 다른 모든 노드와 동기화한다.**      
  
* 이는 트랜잭션이 커밋될 때 모든 노드가 동일한 값을 갖는다는 것을 의미한다.      
* 이 프로세스는 그룹 통신을 통한 쓰기 데이터 복제를 사용하여 발생한다.   


