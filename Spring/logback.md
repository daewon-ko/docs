# SLF4J와 Logback

**Slf4란?**

*Simple Loggging Facade For Java*



- 다양한 로깅 프레임워크에 대한 추상화(인터페이스) 역할
  - SLF4j는 추상 로깅 프레임워크이므로 단독으로 사용하지 않는다.
  - SLF4J를 의존하는 클라이언트 코드는 실제 구현을 몰라도 된다.
  - (DIP)
- 
- Logback은 Slf4j의 구현체이며 Log4J를 토대로 만든 프레임워크이다.
- Slf4J의 구현체로 Log4J, Logback이 존재한다.
- 스프링부트에는 내장으로 log4j, loback 모두 존재한다.



SL4J는 3가지 모듈을 제공한다.

- Bridging Module
  - SLF4J 이외의 다른 로깅 API로의 Logger 호출을 SLF4J 인터페이스로 Redirect 한다.
  - 레거시 로깅 프레임워크를 위한 라이브러리
- SLFJ API
  - 로깅에 대한 추상화 계층엘 제공하는 인터페이스 집합
  - SLF4J는 Logger, LogerFactory 인터페이스를 통해 로깅 메시지 기록
  - Logback, Log4j와 같은 특정 로깅 구현체에 의존하지 않는다.
- SLF4J Binding(.jar)
  - SLF4J 인터페이스를 로깅 구현체와 연결하는 어댑터 역할을 수행
  - SLF4J API를 구현한 클래스에서 Binding으로 연결될 Logger API를 호출



- 스프링부트는 기본적으로 SLF4J, Logback을 채택하고 있다.





**Logback이란?**

- Log4J의 후속버전인 라이브러리
- Logging 라이브러리로는 Log4J, Logback, Log4J2를 보통 사용하며 개발순서도 해당순서와 동일하다.



**Logback의 구조**

- logback-core
  - 나머지 두 모듈을 위한 기반 역할을 수행
  - Appender, Layout(Encoder) 인터페이스가 속한다.
- logback-classic
  - logback-core에서 확장된 모듈
  - Logger 클래스가 속한다.
  - logback-core / SLF4J API 라이브러리를 가지고 있음
    - 전이적 의존관계로 가지고 온다.

![image](https://github.com/user-attachments/assets/445e9ad4-cf67-4bba-9da1-a09205ace2aa)

- logback-access
  - Servlet Container와 통합되어 HTTP 액세스에 대한 로깅 기능 제공
  - 웹 애플리케이션 레벨이 아닌 컨테이너 레벨에 설치
  - 따라서 클래스 패스에 포함되는 라이브러리가 아니다.

 



LogBack의 구체적 구성요소를 살펴보자.



**Logger**

- 실제 로깅을 수행하는 구성요소
- logback.classic 내 위치
- Level 속성을 통해 출력 로그를 지정가능하다.



import org.slf4j.Logger;

import org.slf4j.LoggerFactory;



public class WebApplication {

  public static void main(String[] args) {

​    Logger logger = LoggerFactory.getLogger(WebApplication.class);



​    for (int count = 1; count <= 10; count++) {

​      logger.trace("trace 로깅 {}", count);

​      logger.debug("debug 로깅 {}", count);

​      logger.info("info 로깅 {}", count);

​      logger.warn("warn 로깅 {}", count);

​      logger.error("error 로깅 {}", count);

​    }

  }

}







**Appender**

- 로그 이벤트를 Write하는 객체
- Logback은 Appender에 로그 이벤트 작성을 위임한다.
- 많은 라이브러리들이 그러하듯 Appender 역시 추상화 되어있다.

![image 2](https://github.com/user-attachments/assets/5c7d2b9e-1b44-41d4-bda2-5c75f1c4b2d9)



간략하게 각 컴포넌트에 대해 설명하자면



- UnsynchronizedAppenderBase
  - 비동기적 동작이 필요할때 사용
  - CF) AppenderBase는 Synchronized하다.
- OutputStreamAppender
  - java.io.OutputStream 객체에 로그 이벤트를 Append
- ConsoleAppender
  - System.out / System.err를 활용해 로그 이벤트를 Append
- FileAppender
  - 파일에 로그를 append한다.
- RollingFileAppender
  - FileAppender를 상속해서 로그 파일을 rollover한다.
    - rollover란 타겟 파일을 바꾸는 것을 의미



**Rolling Policies**

- TimeBasedRollingPolicy
  - 특정 시간을 기준으로 rollover를 수행
  - 보통 일/월 단위 수행
  - rollover 뿐 아니라 Trigger 역할 역시 수행
  - 필수적으로 fileNamePattern 속성을 가진다.
    - 파일 쓰기가 종료된 이후 log 파일명의 패턴을 지정한다.
  - 아래는 예시



<configuration>

 <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">

  <file>logFile.log</file>

  <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">

   <!-- 하루 동안의 log를 남김 -->

   <fileNamePattern>logFile.%d{yyyy-MM-dd}.log</fileNamePattern>



   <!-- 30일동안, 총 최대 3GB의 log를 저장함-->

   <maxHistory>30</maxHistory>

   <totalSizeCap>3GB</totalSizeCap>



  </rollingPolicy>



  <encoder>

   <pattern>%-4relative [%thread] %-5level %logger{35} - %msg%n</pattern>

  </encoder>

 </appender> 



 <root level="DEBUG">

  <appender-ref ref="FILE" />

 </root>

</configuration>

 



- SizeAndTimeBasedRollingPolicy
  - TimeBasedRollingPolicy에서 각 로그 파일에 대한 크기를 제한하는 부분이 추가
  - fileNamePattern에서 %i, %d가 필수적
  - 예씨



 <configuration>

 <appender name="ROLLING" class="ch.qos.logback.core.rolling.RollingFileAppender">

  <file>mylog.txt</file>

  <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">

   <!-- 하루 동안의 log를 남김 -->

   <fileNamePattern>mylog-%d{yyyy-MM-dd}.%i.zip</fileNamePattern>

   <!-- 각각의 파일은 100MB로 저장되고, 30일동안, 최대 3GB의 log를 저장함-->

​    <maxFileSize>100MB</maxFileSize>   

​    <maxHistory>30</maxHistory>

​    <totalSizeCap>3GB</totalSizeCap>

  </rollingPolicy>

  <encoder>

   <pattern>%msg%n</pattern>

  </encoder>

 </appender>



 <root level="DEBUG">

  <appender-ref ref="ROLLING" />

 </root>



</configuration>









**Encoder**

![image 3](https://github.com/user-attachments/assets/872b72bc-e9d6-4f74-bd7e-171a792ad47a)



- 로그 이벤트를 바이트 배열로 변환 후 OutputStream에 쓰는 작업 수행
- FileAppender는 Layout이 아닌 Encoder 객체를 필요로 한다.





**Log Level 순서**

*TRACE → DEBUG → INFO → WARN → ERROR → FATAL*









참고

https://blog.pium.life/server-logging/

[[Logging\] slf4j, log4j, logback, log4j2](https://minkwon4.tistory.com/161)

[Logback 이란?](https://agileryuhaeul.tistory.com/entry/Logback-이란)

[Logback 으로 쉽고 편리하게 로그 관리를 해볼까요? ⚙️](https://tecoble.techcourse.co.kr/post/2021-08-07-logback-tutorial/)

https://www.youtube.com/watch?v=JqZzy7RyudI

[[Logging\] Logback이란?](https://livenow14.tistory.com/64)