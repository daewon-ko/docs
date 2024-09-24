# EntityManager 

### EntityManager

- Spring 프레임워크 내에서 EntityManager는 트랜잭션 범위 내에서 관리된다.
  - 스프링같은 컨테이너 환경에서 JPA를 사용하기에 가능한 것이며, 만약 그런 환경의 도움을 받지 못한다면, 직접 EntityManager를 생성하고 관리해야한다.
- 즉 하나의 트랜잭션 안에서 같은 스레드가 EntityManager를 사용할때, 동일한 인스턴스를 공유한다는 뜻이다.
  - 즉, 각각 다른 Repository를 이용하여 다른 EntityManager를 사용한다고 해도 동일한 트랜잭션 범위 내에서는 동일한 영속성 컨텍스트임이 보장된다.
- 따라서 쓰레드마다 고유하게 관리되므로 멀티쓰레드 상황에서 안전하다.