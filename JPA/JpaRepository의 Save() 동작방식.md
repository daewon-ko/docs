# JpaRepository save() 동작원리









**배경**



토이 프로젝트를 진행하며 멀티모듈을 도입하고 각 모듈의 성격에 맞게 코드를 작성하고 싶어서 도메인 객체와 Entity를 분리하였습니다. 또한 도메인 객체는 최대한 POJO로 구성하여 해당 객체를 특정 DB또는 기술과의 결합도를 낮추고자 하였습니다. 



이에 따라 트레이드오프로서 도메인 객체가 JPA를 사용하는 RDB를 통하여 읽기 작업 / 쓰기작업을 수행할때 변환해야하는 일이 많아졌습니다.



아래 그림과 같이 Domain 객체를 Entity로 서로 변환하고 모듈을 이동할때는 최대한 Domain 객체만을 전달하려고 노력하는 등 하다보니 Entity를 조회하고, 또 해당 Entity의 PK값을 이용해서 다른 Entity를 새로 생성하고 저장할때 새롭게 생성해서 저장하였습니다. 





![jpaRepository (1).jpg](blob:file:///6435f12c-c26f-46a1-a309-9e5d366ccd39)

**문제 상황**



조금 더 구체적으로 설명해볼까요? 



- 기존의 코드



과거에 단일 모듈에서 Entity와 Domain 객체를 별도로 분리하지 않은 채로 코드를 작성할 때는 특정 Service Layer에서 여러 Repository를 호출하여 코드를 조합해서 사용하곤 했습니다. 



가령 주문을 생성한다는 로직을 다음과 같이 구성했습니다.

(예시입니다.)





@Service

@RequiredArgsConstructor

public class OrderService{

​	private final OrderRepository orderRepository;

​	private final UserRepository userRepository;

​	private final XXXRepository xxxRepository;



​	@Transactional

​	public Long makeOrder(final OrderCreateRequestDto orderRequestDto){

​		User findUser = userRepository.findByUserId(orderRequestDto.getUserId());

​		orderRdbService.createOrder(findUser);

​	}





}





위와 같이 Service Layer에 Entity 객체를 직접 올려서 Service Layer에서 다시 Repository Layer로 Entity 객체를 매개변수 등으로 보내주곤 했습니다. 또한 OrderService에서는 Order를 Builder 등으로 생성하여 OrderRepository의 save()메서드를 호출합니다.





@Transactional로 선언된 하나의 메서드 내부에서는 하나의 영속성 컨텍스트가 보장되기 때문에 UserRepository에서 조회한 findUser를 OrderRdbService 메서드의 매개변수로 보내줄 때는 동일한 영속성 컨텍스트가 보장될 것임을 직관적으로 알 수 있습니다. 





- 멀티모듈 도입 이후



 과거에 단일 모듈에서는 하나의 도메인 Service가 담당하던 여러가지 로직을 기능에 따라 세분화 하였는데요. 

멀티모듈 도입 이후에는 domain-service라는 특정 DB와 결합되어 있지 않은 모듈에서 주문을 생성하는 ‘OrderCreateService’라는 클래스가 해당 기능을 수행합니다. 



OrderCreateService는 주문을 생성하는 메서드는 OrderUsecase의 메서드에서 호출하며, 해당 메서드는 @Transactional이 선언되어 있습니다. (즉 주문을 생성한다는 전체 로직은 OrderUsecase라는 상위의 Application Layer에서 하나의 트랜잭션으로 묶여있는 셈입니다.)



세부 클래스는 다음과 같습니다. 

```java

@DomainRdbService

@RequiredArgsConstructor

public class OrderCreateService {

  private final OrderRdbService orderRdbService;

  private final CartProductRdbService cartProductRdbService;

  private final UserRdbService userRdbService;

  private final ProductRdbService productRdbService;

  private final OrderProductRdbService orderProductRdbService;







(중략)...



private Long makeOrder(OrderCreateRequestDto orderCreateRequestDto) {

  UserDomain userDomain = userRdbService.findByUserId(orderCreateRequestDto.getUserId());



  AddressDomain addressDomain = AddressDomain.builder()
	.zipCode(orderCreateRequestDto.getZipCode())
	.detailAddress(orderCreateRequestDto.getDetailAddress()).build();





  OrderDomain orderDomain = OrderDomain.createForWrite(OrderStatus.NEW, userDomain.getUserId(), addressDomain);





  // Order 생성

  return orderRdbService.createDirectOrder(orderDomain);

}



}

```





OrderCreateService가 호출하는 OrderRdbService의 코드는 다음과 같습니다. 





```java
@DomainRdbService

@RequiredArgsConstructor

@Transactional(readOnly = true)

public class OrderRdbService {



  private final OrderRepository orderRepository;

  private final OrderQueryRepository orderQueryRepository;



@Transactional

public Long createDirectOrder(final OrderDomain orderDomain) {

  return createCommonOrder(orderDomain);

}





private Long createCommonOrder(final OrderDomain orderDomain) {



  // Order 생성(DB 저장)

  Order order = Order.builder()
  	.user(User.fromUserId(orderDomain.getUserId())
		.detailAddress(orderDomain.getAddressDomain().getDetailAddress())
		.zipCode(orderDomain.getAddressDomain().getZipCode())
		.orderStatus(orderDomain.getOrderStatus())
		.build();





  Order savedOrder = orderRepository.save(order);

  return savedOrder.getId();





}

}

```

















위와 같이 **여러 메서드에서 @Transactional이 선언되어서 여러가지 논리 트랜잭션이 존재하고 모듈 간 Entity를 Domain으로 변환하며 전달할때 영속성 컨텍스트에는 어떠한 Entity가 관리되고 있는지**? 또, 영속성 컨텍스트에서 관리되고 있지 않은 새로운 객체를 생성해서 연관관계를 맺어준다고 해도 새**로운 Entity(Order)는 정상적으로 save()가 되는지가 궁금**하였습니다. 



**트랜잭션과 영속성 컨텍스트** 

JPA의 영속성 컨텍스트는 하나의 트랜잭션 안에서 동작합니다. 

여기서 말하는 ‘트랜잭션’의 범위란 스프링이 관리하는 논리 트랜잭션을 의미합니다. 



기존의 코드에서는 별도로 트랜잭션을 전파하지 않고 하나의 메서드에서 하나의 기능을 포괄하였지만, 리팩토링 이후에는 여러 논리적 트랜잭션 메서드가 하나의 실제 트랜잭션으로 동작합니다. 여기서 말하는 ‘트랜잭션’이란 ‘논리 트랜잭션’으로서 작업의 범위를 의미합니다. 즉, UserRdbService에서도, OrderRdbService 등에서도 @Transactional을 가지고 있지만, 결과적으로 OrderUsecase의 @Transactional이라는 하나의 트랜잭션 내부에서 동작하기 때문에(전파 옵션 등이 없기에) 동일한 트랜잭션 범위에 들어가며 동일한 영속성 컨텍스트 내부에서 동작하는 것이입니다.



즉 결과적으로 아래 그림과 같은 형태이고 OrderCreateService에서 OrderRdbService에 조회했던 User객체도 동일한 영속성 컨텍스트에서 관리되고 있습니다. 







![jpaRepository (4).jpg](blob:file:///0be1df6f-16dd-446d-b2df-37f8053fa6cb)







그렇다면 save()메서드는 정상적으로 동작할까요? 

또 정상적으로 동작한다면 왜 그럴까요? 



**JpaRepository의 save메서드의 동작방식**



JpaRepository는 다음과 같이 인터페이스 형식으로 상속받아 편리하게 CRUD메서드를 사용합니다. 



public interface OrderRepository extends JpaRepository<Order,Long>





JpaRepository 인터페이스를 구현하는 SimpleJpaRepository를 target으로 가지는 Proxy가 Bean으로 등록됩니다. 



즉, 실제 CRUD는 SimpleJpaRepository가 수행하게 됩니다.

해당 클래스의 save 메서드는 다음과 같습니다. 



```java
@Transactional

public <S extends T> S save(S entity) {

  Assert.notNull(entity, "Entity must not be null");

  if (this.entityInformation.isNew(entity)) {

​    this.entityManager.persist(entity);

​    return entity;

  } else {

​    return this.entityManager.merge(entity);

  }

}
```











EntityInfromation isNew 메서드를 통해 전달받는 Entity가 새로 등록되는 Entity라면 Persist를 하고, 그렇지 않으면 merge를 합니다. 



- Persist
  - 비영속(new)상태의 Entity를 영속(Persist)상태로 만든다.
  - 영속성 컨텍스트와 전혀 관계를 맺지 않지 않았던 Entity를 영속성 컨텍스트에 저장하여 관리한다.
- Merge
  - 기존에 영속성 컨텍스트에 의해 관리되다가 분리(Detached)되어 다시 영속상태로 만든다.



참고로 isNew메서드의 경우 JpaEntityInfromationSupport 클래스에서 엔티티의 식별자(Id)가 Null인지 여부를 통해 검사합니다. 





따라서 새롭게 생성하는 Order Entity 의 경우 기존에 **UserEntity가 영속성 컨텍스트에서 관리되고 있는지 여부와 관계없이** 식별자가 null이기 때문에 SimpleJpaRepository의 save()메서드 내부에서 isNew가 true로 판단되어 새롭게 영속성 컨텍스트에 관리될수 있으며 새롭게 데이터베이스에 Insert 쿼리가 수행될 수 있습니다. 



그렇다면 JPA는 Entity의 연관관계에 있는 객체를 생성해서 저장할때 무엇이 필요할까요? 



**JPA가 외래키를 처리하는 방식**



객체 관점에서는 연관관계를 맺어줄때 객체 자체가 필요합니다. 

즉 Order Entity를 보면 다음과 같습니다. 





```
package shoppingmall.domainrdb.order.entity;



import jakarta.persistence.*;

import lombok.AccessLevel;

import lombok.Builder;

import lombok.Getter;

import lombok.NoArgsConstructor;

import org.springframework.lang.Nullable;

import shoppingmall.domainrdb.cart.entity.Cart;

import shoppingmall.domainrdb.order.domain.OrderDomain;

import shoppingmall.domainrdb.order.domain.OrderId;

import shoppingmall.domainrdb.order.domain.OrderStatus;

import shoppingmall.domainrdb.user.entity.User;



import java.io.Serial;

import java.io.Serializable;



@Entity

@Table(name = "orders")

@NoArgsConstructor(access = AccessLevel.PROTECTED)

@Getter



public class Order implements Serializable {

  @Serial

  private static final long serialVersionUID = -4316946379860671944L;



  @Id

  @GeneratedValue(strategy = GenerationType.IDENTITY)

//  @Column(name = "orders_id")

  private Long id;



  @Enumerated(EnumType.STRING)

  @Column(name = "order_status")

  OrderStatus orderStatus;



  @Column(name = "zip_code")

  String zipCode;



  @Column(name = "detail_address")

  String detailAddress;





  @ManyToOne(fetch = FetchType.LAZY)

  @Nullable

  Cart cart;



  @ManyToOne(fetch = FetchType.LAZY)

  @JoinColumn(name = "user_id")

  User user;



  @Builder

  private Order(Long id, OrderStatus orderStatus, String zipCode, String detailAddress, @Nullable Cart cart, User user) {

​    this.id = id;

​    this.orderStatus = orderStatus;

​    this.zipCode = zipCode;

​    this.detailAddress = detailAddress;

​    this.cart = cart;

​    this.user = user;

  }



  public static Order fromOrderId(final OrderId orderId) {

​    Order order = new Order();

​    order.id = orderId.getValue();

​    return order;

  }



  public void cancelOrder() {

​    orderStatus = OrderStatus.CANCELED;

  }



  public static OrderDomain toDomain() {



  }

}
```













그런데 객체 관점에서는 그렇지만, DataBase 관점에서는 테이블과 연관관계를 맺어줄때 FK값만 있으면 연관관계를 맺어줄 수 있습니다. 따라서 위에서 Order를 생성하는 메서드가 있을때 Order와 연관관계가 있는 User의 모든 필드는 필요하지 않습니다. 즉 User를 식별할 수 있는 식별자만 존재하면 됩니다. 



아래 코드를 다시 살펴볼까요? 







```

private Long createCommonOrder(final OrderDomain orderDomain) {



  // Order 생성(DB 저장)

  Order order = Order.builder()

​    .user(User.fromUserId(orderDomain.getUserId())

​    .detailAddress(orderDomain.getAddressDomain().getDetailAddress())

​    .zipCode(orderDomain.getAddressDomain().getZipCode())

​    .orderStatus(orderDomain.getOrderStatus())

​    .build();





  Order savedOrder = orderRepository.save(order);

  return savedOrder.getId();





}
```









orderRepository에 저장하는 order 객체를 생성할때, 연관관계를 맺어주는 User의 모든 필드는 불필요하고 Id만 존재해서 식별만 하면 DB입장에서는 FK를 맺어줄 수 있습니다. 그러한 관점에서 User가 영속상태에 있든 비영속상태이든 여부는 중요하지 않습니다. DB Table에 Order를 생성하는 Insert쿼리에 존재하는 User의 FK가 실제로 존재하는지 여부가 중요합니다. (물론 이는 앞에서 UserEntity가 실제로 존재하는지 검증합니다.)



**결론 및 마무리**



JPA에서 엔티티를 저장할 때, save() 메서드는 **엔티티의 식별자 보유 여부에 따라** persist **또는** merge**를 결정**합니다. 식별자가 없는 신규 엔티티는 영속성 컨텍스트에 새로 등록되고(persist), 식별자가 있는 엔티티는 이미 존재한다고 보고 갱신(merge)합니다.



또한 동일 트랜잭션을 공유한다면, 여러 @Transactional이 선언되어 있어도 **결과적으로 하나의 물리 트랜잭션**에서 엔티티를 JPA의 영속성 컨텍스트로 일관성 있게 관리할 수 있습니다.



**DB 차원에서 중요한 것은 FK의 식별자 값**이며, JPA 입장에서는 FK로 사용되는 User 엔티티에 식별자가 제대로 세팅되어 있으면 문제없이 persist가 이뤄집니다. 따라서 멀티모듈 구조에서 도메인 객체와 엔티티를 분리해도, 필요한 순간에 식별자를 바탕으로 엔티티를 생성하고 save() 메서드를 호출하면 정상적으로 Insert 쿼리가 수행됩니다.



다만, **논리 트랜잭션**이 많아지면 트랜잭션 경계를 추적하기 어려워지고, 예기치 않은 스코프 확장이나 중복 호출이 생길 수 있으므로 설계 단계에서 주의가 필요할 것으로 예측됩니다. 


 최종적으로 JpaRepository의 save()가 어떻게 동작하는지, 그리고 어떤 상황에서 persist 또는 merge가 일어나는지를 이해하면, 도메인 객체와 엔티티를 분리한 환경에서도 문제없이 데이터를 저장하고 읽어올 수 있음을 확인하였습니다.