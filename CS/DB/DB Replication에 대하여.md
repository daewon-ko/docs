**DB - Master / Slave**

\#docs/DB

\#docs/쇼핑몰프로젝트



대규모 트래픽을 처리하는 서버를 떠올려보자.



서버는 각 요청마다 쓰레드를 할당할 것이고 해당 요청이 읽기 /쓰기 작업을 요한다면 뒷단을 넘어 DB 서버에도 요청이 갈 것이다. 



그런데 보통의 트래픽이 아니라 대규모 트래픽이라면 수 많은 DB로의 요청은 DB 입장에선 많은 부하를 감당해야한다.



DB의 성능을 향상시켜서 Scale-Up을 한다고 해도 한계가 존재하기 때문에 WAS를 Scale-Out하여 성능향상을 꾀하는 것처럼 DB역시 Scale Out을 하게된다.





**Replication**



DB를 Scale Out하는 과정에서 Replication이라는 기술을 사용하게된다.



Replication은 메인 DB(Master Or Source DB)가 서브 DB(Slave Or Replica DB)와 동기화를 자동으로 유지하는 과정을 의미한다. 



통상 Master(Replica) DB에 쓰기 작업과 읽기작업을 할당하고, Slave DB에는 읽기 작업만 전담시키도록 구성을 한다. 





이하에선 Master DB를 소스 서버(DB), Slave DB를 레플리카 서버(DB)로 혼용하며 사용하도록 하겠다. 



**Replication의 과정**



![image](https://github.com/user-attachments/assets/0b527708-6ba8-4d8b-95c0-671c241111a9)











**데이터 동기화의 문제**



Replication 과정 중 중요한 포인트 중 하나는 DB간의 데이터 동기화를 어떻게 맞추는가에 관한 문제이다.



아무리 Master DB와 Slave DB의 역할분담을 한다고 해도, 동일한 요청에 대해서 다른 데이터 값을 반환해서는 안되기 때문이다.  



따라서 Master DB와 Slave DB간의 Sync를 맞추는 과정이 매우 중요한데 그 과정은 과 ‘비동기식 방법’과 ‘반동기식 방법’이 존재한다.







**비동기식 방식**

- Master 노드의 변경과 Slave Node의 변경이 시차를 두고 반영된다.
- 빠르게 응답이 가능하지만, 데이터 일관성에서 문제가 발생할 수 있다.

![IMG_9277](https://github.com/user-attachments/assets/9f47f64a-3217-4da6-b3f4-1bde2af39b94)



- 소스서버에서 레플리카 서버로 복제작업을 하는 것이 주기적으로 일어난다.
- 소스서버의 데이터 저장/수정 작업과 레플리카 서버의 작업은 연관이 없다.
- 소스서버 관점에서는 트랜잭션 처리 시에 레플리카 작업의 유무와 상관없으므로 빠른 처리가 가능하고, 레플리카 서버에서 문제점이 발생해도 롤백이 되지 않으므로 성능 상 이점이 존재한다.
  - 그러나 소스서버에 문제가 생겨서 레플리카서버가 새로운 소스서버로 변경될 시에 누락된 트랜잭션이 있을 수도 있기에 개발자가 일일이 확인 후 수동으로 적용해줘야한다.



**반동기식 방식**

- 소스서버에 데이터 변경이 발생하면 즉시 레플리카 서버까지 변경이 적용된다.
- 데이터 일관성이 높지만 Latency가 증가한다.

- 반동기 복제 방식도 어느 시점에 이벤트를 레플리카 서버로 전송하고 응답받느냐에 따라 AFTER_SYNC와 AFETR_COMMIT으로 나뉜다.



1. AFTER_SYNC

   ![IMG_9278](https://github.com/user-attachments/assets/ae551e41-dce7-4239-852d-3e430107b78b)

*AFTER_SYNC 반동기 복제방식*



- 소스 서버에 바이너리 로그에 기록 후 레플리카 서버로 바이너리 로그를 dump한다.
- 레플리카 서버의 릴레이 로그에 이벤트가 잘 기록되면 응답을 주고, 소스서버는 응답을 받은 후에 스토리지 엔진에 트랜잭션을 커밋시키고 클라이언트에 결과를 반환한다.
  - 레플리카 서버에 이벤트 전송을 보장하는 것이지, 해당 트랜잭션이 레플리카 서버에까지 적용되는 것과는 연관이 없다.
- 팬텀리드를 방지한다.

2. AFTER_COMMIT

![IMG_9279](https://github.com/user-attachments/assets/615f92d7-be37-4cae-8a24-56ba4c6e2ce1)

- 소스서버의 바이너리 로그에 기록을 하고 스토리지 엔진에 커밋을 한 이후 레플리카 서버로 이벤트를 보낸다.
- 레플리카 서버에서 문제가 생겨 롤백이 될경우, 소스서버에만 데이터가 남는 문제가 있다.
- 따라서 MYSQL 8.0부터는 AFTER_SYNC가 기본값이다.



참고 : Real Mysql 8.0 2권











[Mysql Replication 구성하기](https://escapefromcoding.tistory.com/710)



[[Springboot\] Multi DataSource(readerDB, writerDB) 설정하고 라우팅하기](https://velog.io/@win-luck/Springboot-Multi-DataSourcereaderDB-writerDB-설정하고-라우팅하기)