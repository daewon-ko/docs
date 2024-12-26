**Spring에서 예외처리 방법**

\#docs/spring/



**배경**

테스트 코드를 작성하던 도중, 실패하는 테스트 코드를 다음과 같이 작성했다.

Mockito를 이용하여 Presentation Layer를 테스트하는 내용이었다.



어떠한 UUID값과 Session 객체가 넘어오면, ChatErrorCode의 INVALID_CHATROOM_NUMBER를 Throw 해줘야했다.



![image](https://github.com/user-attachments/assets/ae003ba0-81f2-4484-b534-b6a30e4291f2)



그러나 테스트 코드를 계속 살펴봐도 Assertions의 값이 200이 나왔다. 

왜 그럴까? 





**Spring MVC의 Exception 처리 방식**



Spring을 이용하여 커스텀한 예외를 정의한 후 해당 Exception을 전역적으로 처리해주기 위해선 @ControllerAdvice 또는 @RestControllerAdvice 클래스를 작성해준다. 



해당 어노테이션이 붙은 클래스 내에는 특정 Exception을 Handling해주는 메서드들을 다음과 같이 정의해준다. 



![image 2](https://github.com/user-attachments/assets/0560104f-8d91-4369-a1f1-dc9079faa429)



상기의 두 메서드는 본인이 커스텀하게 정의한 ApiResponse의 of라는 정적팩토리 메서드를 이용해서 객체를 리턴해주고 있다. 또한 ApiResponse객체는 매개변수로 HttpStatus Code와 관련된 메시지를 받고있다.



해당의 두 메서드는 @ExceptionHandler라는 어노테이션 이외에도 @ResposneStatus라는 어노테이션을 붙여주고 있다. @ResposneStatus 어노테이션의 옵션으로 HttpStatus.BAD_REQUEST를 설정값으로 주고 있는데, ApiResponse 객체의 of메서드에서 HttpStatus.BAD_REQUEST라는 값을 주고 있음에도 중복으로 해당 옵션을 줘야하는 이유는 무엇일까? 



**Spring의 기본 예외 처리 방식**

상기의 의문에 답하기 앞서 Spring 자체의 예외 처리 방식에 대해서 알아야 한다. 일반적인 Web EndPoint가 있다고 하고, 해당 EndPoint 내부에서 어떠한 예외가 터졌다고 가정해보자. Spring은 어떻게 예외를 처리할까? 



Spring 내부에서 예외가 터졌다고 가정하면 다음과 같은 순서로 이루어진다.



*WAS → 서블릿 필터 → 프론트컨트롤러(디스패처 서블릿) → 인터셉터 → 컨트롤러*



컨트롤러 하위에서 예외 발생시, 예외처리를 하지 않을 경우, WAS까지 전파되는데, WAS는 여기서 Spring 내부에 ‘/error’ 경로로 이미 구현해 놓은 BasicErrorController를 호출한다.

*컨트롤러(예외 발생) → 인터셉터 → 프론트 컨트롤러 → 서블릿필터 → WAS → 필터 → 프론트컨트롤러 → 인터셉터 → BasicController*



**HandlerExceptionResolver**

<img width="1029" alt="image 3" src="https://github.com/user-attachments/assets/b29dea8a-080e-4248-b6a8-5d464e9db357" />



Spring은 예외 발생시 처리 방식을 공통화 하기 위해HandlerExceptionResolver라는 인터페이스를 활용한다. 스프링 자체가 수 많은 디자인패턴의 총합이기에, HandlerExceptionResolver를 구현체들이 있

고 우선순위는 다음과 같다.





- DefaultErrorAttributes
  - 예외를 직접 처리하진 않고 속성만 관리한다.
  - Spring 내 BasicErrorController에서 해당 속성을 이용한다,.
- ExceptionHandlerExceptionResolver
  - Controller, ControllerAdvice에 있는 ExceptionHandler를 처리
- ResponseStatusExceptionResolver
  - @ResponseStatus / ResposneStatusException 처리
- DefaultHandlerExceptionResolver
  - 스프링 내부 기본적 예외 처리



**@ResponseStatus, ResponseStatusException**

- HttpStatusCode를 조작할 수 있다.
- 두 방식 모두 ResponseStatusExceptionResolver가 처리한다.
- 해당 방식의 가장 큰 문제는 WAS까지 예외를 전파시키게 되고 BasicErrorController가 해당 예외를 처리하게 된다.
- 이에 따라 클라이언트에게 세밀한 응답을 내줄 수 없다.



상기의 이유 및 예외처리 코드의 일관성을 위해 전역적으로 예외처리가 가능한 @RestControllerAdvice, @ControllerAdvice, @ExceptionHandler 등이 결합되어 많이 사용된다.



**@ExceptionHandler**

- @ExceptionHandler는 속성 값으로 Exception 클래스를 지정하여 처리할 예외를 설정할 수 있다.
- @ResponseStatus와 결합 가능하다.
  - 해당 메서드가 ResponseEntity로 반환한다면, @ResponseStatus보다 ResponseEntity가 더 우선순위가 높다.
- Spring에서 예외처리 시, 구체적 예외처리 핸들러를 찾고, 해당 핸들러가 없으면 부모 클래스를 속성으로 갖는 핸들러를 찾는다.



**@ControllerAdivice, @RestControllerAdvice**

- 전역적으로 에러를 처리할 수 있다.
- 클래스 상단에 해당 어노테이션을 선언하고, 둘 간의 차이는 Controller - RestController처럼 @ResponseBody 어노테이션의 유무 차이이다.(즉 Json으로 값을 응답하느냐 유무이다.)
- 









[[Spring\] 스프링의 다양한 예외 처리 방법(ExceptionHandler, ControllerAdvice 등) 완벽하게 이해하기 - (1/2)](https://mangkyu.tistory.com/204)





