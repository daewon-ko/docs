# 스프링이 지원하는 동적 프록시

* JDK Dynamic Proxy
  * JDK 기본 지원
  * 인터페이스 지원
  * InvocationHandler 구현

* CGLIB(Enhancer~)
  * 스프링? 서블릿 지원
  * 구체클래스 지원
  * MethodIntercepor 구현
* 구체적기술을 스프링이 통합
  * 즉, 어쩔때는 CGLIB을 구현하고 어쩔때는 JDK Dynamic Proxy를 구현하긴하지만, 
    ProxyFactory로 통합하여 구현

### ProxyFactory

* 개발자는 MethodInterceptor(spring.aop. 내 모듈 / CGLIB과 다르 패키지이다.)를 구현하는 Advice를 작성

  * Advice는 부가기능
  * pointcut
    * 어디에 부가기능을 적용할지 안 할지를 판단하는 필터링 로직
    * 클래스와 메서드 이름으로 필터링
  * Advisor
    * Advice + Pointcut

  ```java
  public class TimeAdvice implements MethodInterceptor {
      @Override
      public Object invoke(final MethodInvocation invocation) throws Throwable {
          log.info("TimeProxy 실행");
          long startTime = System.currentTimeMillis();
          Object result = invocation.proceed();
          long endTime = System.currentTimeMillis();
          long resultTime = endTime - startTime;
          log.info("TimeProxy 종료 resultTime={}ms", resultTime);
          return result;
      }
  }
  ```

* invocation.proceed()를 통해 실제 메서드 호출
* proxyFactory의 setProxyTargetClass(true); 설정 시, 인터페이스 기반이어도, 
  강제로 CGLIB기반 동적 프록시 생성 가능 
  (스프링은 기본 셋팅값이 true이다. )
* Proxy에 여러 어드바이저를 추가한다고 해서 Proxy가 계속 생성되는 것은 아니다.
  * 즉, 여러 부가기능(Advice)를 붙여야한다고 해서 Proxy를 해당 개수만큼 생성하는 것이 절대 아님!
* target객체에 해당하는 Proxy를 생성하고 여러 Advice를 추가할 수 있다. 

```java
@Test
@DisplayName("하나의 프록시, 여러 어드바이저") void multiAdvisorTest2() {
     //proxy -> advisor2 -> advisor1 -> target
     DefaultPointcutAdvisor advisor2 = new DefaultPointcutAdvisor(Pointcut.TRUE,
 new Advice2());
     DefaultPointcutAdvisor advisor1 = new DefaultPointcutAdvisor(Pointcut.TRUE,
 new Advice1());
     ServiceInterface target = new ServiceImpl();
     ProxyFactory proxyFactory1 = new ProxyFactory(target);
     proxyFactory1.addAdvisor(advisor2);
     proxyFactory1.addAdvisor(advisor1);
     ServiceInterface proxy = (ServiceInterface) proxyFactory1.getProxy();
//실행
     proxy.save();
}
```

실행결과

> MultiAdvisorTest$Advice2 - advice2 호출
> MultiAdvisorTest$Advice1 - advice1 호출
> ServiceImpl - save 호출

이와 같이 하나의 프록시에 다른 Advice를 적용가능

즉, 하나의 target에 여러 AOP가 적용해도, 스프링의 AOP는 target마다 하나의 프록시만을 생성한다. 

### PointCut

* 스프링은 대부분의 PointCut을 구현해놓음
  * NamedMatchMethodPointCut
  * JDK~..
  * TruePointPointCut
  * AspectJExpressionPointCut : 사실상 가장 많이 쓰임



### 프록시 팩토리의 문제

1. @Confinguration의 Bean마다 AOP를 적용해줘야함. 
   - Configuration 폴더가 여러개라면? 
   - 수동 등록하는 Bean이 매우 많다면 각각 Bean마다 ProxyFactory 생성코드를 작성해줘야한다.
2. Component Scan을 사용하면 프록시 적용이 불가
   - @Service, @Controller, @Repository와 같은 일반적으로 스프링부트 프로젝트
     이용시 자주사용하는 Annotation에 프록시 생성이 불가능하다. 



