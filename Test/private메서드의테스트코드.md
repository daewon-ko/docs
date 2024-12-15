# private 메서드의 테스트 코드
#docs/Test

* Private 메서드의 테스트 코드를 작성해야할까? 
* 작성하는 것을 지양해야한다고 어렴풋 알고 있다.
  * 그 이유는 무엇일까?
* 그럼에도 private 메서드를 테스트 하는 방법

### Java Reflection API를 이용한 메서드 호출
* 클래스, 메서드 정보를 비롯한 클래스의 변수, 메서드의 파라미터 타입, 메서드 이름 등에 관한 메타 정보는 자바에서 메타정보로 추상화하여 ‘Method’로 관리한다.
  * 해당 기능은 자바가 Reflection API를 이용하기 때문에 이용가능하다


```java
package io.springbatch.privatetest;

import org.assertj.core.api.Assertions;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.mockito.InjectMocks;
import org.springframework.boot.test.context.SpringBootTest;

import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
class PrivateTestClassTest {

    @InjectMocks
    private PrivateTestClass target;


    @DisplayName("")
    @Test
    void test() throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {
        //given

        String name = "Hello";
        Method method = target.getClass().getDeclaredMethod("isPredefiened", String.class);

        method.setAccessible(true);
        //when
        boolean result =(boolean) method.invoke(target, name);

        //then
        Assertions.assertThat(result).isEqualTo(true);

    }


}

```


### Spring의 Reflection TestUtils를 이용한 메서드 호출

* 형태만 다르지, 스프링에서 제공하는 편의 기능을 이용한다.
```java

@SpringBootTest
class PrivateTestClassTest {

    @InjectMocks
    private PrivateTestClass target;


    @DisplayName("")
    @Test
    void test() throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {
        //given

        String name = "Hello";
        Method method = target.getClass().getDeclaredMethod("isPredefiened", String.class);
        method.setAccessible(true);

        ReflectionUtils.invokeMethod(method, target);

        //when
        boolean result =(boolean) method.invoke(target, name);

        //then
        Assertions.assertThat(result).isEqualTo(true);

    }


}

```


### 그럼에도 불구하고 Private 메서드의 테스트 코드를 해야하는가?
* 일단 상기의 두 방식(사실 Reflection API를 이용한다는 측면에선 거의 동일하지만)의 경우 기본적으로 런타임에 동작하는 기술이다. 
* 따라서 컴파일 타임에는 테스트 코드를 체크할 수가 없다. 
* 위와 같이 private 메서드를 테스트 하게되면, private 메서드의 내용이 변경하게 되면 클라이언트(테스트 코드)가 깨지게 된다. 
* 이는 테스트에 대한 유지보수 비용을 증가시키며, 당연히 클라이언트(테스트 코드)와 결합도도 높이게 되는 원인이 된다. 


따라서 가장 권장되는 방식은, private 메서드를 직접 테스트하기 보다는 접근제어자가 public인 메서드를 가장 우선적으로 테스트하되 성공하는 테스트 케이스와 실패하는 테스트 케이스를 나눠서 private 메서드를 커버해야한다. 

가령, @ParameterizedTest를 이용하게 되면 동일한 테스트를 여러 개의 파라미터로 실행 가능하다.

[\[JUnit5\] @ParameterizedTest로 한 번에 테스트하자](https://velog.io/@ohzzi/junit5-parameterizedtest)

### 클래스를 분리하라

그럼에도 분리하고 private 메서드 자체를 직접 테스트해야하는 상황이라면 해당 메서드의 기능이 너무 커진 것 아닌가? 혹은 클래스를 분리해야하는가를 고민해야하는 시점이라고 할 수 있겠다. 

즉 동일한 클래스에서 public 메서드에서 private 메서드를 호출하는 것이 아닌, 해당 private 메서드를 새로운 클래스를 작성하여 의존관계를 주입하여 작성하는 것이 객체지향의 관점에서는 더 올바르다고 할 수 있겠다. 




참고
https://mangkyu.tistory.com/235
[인프런 Practical Testing 강의](https://www.inflearn.com/course/practical-testing-%EC%8B%A4%EC%9A%A9%EC%A0%81%EC%9D%B8-%ED%85%8C%EC%8A%A4%ED%8A%B8-%EA%B0%80%EC%9D%B4%EB%93%9C/dashboard)
