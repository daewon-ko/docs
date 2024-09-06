# @PostConstructor와 @EventListener의 차이



### 배경

> `@EventListener(ApplicationReadyEvent.class)` : 스프링 컨테이너가 완전히 초기화를 다 끝내고,
>
> 실행 준비가 되었을 때 발생하는 이벤트이다. 스프링이 이 시점에 해당 애노테이션이 붙은 `initData()` 메서드 를 호출해준다.
>
> 참고로 이 기능 대신 `@PostConstruct` 를 사용할 경우 AOP 같은 부분이 아직 다 처리되지 않은 시점에 호출될 수 있기 때문에, 간혹 문제가 발생할 수 있다. 예를 들어서 `@Transactional` 과 관련된 AOP가 적 용되지 않은 상태로 호출될 수 있다.
>
> `@EventListener(ApplicationReadyEvent.class)` 는 AOP를 포함한 스프링 컨테이너가 완전 히 초기화 된 이후에 호출되기 때문에 이런 문제가 발생하지 않는다.

참고 : 김영한 스프링 DB2

스프링부트에서 서버를 띄우면서 간단한 초기 데이터 설정 시에 통상 @PostConstructor를 통해서 초기화 한다. 그런데 위와 같은 내용을 학습하면서 마주했다. 



@EventListner는 @PostConstructor와 무엇이 다르고 또한 @PostConstructor는 AOP와 무슨 관계가 있을까?



### @PostConstructor의 적용시점

일반적으로 @PostConstructor는 빈이 초기화 된 이후에 적용되는 어노테이션이다. 그런데, @Transactional과 같은 스프링 AOP를 이용할 경우에는 프록시 객체로 빈을 초기화하게된다. 즉 적용시점에서 있어서 둘 간의 간극이 존재하는 것이다. 



@PostConstuctor의 경우 모든 빈(스프링 AOP를 이용하여 프록시로 등록되는 빈을 포함)들이 초기화 된 이후에 적용되는 메서드이다. 그렇기에 만약 @Transactional과 같은 어노테이션과 @Postconstructor가 함께 쓰인다면 @Transacitonal은 적용이 불가능하다. 



### @EventListener란? 

빈을 초기화한 직후에 바로 트랜잭션 처리를 자동으로 해야하는 로직 등이 있을 때 사용된다. 어노테이션 이름에서 유추할 수 있듯이 특정 이벤트가 발생하면 자동으로 로직이 수행되게끔 구성되는데, Option을 줄 수 있다. 위와 같이 구체적으로 스프링 컨테이너가 뜨자마자 바로 트랜잭션처리를 해줘야한다면 아래와 같이 사용될 수 있다. 

~~~java
    @EventListener(value = ApplicationReadyEvent.class)
         @Transactional
         public void init2() {
         // 트랜잭션을 보장하는 특정 로직을 자동수행하게끔 하는 로직을 작성할 수 있다.
					logic();
         }
}
~~~













