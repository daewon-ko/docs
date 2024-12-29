**@Transactional의 동작원리**



**@Transactional은 어떻게 동작할까?**



Spring을 이용하여 코드를 작성하면서 @Transactional을 이용해본적 없는 개발자는 없을 것이다. 해당 어노테이션은 전파유형, 롤백규칙, 격리 수준 등을 커스텀하게 설정할 수 있는 아주 흔히 쓰이며 유용성 높은 어노테이션이다. 



해당 기능은 스프링의 정수라고 할 수 있는 AOP와 PSA를 아주 잘 이용하고 구축한 사례라고 할 수 있는데, 이를 그래서 누군가에게 설명한다고 가정해보니 생각보다 막연하다는 생각에 한 번 정리해본다. 



**Spring과 Transaction**



@Transactional 이해에 앞서 Spring 자체에서 트랜잭션을 어떠한 관점에서 보고 어떻게 다루는지를 이해할 필요가 있다. 



스프링은 다음의 3가지 방식으로 트랜잭션을 다룬다.

- 트랜잭션 동기화
  - TransactionSynchronizationManager를 통해 트랜잭션 동기화를 실행한다.
  - TransactionSynchronizationManager는 내부에 트랜잭션 관련된 정보를 ThreadLocal을 이용하여 저장하여 동일한 스레드 내에서 같은 정보를 공유할 수 있게끔 구성한다.
    - 즉 ThreadLocal 내에 트랜잭션 관련 정보를 저장함으로써 동일한 트랜잭션 보장이 가능하다.

![image 3.png](blob:file:///c61ce84d-4b60-4bd1-a09a-6ff537c4fbd1)

- 또한 PlatformTransactionManager의 각 구현체에서 TransactionSynchronizationManager를 아래와 같은 방식으로 이용하여 하나의 쓰레드 내에서 트랜잭션 컨텍스를 유지할 수 있다.
  - DataSourceTransactionManager의 JDBC 트랜잭션 생성 코드



  @Override protected void doBegin(Object transaction, TransactionDefinition definition) { DataSourceTransactionObjecttxObject = (DataSourceTransactionObject) transaction; 

   

  Connection con =DataSourceUtils.getConnection(this.dataSource); txObject.setConnectionHolder(new ConnectionHolder(con));

   // ThreadLocal에 트랜잭션 동기화 TransactionSynchronizationManager.bindResource(this.dataSource,txObject.getConnectionHolder()); 

  } 



- 트랜잭션 추상화
  - Spring은 PlatformTransactionManager라는 인터페이스를 통해 트랜잭션 추상화 기술을 제공한다.
  - JDBC, JPA 등 DB 접근하는 기술이 무엇이느냐에 따라서 각 구현체가 다르다 (하단 그림참조)

![image 2.png](blob:file:///3c3ea46e-df5e-44cc-899b-d1c4d23d479f)

- AOP를 이용한 트랜잭션 분리
- 
-  public void addUser(User user) {
-  	TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
-  	
-  	try {
-  		userRepository.save(user) // 비즈니스 로직
-  
-  		this.transactionManager.commit(status);
-  	} catch (Exception e) {
-  		this.transactionManager.rollback(status);
-  		throw e
-  	}
-  }
-  
- 
- 위와 같은 코드에서 비즈니스로직과 Transaction 관련 코드가 뒤섞여있는데, 이를 @Transactional이라는 선언형 어노테이션을 통해 비즈니스 로직에 집중할 수 있다.





**다시 돌아와 @Transactional에의 이해**

@Transactional은 해당 어노테이션이 붙은 클래스 혹은 메서드를 AOP의 ‘PointCut’으로 등록한다는 의미인데, 여기서 말하는 PointCut은 Spring AOP에서 말하는 주요 개념 중 하나로서 Advice가 적용될 위치 또는 끼어들 수 잇는 시점을 의미한다. 자세한 개념은 AOP를 학습해야하므로 해당 글의 범위와 약간 맥락이 다를 수 있기 때문에 일단은 스킵하겠다. 



**Spring에서 @Transactional 이용 시 기본 예외 전략**



Spring에서 @Transactional은 UnCheckedException에 대해서 예외가 발생하면 자동적으로 RollBack이 되게끔 구성되어있다



한 예로 Spring의 data접근과 관련되어 발생하게 되는 예외(예를 들어 SqlException)는 내부적으로 UnCheckedException 계열(DataAccessException)으로 전환해서 다시 예외를 내던지는 방식으로 작성이 되어있고 이는 Spring의 기본적 예외처리 관련된 철학의 일환이라고 볼 수 있다. 



이는 UnCheckedException의 경우에 프로그래밍적 오류나 논리적으로 복구 불가능한 오류 혹은 개발자가 처리할 수 없는 것 등에 해당되는 경우가 많기에 롤백되게끔 스프링에서 설계를 해뒀기 때문이다.



물론 해당 옵션은 기본값이지만, @Transactional의 속성 값으로 ‘rollBackFor’라는 값이나 ‘noRollbackFor’에 특정 예외를 설정해줌으로써 해당 예외에만 롤백을 적용하거나 적용하지 않는 방식으로 커스텀하게 작성할 수도 있다. 