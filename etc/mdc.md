# MDC 이용를 이용한 HTTP 로깅(1)

#docs/logging


### 배경
토이 프로젝트를 진행하면서 HTTP 요청과 응답에 대해서 전반적인 로깅을 할 필요성을 느꼈다. 
이전에 Logback에 관해서 학습을 하기도 했고, 간단하게 해당 Logback 등을 적용해서 전반적인 HTTP 요청과 응답에 대해서 로깅을 적절한 방식으로 남겨보고 싶었다. 

Logback과 잘 통합되어 사용하는 것들 중에 MDC라는 키워드를 알게되어서 학습하게 되었다. 


### MDC(Mapped Diagnostic Context)

* 스레드 별로 로그를 모아서 볼 수 있다.
* 통상 멀티스레드 환경에서 각 스레드의 실행 컨텍스트를 구분하는데 용이하다.
* 이름에서 알 수 있듯이 어떤 ‘컨텍스트’ 전체를 파악할 수 있다.

그렇다면 MDC는 쓰레드 별로 로그를 어떻게 모아서 볼 수 있을까? 


### MDCAdapter
Mdc 코드를 내부적으로 살펴보면 다음과 같다. 
통상 로깅을 적용할때 MDC.put메서드를 호출하는데 mdc는 내부의 mdcAdapter에 이를 위임한다.

Logback과 MDC는 통합되어 자주 사용되기에 보통은 LogbackAdapter가 구현체로 사용된다. 해당 구현체의 코드를 보면 다음과 같다. 


```java
package ch.qos.logback.classic.util;

import java.util.Collections;
import java.util.Deque;
import java.util.HashMap;
import java.util.Map;
import java.util.Set;
import org.slf4j.helpers.ThreadLocalMapOfStacks;
import org.slf4j.spi.MDCAdapter;



public class LogbackMDCAdapter implements MDCAdapter {
    final ThreadLocal<Map<String, String>> readWriteThreadLocalMap = new ThreadLocal();
    final ThreadLocal<Map<String, String>> readOnlyThreadLocalMap = new ThreadLocal();
    private final ThreadLocalMapOfStacks threadLocalMapOfDeques = new ThreadLocalMapOfStacks();

    public LogbackMDCAdapter() {
    }

    public void put(String key, String val) throws IllegalArgumentException {
        if (key == null) {
            throw new IllegalArgumentException("key cannot be null");
        } else {
            Map<String, String> current = (Map)this.readWriteThreadLocalMap.get();
            if (current == null) {
                current = new HashMap();
                this.readWriteThreadLocalMap.set(current);
            }

            ((Map)current).put(key, val);
            this.nullifyReadOnlyThreadLocalMap();
        }
    }


...(이하 생략)
```


LogbackMdcAdapter 역시 내부적으로는 ThreadLocal을 사용하고 있음을 알 수 있다.

그렇다면 ThreadLocal은 무엇일까?

### ThreadLocal

ThreadLocal은 각 쓰레드마다 별도로 가지고 있는 내부의 저장소 같은 개념이라고 생각하면 된다. 
따라서 동일한 인스턴스의 필드변수라고 하더라도 다른 쓰레드가 쓰레드로컬로 선언된 변수에 접근하여 데이터를 변경한다고 할지라도 동시성 문제에서 자유롭다. 

ThreadLocal은 java.lang 패키지 내에 위치하므로 자바 언어 차원에서 지원하는 기술이다.
그런데 ThreadLocal은 아주 중요한 특성을 가지고 있는데 사용하고 난 ThreadLocal은 반드시 remove를 해줘야 한다는 것이다.

왜그럴까? 


이것은 WAS가 멀티쓰레드 환경에서 쓰레드 풀을 이용하기 때문에 발생하기 때문이다. 

### Thread-Pool

(거의 모든) 대부분의 WAS는 멀티쓰레드 환경을 지원한다. WAS는 멀티쓰레드 환경에서 동시성 작업을 처리하기 위해서 쓰레드풀을 통해서 멀티 쓰레드를 지원하는데, 해당 과정은 아래와 같은 형식이다.

1. 사용자 A가 Http 요청을 한다.(프로토콜 자체는 중요하지 않고 ‘요청’을 했다는 것이 중요하다.)
2. WAS는 쓰레드 풀에서 쓰레드를 하나 조회한다.
3. 쓰레드는 Thread-A가 할당된다.


문제는 위의 상황에서 ThreadLocal을 Thread가 이용하게 될때인데, ThreadLocal을 이용하게 되면 아래와 같은 흐름의 순서를 갖게된다.


1. 사용자 A가 Http 요청을 한다.(프로토콜 자체는 중요하지 않고 ‘요청’을 했다는 것이 중요하다.)
2. WAS는 쓰레드 풀에서 쓰레드를 하나 조회한다.
3. 쓰레드는 Thread-A가 할당된다.
4. (ThreadLocal을 이용한다면) Thread-A는 사용자 A의 데이터를 쓰레드 로컬에 저장한다.
5. 요청된 작업을 수행 후 종료되면 Thread A가 쓰레드 풀에 반납된다.
   - 쓰레드 생성비용은 적지 않기때문에 제거되지 않고 반납되어 다시 재사용된다.
   - Thread A는 쓰레드 풀에 반납되었지만, ThreadLocal 자체는 계속해서 남아있게 된다.
   - 따라서 Thread가 반납되기 전에 **반드시 명시적으로 ThreadLocal을 삭제해줘야한다.**

![](image%202.png)
> 참고 : 쓰레드로컬이 사용되는 일반적 원리



만약 쓰레드를 이용한 이후에 쓰레드풀에 반납하기 전에 ThreadLocal을 삭제하지 않으면, 다른 사용자의 요청에 해당 쓰레드가 재활용하게 되면 다른 데이터를 읽거나 쓰게되는 아주 심각한 버그가 발생할 수 있게된다. 

따라서 ThreadLocal을 사용한 이후엔 반드시 remove 메서드 등을 호출하여 삭제해주는 것이 꼭 필요하다.

![](image.png)


> 참고: 쓰레드로컬 사용시, 문제상황을 가정한 그림



### MDC 이용시 주의사항

처음으로 다시 돌아가보자. 
MDC는 내부적으로 ThreadLocal을 사용한다고 했다. 

그렇다면 MDC를 사용한다면 무엇을 주의해야할까? 

ThreadLocal 사용시 주의점인 ThreadLocal을 사용 이후엔 반드시 remove()메서드를 호출해야하는 것처럼, 
MDC를 사용한 이후엔 MDC를 삭제해주는 메서드를 호출해줘야한다.

실제로 MDC의 실제 remove()메서드를 수행하는 LogbackMDCAdapter의 remove()메서드를 보면 key가 null이 아닐경우에 ThreadLocal을 꺼내서 key에 mapping된 value값을 삭제해준 이후에 해당 ThreadLocal을 null로 처리해주는 것을 알 수 있다.



```
public void remove(String key) {
    if (key != null) {
        Map<String, String> current = (Map)this.readWriteThreadLocalMap.get();
        if (current != null) {
            current.remove(key);
            this.nullifyReadOnlyThreadLocalMap();
        }

    }
}

private void nullifyReadOnlyThreadLocalMap() {
    this.readOnlyThreadLocalMap.set((Object)null);
}

```

> 참고 : LogbackMDCAdapter의 코드 중 일부
 
### 마무리 


MDC는 내부적으로 ThreadLocal을 사용한다. 
ThreadLocal을 사용하는 Thread는 ThreadPool에서 재활용된다. 
따라서 ThreadLocal을 내부적으로 이용하는 MDC또한 사용하고 난 이후엔 remove()메서드를 호출해서 
ThreadLocal을 삭제해줘야한다. 

다음 글에는 MDC를 어느곳에서 어떠한 방식으로 적용하는 것이 좋을지에 대해서 작성해보겠다. 

