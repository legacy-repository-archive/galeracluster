# Database Replication

데이터베이스 이중화는 하나의 데이터베이스에서 다른 데이터베이스로 데이터를 복사하는 것을 맗나다.     
데이터베이스 이중화 시스템은 모든 노드가 동일한 수준의 정보를 공유하는 분산 데이터베이스이다.    

## Master And Slave

대다수의 DBMS는 데이터베이스 이중화를 지원한다.   
가장 일반적인 이중화 구조는 `마스터(원본)/슬레이브(복사본)` 관계를 채택하고 있다.  

![asynchronousreplication](https://user-images.githubusercontent.com/50267433/164360492-25bade02-af8a-476d-a07c-eaffc65bf90e.png)

마스터 데이터베이스 서버는 데이터 업데이트를 기록하고 **해당 로그**를 네트워크를 통해 슬레이브로 전파한다.   
슬레이브 데이터베이스 서버는 마스터로부터 업데이트 스트림을 수신하고 이러한 변경 사항을 적용한다.   

![synchronousreplication](https://user-images.githubusercontent.com/50267433/164360556-850e1775-727b-4ab6-b25f-d7958769e429.png)

다중 마스터 이중화 시스템에서는 아무 데이터베이스 노드에 업데이트를 제출할 수 있으며 네트워크를 통해 다른 데이터베이스 노드로 전파된다.    
모든 데이터베이스는 마스터로 작동한다.     
변경에 대한 로그가 없으며 업데이트 성공 여부를 알려주는 표시기가 전송되지 않는다.  

## Asynchronous and Synchronous Replication(동기 및 비동기 이중화)  
 
