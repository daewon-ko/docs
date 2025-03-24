# 2PC와 Saga패턴





**2PC**



- 2Phase Pattern
  - 분산환경에서 트랜잭션의 원자성 보장을 위한 패턴
- Coordinator
  - 분산 트랜잭션을 관리하는 주체
- 2가지 Phase가 존재
  - Prepare Phase
  - Commit Phase



**Flow**



**Prepare Phase** 



- Prepare Phase에서 특정 트랜잭션에 Coordinator에게 Prepare여부를 물어본다.
  - ex) Board DB에 Board 생성 가능 여부 질의 후 해당 Row 잠금처리
  - Member DB에 해당하는 Member Row 수정가능 여부 질의 후 해당 Row 잠금
- 대답은 ‘Ok’ Or ‘NO’
- 각 서비스의 응답이 모두 온 이후에 Commit Or RollBack 여부 판단
- 따라서 Prepare Phase에서는 해당하는 DB에 관련된 모든 서비스의 응답이 올때까지 Lock을 설정



**Commit Phase**

- 하나의 DB라도 요청을 처리할 수 없으면 RollBack
- 모든 DB에서 요청을 처리할 수 있으면 Commit



**문제점**

- 모든 요청을 처리할때까지 DB Lock
  - → 지연시간 상승
- 서비스간 강결합 초래
  - MSA의 장점인 결합도를 낮추는 것의 장점이 퇴색됨.





**SAGA 패턴**



2PC와 다르게 모든 DB 트랜잭션을 순차적으로 처리



- SAGA는 DB 트랜잭션을 동시에 처리하지 않고 각 DB의 트랜잭션(로컬트랜잭션)을 처리한다.
- 각 서비스 간 이벤트를 통해 로컬 트랜잭션을 순차적으로 처리
- 트랜잭션 상태 체크시 처리가 안될 경우 ‘보상트랜잭션’을 통해 원자성 보장