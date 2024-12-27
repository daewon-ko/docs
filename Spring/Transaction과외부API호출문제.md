# 트랜잭션과 외부 API 호출문제
#docs/Spring
#docs/쇼핑몰프로젝트

### 하나의 트랜잭션 내에서 DB접근과 외부 API 호출이 묶여있는 경우의 문제


다음은 결제 서비스에서 외부 API를 호출하는 코드이다.
Service Layer의 confirm이라는 메서드에서 TossAPI를 호출하는 paymentClient의 특정 메서드를 호출한다. 그런데 아래와 같은 방식의 트랜잭션 처리는 과연 문제가 없을까? 


```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class PaymentService {

    private final PaymentClient paymentClient;
    private final OrderRepository orderRepository;
    private final PaymentRepository paymentRepository;
    private final RedisTemplate<String, Object> redisTemplate;

    private static final String REDIS_PAYMENT_PREFIX = "payment_order";

    //TODO : 결제에서의 트랜잭션 처리는 어떻게?
    // 외부 API를 연동하는데 트랜잭션 처리를 어떻게 해주는게 적절할까?
    @Transactional
    public PaymentResponse confirm(final TossPaymentConfirmRequest request, final Long orderId) {
        Order findOrder = orderRepository.findById(orderId).orElseThrow(() -> new ApiException(OrderErrorCode.NOT_EXIST_ORDER));

        // 외부 API 호출
        final TossPaymentConfirmResponse tossPaymentConfirmResponse = paymentClient.confirmPayment(request);

        // TossPayment 객체 생성 및 저장
        TossPayment tossPayment = TossPayment.of(tossPaymentConfirmResponse, findOrder);
        paymentRepository.save(tossPayment);

        // Write-Through : Redis에 저장
        String paymentKey = REDIS_PAYMENT_PREFIX + tossPayment.getOrder().getId();

        redisTemplate.opsForValue().set(paymentKey, tossPayment);

        // Cache Store 내에 하루 저장
        // TODO : 결제 내역을 Cache에 얼마나 저장해야하는가?
        redisTemplate.expire(paymentKey, Duration.ofDays(1));

        PaymentResponse paymentResponse = PaymentResponse.from(tossPayment);

        return paymentResponse;

    }


```


### 현재 방식의 문제점

* 외부 API 콜이 실패하면 해당 로직 이하는 모두 실행되지 않는다.
  * DB에 저장하는 로직이 실행되지 않으므로 롤백 등이 필요가 없다.
* 그러나 **외부 API 콜에는 성공하지만 DB 저장 로직 등에서 실패**하게 된다
  * 외부 API는 성공했기 때문에, 실제 결제를 수행하는 Toss Server에서는 결제이력이 남지만, 본인의 서버 DB에는 결제이력이 남지 않는 **데이터 정합성 문제**가 발생한다.

현재 파악한 문제점을 기반으로 해결할 수 있는 방안에 대해서 고민해봤다.

### 고려한 선택지

1. 외부 API 호출 순서를 뒤로 수정한다.
   - 이 경우엔 서버 내 DB 쓰기 작업에서 예외가 발생해도 실제 결제를 수행하는 외부 API 호출과는 무관하다. 
   - 그러나 현 상황에서 DB 스키마(TossPayment 객체는 Entity이다.) 등의 변경이 불가피하다.
2. 외부 API 호출 순서를 현재와 같이 유지하되, DB 쓰기작업 실패시에 보상 API를 구현해준다. 

1번과 2번 선택지에 대해서, DB 스키마 변경

### 보상 API의 구현 방식

API 클라이언트에 결제취소를 요청하는 메서드를 새롭게 PaymentClient에 다음과 같이 정의한다. 

1. 단순 try-catch로 잡아서 paymentClinet에게 취소 요청을 보낸다.
2. Transaction Phase가 Rollback일 경우, 특정 이벤트를 생성해서 eventPublisher가 이벤트를 발행한다. 
   * event를 소비하는 eventHandler가 로직을 구성한다.
   * 결제 실패 후 보상API 로직(결제 취소 요청)에 대하여 **비동기 처리**가 가능하다.

보상 API를 구현함에 있어서 위와 같은 두 가지 선택지를 고려했다.
단순하게 1번 방식으로 구현할 수도 있지만, 결제가 실패하여 외부 API 서버에 결제 실패 처리 요청의 경우 반드시 동기적으로 작성할 필요가 있을까? 라는 질문이 스스로 들었다. 

나는 보상 API를 구현하는 선택지에서 2번을 선택하였다. 

그 근거는 다음과 같다. 

1. WAS 내에서 결제는 실패하였고, 저장하지 못한 결제내역에 대해서 실제 결제를 수행하게 되는 대행사 서버(외부 API)에게 결제실패로 변경해달라는 것이 **반드시 동기적일필요는 없다**라고 생각이 들었다.

2. 외부 API호출을 비동기적으로 처리할 경우에 응답대기를 최소화할 수 있다. 무엇보다 외부 서버에 결제 취소를 요청하는 것에서 데이터 정합성을 맞추는 것에서 대기할 필요가 없다.
3. 즉각적 사용자 피드백이 불필요한 영역이고, 외부 API와 데이터 정합성을 맞추는 과정이다. 즉, 서버 DB와 토스 서버 간의 정합성 유지가 핵심이다. 
4. 또한 추후 알림과 같은 도메인 구성 시에, 해당 도메인 역시 현재 상황에서 비동기처리로 하는 것이 더 깔끔하고 적절하다고 판단되다. 또한 단순 비동기 처리가 아니어도, 결합도 측면에서도 Event등의 Layer를 두어서 분리하는 것이 더 적절하다.

> **CF) Retry는 언제, 어느 순서에 구성해야할까?** 
```
외부 API 호출의 순서를 먼저할 것이냐 아니면 뒤에 할것이냐에 상관없이 Retry 로직은 필요할 수 있다. 

가령, 외부 API 호출을 먼저한다고 해서 성공한다고 해도 서버 내부에 TossPayment 객체를 저장하는 과정 등에서 예외가 발생한다면, 외부 API 서버에만 결제 내역이 남게되고 본인의 서버 내에는 결제이력이 DB에 저장되지 않느다. 이 경우에 문제가 생길 수 있기에 Retry 등이 필요하다.

또한 외부 API 호출을 로직의 마지막에서 한다면, TossPayment 객체를 DB에 하여 서버 내에서는 이력을 갖고 있는데 외부 API 호출 과정에서 예외가 발생한다면(secret-key 등의 문제) 전체 모든 로직을 롤백하고 Retry를 실행해줘야한다. 

따라서 외부 API 호출의 순서와 관계없이 Retry 로직은 필요하다.


```

> 재시도와 관련된 로직은 본 글의 취지와 맞지 않은 관계로 일단 간략히 필요성에 대해서만 언급하겠다. 추후 필요시 Retry와 관련된 로직을 구성하는 방안에 대해서 더 깊게 고민해보겠다.


### Spring의 이벤트 기반 통신 구조

Spring이 지원하는 이벤트 기반 통신은 하나의 어플리케이션 컨텍스트 내에서 지원한다. 따라서 하나의 모듈 내에서 여러 패키지로 구분될떄 혹은 하나의 컨텍스트이지만 여러 모듈로 분리되었을때 이를 이용할 수 있다. 
만약 MSA와 같이 각각의 어플리케이션 컨텍스트로 구분되고 이를 이벤트 기반으로 통신하기 위해서는 Kafka, Redis Pub/Sub 구조, RabbitMQ등을 이용해야한다. 

Spring Event는 Event 자체인 ‘Event’, 이벤트를 발행하는 주체인 ‘publisher’, 이벤트를 처리하는 ‘listenr’가 있다.

큰 틀에서 컴포넌트들은 다음과 같다. 


이벤트 통신의 주요 컴포넌트
- ApplicationEventPublisher
  - 이벤트 객체를 생성하기 위해 상속받아야 하는 클래스
  - publishEvent 메서드를 통해 이벤트 생성
  - ApplicationContext가 해당 클래스를 구현한다. (즉 ApplicationEventPulisher가 ApplicationContext보다 더 상위 인터페이스이다.)
  - 이벤트를 발행해야하는 로직의 클래스에서 이를 필드로 둔다.
- ApplicationEvent
  - 이벤트 객체를 생성하기 위해 상속받아야하는 클래스
  - Spring 4.2부터는 CustomEventClass가 상속하지 않아도 된다.
- ApplicationListener
  - 이벤트를 리스닝하기 위해 구현해야하는 클래스
  - CustomHandler가 해당 클래스를 구현하여 onApplicationEvent(E)메서드를 구현한다.
    - Spring 4.2부터 CustomHadler가 메서드를 구현하지 않고, @EventListener 어노테이션을 메서드 레벨에 붙여서 작성한다.
  - 해당 메서드는 ApplicationEventPublisher에서 특정 이벤트를 publish하면 자동으로 실행된다.


