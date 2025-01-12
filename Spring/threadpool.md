# ThreadPool 
#docs/Spring

### ThreadPool
WAS는 하나의 프로세스이다. 

프로세스는 다수의 Thread라는 subset으로 구성되어있는데, 이는 프로세스 내 하나의 일꾼이자 자원이라고 할 수 있다. 

(OS적 관점에서 프로세스와 쓰레드의 이론적 고찰 내지는 깊은 차원의 논의는 현 상황에선 일단 배제하겠다. ThreadPool이라는 것의 이해를 위해 조금은 러프하게 설명하도록 양해를 바란다.)

Thread가 자원이라고 하였기 때문에 Thread를 생성하는 것은 당연히 비용이 드는 일인데, 만약 무제한 적으로 Thread를 생성하게 되면 CPU 코어 자원을 낭비하게 된다. 

이는 당연히 성능상 문제를 유발할 수 밖에 없다. 

만약 특정 시간에 서버에 매우 많은 수의 급증한 트래픽이 몰린다고 가정해보자.
(가령 쿠팡같은 경우에 새로운 아이폰 시리즈 판매를 개시한다든가, 배달과 관련된 서버에서 큰 폭의 할인 쿠폰을 뿌린다든가 하는 때)

매 요청시마다 Thread를 생성해서 할당한다면 서버는 금새 다운이 되버릴 것이다.

상기의 이유로 WAS는 실행될때, 미리 Thread를 생성해놓고 요청시마다 Thread를 할당하는 식으로 동작을 하는데 이것이 ‘Threadpool’이다.

사실 생각해보면 다 리소스 낭비를 막기위한 방책인 것이다. 


### ThreadPool의 동작방식
ThreadPool을 생성하는 클래스는 ‘ThreadpoolExecutorService’이다.
생성 시 매개변수로는 corePoolSize, maximumPoolSize, keepAliveTime, TimeUnit, workQueue가 있다.

* corePoolSize
  * ThreadPoolExecutor가 동시에 수행할 수 있는 Threa 수 
* maximumPoolSize
  * ThreadPoolExecutor가 최대 수행할 수 있는 Thread수
  * workQueue가 모두 차게되면, corePoolSize보다 많은 Thread를 생성하여 maximumPoolSize까지 Thread수가 증가하게된다.
* keepAliveTime
  * maximumPoolSize에 의해 생성된 스레드는 corePoolSize 이상의 추가 스레드가 되는데, 해당 스레드들이 작업 이후에 유휴상태로 전환된다.
  * 해당 스레드가 유휴상태로 유지될 수 있는 시간이 keepAliveTime이며 해당 시간 이후에 추가 작업이 주어지지 않으면 자동으로 종료된다.
* timeUnit
  * keepAliveTime 옵션을 위한 시간 단위
* workQueue
  * Thread Pool에서 실행가능한 Thread가 없을 경우 , 해당 공간에 Queue구조와 같이 Thread가 대기한다.

해당 클래스는 ThreadPool, WorkQueue를 통해서 제어하는데 해당 내용은 다음과 같다. 
```markdown
ThreadPoolExecutor 동작 방식
	1. corePoolSize보다 적은 수의 스레드가 실행 중이면 즉시 새로운 스레드 할당
	◦	요청이 들어왔을 때 실행 중인 스레드의 수가 corePoolSize 미만이라면, 바로 새로운 스레드를 생성하여 작업을 할당합니다.
	◦	이 작업은 대기열(workQueue)을 사용하지 않습니다.

	2	corePoolSize만큼 스레드가 이미 실행 중이라면, 작업은 workQueue에 저장
	◦	실행 중인 스레드가 corePoolSize와 같거나 유휴 스레드가 없는 경우, 새로운 요청은 대기열(workQueue)에 저장됩니다.
	◦	대기열의 크기가 설정되어 있다면, 대기열이 가득 찰 때까지 요청을 계속 저장합니다.

	3	workQueue가 가득 찬 경우, maximumPoolSize까지 추가 스레드 생성
	◦	대기열이 가득 차면, 스레드 풀은 maximumPoolSize까지 새로운 스레드를 생성하여 작업을 처리합니다.
	◦	이때 생성되는 추가 스레드의 수는 maximumPoolSize와 현재 실행 중인 스레드 수의 차이만큼입니다.

	4	maximumPoolSize와 workQueue가 모두 가득 찬 경우, 예외 발생
	◦	스레드 풀에 더 이상 스레드를 생성할 수 없고, 대기열에도 작업을 추가할 수 없는 경우, RejectedExecutionHandler를 통해 **RejectedExecutionException**이 발생합니다.

```

4번에서 언급했듯이, 모든 ThreadPool이 maxPoolSize만큼 꽉차게되면 정책을 설정할 수 있는데, ThreadPoolExecutor에서 기본적으로 정의한 내용은 다음과 같다. 

* Abort Policy
  * 기본전략
  * RejectedExecutionException 내뱉음
* CallerRunsPolicy
  * Reject된 Task를 실행중인 main쓰레드에서 동작
* DiscardPolicy
  * Reject된 task폐기, Exception 발생하지 않는다.
* Discard-OldestPolicy
  * 가장 오래된 처리되지 않은 요청을 삭제하고 다시 시도


ThreadPool은 Thread들이 모여있는 일종의 가상 풀이다.


CF) ThreadPoolExecutor를 실행할때 submit, execute 2가지 방식이 존재한다.

```
execute() 방식은 작업 중 예외가 발생하면 해당쓰레드가 종료되고 JVM까지 전파된다. 그 이후 새 스레드를 생성하여 스레드 생성비용이 존재한다.

submit() 방식은 Future<?>를 반환하여, 예외가 발생할지라도 스레드는 종료되지 않고 재사용된다.

상황에 따라 다르겠지만, 생성 비용의 관점에선 submit()사용이 권장된다.
```



### Spring의 ThreadPoolTaskExecutor

Spring에서는 ThreadpoolExcutor대신 Spring에 커스텀하게 맞춘 ThreadPoolTaskExecutor를 사용한다. 

```java
package org.springframework.scheduling.concurrent;

import java.util.Map;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.Callable;
import java.util.concurrent.Executor;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Future;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.RejectedExecutionException;
import java.util.concurrent.RejectedExecutionHandler;
import java.util.concurrent.SynchronousQueue;
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;
import org.springframework.core.task.AsyncListenableTaskExecutor;
import org.springframework.core.task.TaskDecorator;
import org.springframework.core.task.TaskRejectedException;
import org.springframework.lang.Nullable;
import org.springframework.scheduling.SchedulingTaskExecutor;
import org.springframework.util.Assert;
import org.springframework.util.ConcurrentReferenceHashMap;
import org.springframework.util.ConcurrentReferenceHashMap.ReferenceType;
import org.springframework.util.concurrent.ListenableFuture;
import org.springframework.util.concurrent.ListenableFutureTask;

public class ThreadPoolTaskExecutor extends ExecutorConfigurationSupport implements AsyncListenableTaskExecutor, SchedulingTaskExecutor {
    private final Object poolSizeMonitor = new Object();
    private int corePoolSize = 1;
    private int maxPoolSize = Integer.MAX_VALUE;
    private int keepAliveSeconds = 60;
    private int queueCapacity = Integer.MAX_VALUE;
    private boolean allowCoreThreadTimeOut = false;
    private boolean prestartAllCoreThreads = false;

(이하 중략)

```

기본 corePoolSize는 1, maxPoolSize와 queueCapacity는 Integer.MAX_VALUE로 설정되어있는 것을 볼 수 있다. 

즉, Integer.MAX_VALUE만큼 workrQueue가 차기 전까지는 하나의 스레드로 작업을 처리한다. 해당 크기만큼 대기열이 차게되면 maxPoolSize만큼의 Thread를 생성하게된다.

