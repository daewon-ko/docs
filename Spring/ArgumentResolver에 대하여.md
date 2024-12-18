# ArgumentResolver



### ArgumentResolver란?

스프링 프로젝트를 만들면서 WAS의 Endpoint인 Presentation Layer를 담당하는 Controller는 많은 매개변수를 처리한다. 



레거시 버전이 아닌 대부분의 스프링 (스프링부트) 프로젝트에서 사용하는 Controller는 @RequestMapping을 사용하는데, DispatcherServlet은 HandlerAdaper를 통해서 적절한 HandlerAdapter를 조회를 한다.



따라서 전술한 이유로, 스프링에서 HandlerAdapter의 구현체로는 RequestMappingHandlerAdapter를 대부분 사용하게 된다.

(하단 그림 참조)





![스크린샷 2024-12-17 오전 10 33 09](https://github.com/user-attachments/assets/420840b3-9471-4c64-beb6-3574800be54e)



![image](https://github.com/user-attachments/assets/173f94c6-e2d4-4474-9e18-96bd7641b34c)

그림 참고 : 인프런 김영한 MVC1



사설이 길었다.

그렇다면 ArgumentResolver란 무엇인가? 



우리가 Controller의 메서드의 매개변수를 작성할때는 잘 생각하지 못하지만, 곰곰히 살펴보면 많은 Type이 매개변수로 들어간다. 



HttpServletRequest, HttpServletResponse, Model, @RequestParam, @ModelAttribute, @RequestBody 등..



매개변수의 다양한 타입의 객체들은 Spring은 어떻게 처리할까?에 대한 대답이 바로 (Handler)ArgumentResolver이다.



HandlerAdapter는 HandlerMethodArgumentResolver를 호출하는데, 스프링의 많은 컴포넌트들의 의존관계가 그러하듯 ArgumentResolver역시 ‘전략패턴’으로 구축이 되어있다.





![image 2](https://github.com/user-attachments/assets/df1ea017-f3d5-4cfe-bced-c1763e20a20e)





HandlerMethodArgumentResolver의 모습은 위와 같고, HandlerAdapter와 같이 인터페이스를 구현하는 여러 구현체의 형식으로 구성되어있다.



ArgumentResolver가 처리할 수 있는 매개변수 타입은 굉장히 다양한데 지원 타입은 다음의 링크를 참고하면 된다. 



[Method Arguments :: Spring Framework](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-methods/arguments.html)



### 그렇다면 ArgumentResolver는 어디에 어떻게 이용될 수 있을까?



Session을 통해서 로그인을 한다고 가정하자. 

(본인의 토이프로젝트를 기준으로 설명하겠다.)

약간의 부연설명을 하자면, 하단의 CustomFilter는 특정 URL(‘api/v1/auth/login’)의 경로에서 작동하는데, 인증을 시도하고, 인증이 성공하면 세션에 로그인 정보(email)를 저장한다.



```java
package shoppingmall.web.filter.session;



import com.fasterxml.jackson.databind.ObjectMapper;

import jakarta.servlet.FilterChain;

import jakarta.servlet.ServletException;

import jakarta.servlet.http.HttpServletRequest;

import jakarta.servlet.http.HttpServletResponse;

import org.springframework.security.authentication.AuthenticationManager;

import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;

import org.springframework.security.core.Authentication;

import org.springframework.security.core.AuthenticationException;

import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

import shoppingmall.common.exception.ApiException;

import shoppingmall.common.exception.domain.AuthErrorCode;

import shoppingmall.core.domain.user.dto.LoginUserRequestDto;



import java.io.IOException;



public class CustomLoginFilter extends UsernamePasswordAuthenticationFilter {

  private final ObjectMapper objectMapper;



  public CustomLoginFilter(AuthenticationManager authenticationManager, final ObjectMapper objectMapper) {
    super.setFilterProcessesUrl("/api/v1/auth/login");

	 this.setAuthenticationManager(authenticationManager);

		this.objectMapper = objectMapper;

  }





  @Override

  public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {

	   try {

     LoginUserRequestDto loginUserRequestDto = objectMapper.readValue(request.getInputStream(), LoginUserRequestDto.class);

		UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(loginUserRequestDto.getEmail(), loginUserRequestDto.getPassword());



	return this.getAuthenticationManager().authenticate(authRequest);



	} catch (IOException e) {

throw new ApiException(AuthErrorCode.INVALID_LOGIN_REQUEST);
	
		}

  }





  @Override

  protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain, Authentication authResult) throws IOException, ServletException {

request.getSession().setAttribute(SessionConst.LOGIN_USER, authResult.getPrincipal());

response.getWriter().write("Login Success");

response.getWriter().write(objectMapper.writeValueAsString(authResult.getName()));

  }



  @Override

  protected void unsuccessfulAuthentication(HttpServletRequest request, HttpServletResponse response, AuthenticationException failed) throws IOException, ServletException {

response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);

response.setContentType("application/json; charset=utf-8");

response.getWriter().write(objectMapper.writeValueAsString(failed.getMessage()));

  }

}
```



상기의 Filter는 Spring Security를 이용하여 Filter를 커스텀한 클래스이다. 

succesfulAuthentication 메서드 내부에서 인증이 만약 성공하게되면, session 객체에서 ‘SessionConst.Login_MEMBER’라는 이름으로 authResult.getPrincipal()객체를 넣어놓는다.



authResult.gerPrincipal()객체는 로그인 요청시 RequestDTO에서 정의한 바에 의하면 email이다.

따라서 만약 로그인이 성공하게 된다면, Session에 해당 email이 저장될 것이다. 

(물론 식별자 이름은 다르겠다만.)





```java
package shoppingmall.web.config.argumentResolver;

import hello.login.domain.member.Member;
import hello.login.web.SessionConst;
import lombok.extern.slf4j.Slf4j;
import org.springframework.core.MethodParameter;
import org.springframework.web.bind.support.WebDataBinderFactory;
import org.springframework.web.context.request.NativeWebRequest;
import org.springframework.web.method.support.HandlerMethodArgumentResolver;
import org.springframework.web.method.support.ModelAndViewContainer;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpSession;

@Slf4j
public class LoginMemberArgumentResolver implements HandlerMethodArgumentResolver {

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        log.info("supportsParameter 실행");

        boolean hasLoginAnnotation = parameter.hasParameterAnnotation(Login.class);
        boolean hasStringType = String.class.isAssignableFrom(parameter.getParameterType());

        return hasLoginAnnotation && hasStringType;
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {

        log.info("resolveArgument 실행");

        HttpServletRequest request = (HttpServletRequest) webRequest.getNativeRequest();
        HttpSession session = request.getSession(false);
        if (session == null) {
			throw new IllegalStateException("로그인이 필요한 요청입니다.");
        }
			


		 Object sessionObj =session.getAttribute(SessionConst.LOGIN_USER);
		if(sessionObj==null){
			throw new IllegalStateException("로그인된 사용자가 아닙니다.");
			// 로그인 페이지로 Redirection?
		}
    }
}

```





위와 같이 로그인과 관련된 LoginArgumentResolver를 작성할 수 있다. 

컨트롤러의 매개변수에 @Login 어노테이션이 존재하고 해당 어노테이션의 지원형식이 String인지 여부를 supportPameter에서 확인한다. 

supportParameter를 통과하면 resolveArgument 메서드를 호출한다. 

참고로 Login 어노테이션은 다음과 같이 지정할 수 있다. 



```java
@Target(ElementType.PARAMETER)

@Retention(RetentionPolicy.RUNTIME)

public @interface Login {

}
```





해당 어노테이션은 런타임시에 파라미터에 적용하는 어노테이션이라는 의미이다.

선언한 Login 어노테이션을 활용하는 방법은 아래와 같다. 



```java
@PostMapping("/product/order")
public ApiResponse<String> orderProduct(@RequestBody @Valid ProductOrderRequestDto requestDto,@Login String email) {
    // 주문 로직 처리
    log.info("Order request by user: {}", email);

    productService.orderProduct(requestDto, email);

    return ApiResponse.of(HttpStatus.OK, "Order placed successfully");
}
```



Session 객체에 email을 저장했다고 가정하면, 위에서 정의한 LoginArgumentResolver가 작동해서 각각의 메서드를 실행하고 판별한다. 



해당 경로의 POST요청을 실행할때 정의한 requestDto만 body에 넣어주면, String email과 관련한 부분에 대해서는 별도의 처리를 하지 않아도 작성했던 LoginArgumentResolver가 자동으로 처리해준다. 











참고

[인프런 김영한 MVC 2](https://www.inflearn.com/course/스프링-mvc-2/dashboard)