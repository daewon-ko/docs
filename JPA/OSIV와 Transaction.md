# OSIV와 Transaction

### 배경

통상 Service Layer의 class에 @Transactional을 건다. 

해당 어노테이션은 웹요청이 있을때 해당 클래스 / 메서드를 호출하는 요청에 대하여 
DB 트랜잭션을 보장하기 위함이다.(트랜잭션은 ACID의 특성을 보장한다.)

코드를 읽고, 작성하다보면 @Transactional(readOnly = true)와 같은 어노테이션을 자주 보거나 사용하게 되는데 해당 어노테이션은 단순조회 시 사용된다. 즉 SELECT와 같은 조회용 쿼리와 INSERT, UPDATE, DELETE와 같은 쿼리를 구분하기 위해서 해당 옵션을 주곤한다.

그런데, 내가 속해있는 오픈카톡방에서 어떤 분께서 다음과 같은 질문을 남겨주셨다.

<img width="558" alt="image" src="https://github.com/daewon-ko/docs/assets/105340285/882b20d2-f960-4296-9639-0cb476201d09">

개념어들은 잘 알고있다고 생각했지만, 곰곰히 생각해보면 위의 의문에 나는 어떻게 답변을 할 수 있을지 잘 모르겠다는 생각이 들었다. 그래서 오개념, 혼동개념들에 대해서 정리해보고자 한다.



### @Transactional(readOnly = true)

@Transactional(readOnly = true)는 트랜잭션을 적용한다. 

해당 어노테이션이 존재하는 범위 내에서 데이터의 변경을 시도할 경우 예외를 발생시킨다.

JPA와 연관되어 사용될 경우, 영속성 컨텍스트 내에 변경감지(Dirty Checking)기능을 비활성화하여 성능을 최척화 한다. 

Service Layer에 해당 어노테이션이 작성되어있으면 DB Connection을 잡고 있게 된다. 



추정컨대, 위에서 의문을 남기신 분은 트랜잭션으로 인해 DB Connection을 잡고 있는 비용이 단순조회라는 기능에 비해서 너무 큰 것이 아닌가 하는 의문이 든 것으로 보인다. 

즉, 어차피 DB에 수정을 가하는 (INSERT, UPDATE, DELETE)와 관련된 것이 아니고 단순 조회라면 굳이 해당 어노테이션을 붙여서 DB connection을 오랫동안 물고 있을 필요가 있느냐 하는 것이다. 



그러나 다음의 코드를 생각해보자.

```java
@Service
public class ProductService {

    @Autowired
    private ProductRepository productRepository;

    @Transactional(readOnly = true)
    public List<Product> getProductsTransactional() {
        List<Product> products = productRepository.findAll();
        // 반복 조회
        List<Product> productsAfterChange = productRepository.findAll();
        return productsAfterChange;
    }

    public List<Product> getProductsNonTransactional() {
        List<Product> products = productRepository.findAll();
        // 반복 조회
        List<Product> productsAfterChange = productRepository.findAll();
        return productsAfterChange;
    }
}

```

단순 조회기능의 메서드이지만 각 메서드의 차이는 @Transactional(readOnly = true)만을 존재한다.

위의 코드에 대해 간단한 테스트 코드를 작성하면 다음과 같다.

```java
    @Test
    public void testTransactional() {
        // Given: 데이터베이스에 몇 개의 제품이 이미 존재한다고 가정
        Product product1 = new Product();
        product1.setName("Product 1");
        product1.setPrice(new BigDecimal("10.00"));
        productRepository.save(product1);

        // When: 첫 번째 트랜잭션에서 데이터 조회
        List<Product> productsTransactional = productService.getProductsTransactional();
        // Then: 동일한 데이터가 조회됨을 확인
        assertThat(productsTransactional).hasSize(1);

        // When: 데이터베이스에 새로운 제품 추가
        Product product2 = new Product();
        product2.setName("Product 2");
        product2.setPrice(new BigDecimal("20.00"));
        productRepository.save(product2);

        // Then: 동일한 트랜잭션 내에서 변경 사항이 반영되지 않음
        productsTransactional = productService.getProductsTransactional();
        assertThat(productsTransactional).hasSize(1);
    }
```

첫번째 조회할때, 데이터가 영속성 컨텍스트에 캐싱이 되고, 두번째 메서드가 다시 한 번 실행될때 같은 트랜잭션 범위 내에 존재하므로, 앞서 DB에 저장된 product2가 영속성 컨텍스트에 반영되지 못한다. 즉 트랜잭션이 보장된다.



다음은 @Transactional(readOnly = true)가 없는 메서드에 대한 테스트 코드이다.

```java
  @Test
    public void testNonTransactional() {
        // Given: 데이터베이스에 몇 개의 제품이 이미 존재한다고 가정
        Product product1 = new Product();
        product1.setName("Product 1");
        product1.setPrice(new BigDecimal("10.00"));
        productRepository.save(product1);

        // When: 첫 번째 조회
        List<Product> productsNonTransactional = productService.getProductsNonTransactional();
        // Then: 동일한 데이터가 조회됨을 확인
        assertThat(productsNonTransactional).hasSize(1);

        // When: 데이터베이스에 새로운 제품 추가
        Product product2 = new Product();
        product2.setName("Product 2");
        product2.setPrice(new BigDecimal("20.00"));
        productRepository.save(product2);

        // Then: 새로운 데이터가 즉시 반영됨
        productsNonTransactional = productService.getProductsNonTransactional();
        assertThat(productsNonTransactional).hasSize(2);
    }
```

메서드가 실행될때마다 DB Connection이 맺어지고 DB를 조회하게 된다. 
즉, 서비스 Layer에서 트랜잭션 설정에 관한 옵션이 없으므로 Repository Layer에 해당 권한을 위임하게 되고, Repository Layer에서 DB에 접근하는 메서드가 실행될때마다 DB Connection이 생성되는 것이다. 

(참고로 Service Layer의 트랜잭션이 Repository Layer의 트랜잭션을 포함하는, 물고있는 개념으로 생각하면 된다. 가령 Repository Layer의 여러 메서드를 Service Layer의 하나의 메서드에서 호출할때 하나의 트랜잭션, 하나의 Connection 내에서 관리된다. )



따라서 데이터 정합성 측면에서도, 한정된 자원인 DB Connection 개수 차원에서도 @Transactional(readOnly = true)를 사용하지 않을 이유가 없는 것이다.



### OSIV

@Transactional(readOnly = true)에 관한 의문이 다소 해결되었는데, 
어떤 익명의 분께서 OSIV라는 키워드에 대한 대답을 남겨주셨다.

OSIV라는 개념어도 처음 들어보는지라 한 번 찾아보게 되었다.

<img width="558" alt="image" src="https://github.com/daewon-ko/docs/assets/105340285/4cea5efe-0e59-40aa-a8ae-f4aa6c2dd8bc">

OSIV는 영속성 컨텍스트를 뷰까지 열어두는 기능이다. 

용어에서 알 수 있겠지만 스프링과 JPA를 이용할때 사용되는 개념이다. 



통상적인 트랜잭션의 범위는 Layer로 치면 Service Layer부터이다. 

해당 범위가 곧 영속성 컨텍스트의 생존 범위이다. 

이 경우에 트랜잭션이 종료되면 영속성 컨텍스트 역시 종료되고 당연히 DB Connection 역시 DB Connection Pool에 반환된다. 따라서 커넥션 리소스가 낭비되지 않는다. 



이는 JPA와 연관되어 조금 더 생각해보자면, JPA의 주된 특징 중 하나인 지연로딩과 아주 밀접한 관계가 있다. 즉, 특정 지연로딩과 관련된 코드를 트랜잭션이 작동하는 범위 내에 집어넣어야하는 것이다. (사실 이부분은 아직 명확히 이해가 가진 않는다. 왜냐하면 지연로딩과 관련된 코드 즉 로직은 대부분 전통적인 3-tier Architecture 상에서는 Service Layer 이하에 존재하기 때문이다. )

그런데, OSIV는 영속성 컨텍스트의 생존 범위를 단순히 Service Layer까지가 아닌 Controller, View단까지 넓히는 것이다. 



옵션의 기본값은 false이고 application.properties와 같은 파일에서 다음의 한 줄을 통해서 설정이 가능하다.

> ```
> spring.jpa.open-in-view : true 
> ```



동작 원리는 다음의 순서로 이뤄진다. 

>```
>1. 클라이언트의 요청이 들어오면 서블릿 필터나, 스프링 인터셉터에서 영속성 컨텍스트를 생성한다. 단 이 시점에서 트랜잭션은 시작하지 않는다.
>2. 서비스 계층에서 @Transeactional로 트랜잭션을 시작할 때 1번에서 미리 생성해둔 영속성 컨텍스트를 찾아와서 트랜잭션을 시작한다.
>3. 서비스 계층이 끝나면 트랜잭션을 커밋하고 영속성 컨텍스트를 플러시한다. 이 시점에 트랜잭션은 끝내지만 영속성 컨텍스트는 종료되지 않는다.
>4. 컨트롤러와 뷰까지 영속성 컨텍스트가 유지되므로 조회한 엔티티는 영속 상태를 유지한다.
>5. 서블릿 필터나, 스프링 인터셉터로 요청이 돌아오면 영속성 컨텍스트를 종료한다. 이때 플러시를 호출하지 않고 바로 종료한다.
>```

위에서 말하는 필터와 인터셉터는 다음과 같다. 

필터 : https://docs.spring.io/spring-framework/docs/5.3.3/javadoc-api/org/springframework/orm/jpa/support/OpenEntityManagerInViewFilter.html

인터셉터 : https://docs.spring.io/spring-framework/docs/5.3.3/javadoc-api/org/springframework/orm/jpa/support/OpenEntityManagerInViewInterceptor.html

Service Layer에서는 트랜잭션이 종료된 이후에 컨트롤러와 뷰 단에서는 트랜잭션이 유지되지 않는다. 그러나, 엔티티의 상태를 변경하지 않는 '단순 조회'의 경우엔 트랜잭션을 사용하지 않아도 가능하다. 따라서 프록시 객체에 대하여 지연 로딩을 사용하는 경우, 단순 조회의 경우 트랜잭션 없이 읽기가 가능하다.
(물론 여기서는 DB의 트랜잭션 격리 수준 그 자체에 대한 논의는 배제한다.)

또한 트랜잭션의 범위 밖인 Controller와 View에서 영속성 컨텍스트의 값을 수정하여도, 영속성 컨텍스트의 변경감지에 의한 데이터 수정은 일어나지 않는다.

1. OSIV는 웹 요청에 대해서 em.flush가 아닌 em.close()를 호출하여 영속성 컨텍스트를 종료시키기 때문이다.
2. em.flush()를 임의로 호출 시 예외가 발생한다.

​	-> javax.persistence.TransactionRequiredException



전술한 OSIV 특성에서 알수 있다시피, OSIV는 웹요청에 대해서 데이터베이스 커넥션 리소스를 API 응답이 끝날때가지 유지한다. 이로 인해 view나 controller등에서 지연로딩 기능을 사용할 수 있는 것이지만 DB Resource에 대해서 큰 부담이 되고 장애로 이어질 수 있는 여지가 된다. 



결국엔 모든게 Trade-Off인 셈이다. 서버의 복잡성을 분리하기 위해서 단순조회용과 그 외 기능으로 분리하였고 해당 기능을 조금 더 잘 사용하기 위해 OSIV라는 기능까지 있지만 해당 기능엔 결국 Resource와 관련된 중대한 문제가 있었다. 



### CQRS

아직은 잘 모른다. 그러나 이름은 들어본 그것이다.

OSIV를 사용하지 않은 채로 위의 복잡성을 용이하게 관리하는 주요한 방법이 Command(INSERT, UPDATE, DELETE성)와 Query(단순 조회)을 분리하는 것이다. 

즉, DB Connection을 앞단까지 물고있지 않음을 통해서 Resource 낭비를 덜하게 하되 
단순 조회와 그 외의 로직을 나눠서 관심사의 분리를 가능케 한다. 



크고 복잡한 어플리케이션을 개발할때 둘의 관심사를 분리하는 것이 유지보수 관점에서 도움이 될 것은 명백하다. 



다음과 같은 방식으로 분리할 수 있다고 한다. 

```
// OrderService를 다음의 두 개의 모듈(클래스?)등으로 나눠서 관리

OrderService: 핵심 비즈니스 로직
OrderQueryService: 화면이나 API에 맞춘 서비스 (주로 읽기 전용 트랜잭션 사용)
```

서비스 Layer에서 트랜잭션을 유지한다.

통합성 있고 일관되게 모두 트랜잭션을 유지하며 지연로딩 기능이 사용 가능하다. 



참고 
https://ykh6242.tistory.com/entry/JPA-OSIVOpen-Session-In-View와-성능-최적화