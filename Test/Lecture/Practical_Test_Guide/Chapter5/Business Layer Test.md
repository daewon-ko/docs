# Business Layer Test

* 비즈니스 로직을 구현하는 역할

* Persistence Layer와의 상호작용을 통해 비즈니스 로직을 전개시킨다. 

* ##### 트랜잭션을 보장해야 한다.

* Persistence Layer를 끼고 정말 통합테스트처럼 작동하게끔 구성한다. 

  

### @DataJpaTest

* @Transactional이 안에 포함되어이

