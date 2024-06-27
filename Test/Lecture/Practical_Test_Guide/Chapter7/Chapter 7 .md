# 더 나은 테스트를 작성하기 위한 구체적인 조언

### 한 문단에 한 주제

테스트 -> 문서의 기능을 한다. 

글쓰기의 관점으로 봤을 때, 하나의 테스트가 하나의 문단으로 본다면, 테스트 역시 하나의 주제를 가져야 한다. 



분기문 , 반복문 -> 하나의 내용을 초과한다. 
논리구조가 들어가므로 테스트 코드를 읽는 사람이 한 번 더 맥락에 대해서 생각을 해야한다. 

이는 테스트 코드가 지향하는 것과 반대된다. 

-> Case를 테스트 하고 싶다면 @ParameteredSize와 같은 것을 이용하는 것이 더 적절하다.

부적절한 예

```java
@DisplayName("상품타입이 재고 관련 타입인지 체크한다.")
...
{

//given
ProductType [] productTypes = ProductType.values();

iter.. 
//when
// productType이 HandMade일떄 로직
//then isFalse이다.

//when
//productTYpe이 Bakery일때 로직
//then..


}
```

이는 DisplayName을 한 문장으로 치환할수있는가와도 연관된 문제이다. 

### 완벽하게 제어하기 

* 랜덤값, LocalDateTIme, 네트워크 통신같은 외부계는 최대한 외부로 분리한다.

### 테스트의 환경의 독립성 보장

```java
  @DisplayName("재고가 부족한 상품으로 주문을 생성하려는 경우 예외가 발생한다..")
    @Test
    void createOrderWithNoStock(){
        //given
        Product product1 = createProduct( BOTTLE, "001", 1000);
        Product product2 = createProduct(BAKERY, "002", 3000);
        Product product3 = createProduct(HANDMADE, "003", 5000);
        productRepository.saveAll(List.of(product1, product2, product3));


        Stock stock1 = Stock.create("001", 2);
        Stock stock2 = Stock.create("002", 2);
        stock1.deductQuantity(1); // todo
        stockRepository.saveAll(List.of(stock1, stock2));



        OrderCreateServiceRequest request = OrderCreateServiceRequest.builder()
                .productNumbers(List.of("001", "001", "002", "003"))
                .build();

        //when // then
        LocalDateTime registeredDateTime = LocalDateTime.now();


        assertThatThrownBy(() -> orderService.createOrder(request, registeredDateTime))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessage("재고가 부족한 상품이 있습니다.");



    }
```

위의 코드에서 문제는, stock.deductQuantity를 하는 순간, 재고 이상의 개수를 차감한다면
(가령 3개를 차감한다고 하면) 해당 메서드에서 예외가 터진다. 

테스트하고 싶은 로직은 주문생성과 관련한 것인데 엉뚱한 곳에서 예외가 발생할 수 있는 것이다. 

또한 이 경우엔, 읽는 사람이 deductQuantity를 보고 로직을 한 번 더 생각해야 하는 문제가 있다. 

테스트가 실패를 하더라도 주문 생성과 관련한 곳(then 절 이하)에서 터져야 하는데, 엉뚱한 곳에서 문제가 생길 수 있는 것이다.  만약 복잡한 테스트라면 유추하기 어려운 포인트가 될 수 있다. 



-> 위와 같이 작성하기 stock의 deductQuantity를 호출하기 보다는 아래의 코드처럼 작성하는 것이 좋다. 



```java
  @DisplayName("재고가 부족한 상품으로 주문을 생성하려는 경우 예외가 발생한다..")
    @Test
    void createOrderWithNoStock(){
        //given
        Product product1 = createProduct( BOTTLE, "001", 1000);
        Product product2 = createProduct(BAKERY, "002", 3000);
        Product product3 = createProduct(HANDMADE, "003", 5000);
        productRepository.saveAll(List.of(product1, product2, product3));


        Stock stock1 = Stock.create("001", 1);
        Stock stock2 = Stock.create("002", 2);
//        stock1.deductQuantity(1); // todo
        stockRepository.saveAll(List.of(stock1, stock2));



        OrderCreateServiceRequest request = OrderCreateServiceRequest.builder()
                .productNumbers(List.of("001", "001", "002", "003"))
                .build();

        //when // then
        LocalDateTime registeredDateTime = LocalDateTime.now();


        assertThatThrownBy(() -> orderService.createOrder(request, registeredDateTime))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessage("재고가 부족한 상품이 있습니다.");



    }
```

위와 같이 테스트 환경 자체에서 이미 부족한 재고 수량이 만들어지게끔 먼저 환경을 구축하는 것이 더 올바르다고 할 수 있다. 





### 테스트 간 독립성을 보장하자

* 전역변수로 두고 각 테스트별로 해당 자원을 이용할 경우, 테스트 순서에 따라서 성공/실패가 결정될 수 있다. 
* 따라서 테스트에서 전역변수를 두고 사용하는 것은 권장되지 않는다.

### 한눈에 들어오는 Test Fixture 구성하기

* Test Fixture
  * 테스트를 위해 원하는 상태로 고정시킨 일련의 객체
  * Ex) given 절에서 생성하게되는 객체들
  * @BeforeEach, @BeforeAll등에서 활용가능
    * 그러나, 위의 공유변수에서 언급했다시피 테스트간 결합도를 높일 수 있고 각 단위테스트의 가독성을 떨어트릴 여지가 있으므로 지양하는 것이 좋다. 
    * 그러면 언제 사용하느냐?
    * 1. 각 테스트 입장에서 봤을 때, 아예 몰라도 테스트 내용을 이해하는데 문제가 없는가?
      2. 수정해도 모든 테스트에 영향을 주지 않는가? 
  * 테스트가 길어진다고 해도 given절에 Test Fixture를 같이 적어주는것이 더 옳다. 
  * 3. 테스트클래스 내부에서 객체를 생성할때(빌더패턴이든 팩토리메서드든), 테스트 자체와 무관한 필드는 빼고, 필요한 필드만 넣어서 빌더로 작성하는 것이 더 적절하다 .
    4. given절의 내용을 data.sql로 관리하는 것 역시 부적절하다.
       * 추후 엔티티 변화 및 확장 시 관리 포인트만 더 늘어나게 될 수 있다. 
         또한 내용을 이해하는데 파편화된다. 

### Test Fixture 클렌징

* @AfterEach tearDown메서드 내
  * deleteAllInBatch 
    * 삭제하는 순서를 고려해야한다.
    * Order<-> OrderProduct<-> Product Entity가 있을때, OrderProduct가 Order와 Product의 PK를 FK로 가지고 있으므로 OrderProduct관련 Repository부터 삭제해야한다.
  * deleteAll
    * 위의 예에서 OrderProductRepository를 삭제하지 않고 Order과 Product관련 Repository에 관해서 deleteAll을 해도 OrderProductRepository가 알아서 삭제된다. 
      * 그러나, 위에서 Product와 OrderProduct간에 양방향 매핑이 안되있는 경우, productRepository.deleteAll만 하는 경우 예외가 발생한다.
    * 즉, 연관관계가 있는 모든 Entity를 조회 후 건건이 삭제한다. 
    * 쿼리가 많아진다. 
    * Entity 관계가 복잡한 경우, 즉 given 절이 많으면 성능차이가 발생할 수 있는 포인트가 있다. 

### @ParameterizedTest

* 하나의 테스트케이스인데, 값을 바꿔가면서 테스트 해보고 싶을 때 사용



### @DynamicTest

* 일련의 시나리오를 검증하고 싶을 때 사용한다 
* 아래와 같은 방식으로 구성

```java
 @DisplayName("")
    @TestFactory
    Collection<DynamicTest> dynamicTest() {
        return List.of(
                DynamicTest.dynamicTest("",() -> {

                }),
                DynamicTest.dynamicTest("", () -> {

                })
        );
    }

// 위와 같은 형식으로 지정
// Collection뿐 아니라 iteratable한 모든 것이 다 가능



    @DisplayName("재고 차감 시나리오")
    @TestFactory
    Collection<DynamicTest> stockDeductionDynamicTest() {
        //given
        Stock stock = Stock.create("001", 1);

        return List.of(
                DynamicTest.dynamicTest("재고를 주어진 개수만큼 차감할 수 있다.", () ->{
                    //given
                    int quantity = 1;

                    //when
                    stock.deductQuantity(quantity);

                    //then
                    assertThat(stock.getQuantity()).isZero();
                }),
                DynamicTest.dynamicTest("재고보다 많은 수의 수량으로 차감 시도하는 경우 예외가 발생한다.", () ->{
                    //given
                    int quantity = 1;

                    //when, then
                    assertThatThrownBy(() -> stock.deductQuantity(quantity)).
                            isInstanceOf(IllegalArgumentException.class).hasMessage("차감할 재고 수량이 없습니다.");

                })
        );
    }
```



### 테스트 수행도 비용이다. 환경 통합하기

* IntegrationTestSupport
  * abstract Class , @SpringBootTest, @ActiveProfies 등 설정하고, 해당 설정을 상속받는 구조로 작성

* @DataJpaTest
  * Repository 관련된 Bean만 띄워주고 트랜잭션을 지원하지만, Service Layer Test 수행시 같이 띄워놓는다면, Spring boot 서버 자체가 실행되는 횟수를 줄일 수 있다. 
  * Repository Bean만 반드시 생성해야하는 상황이 아니라면 @SpringBootTest를 사용하는 것이 더 적절할 수 있다.



### private 메서드의 테스트는 어떻게 하나? 

* 할 필요 없다. 
  * 고민할 사안은 딱 하나이다.
  * 객체가 공개한 public 메서드에 대해서 검증(테스트 코드 작성)을 하면 알아서 내부적으로 private 메서드 자체에 대해서 검증하게된다 
  * 그러나, 해당 메서드의 기능에 대해서 아래와 같은 고민을 해봐야한다. 

* ##### 객체를 분리할 시점인가? 

  * 만약 객체를 분리한다면, 해당 객체에 대한 테스트 코드를 작성해주면 된다. 

### 테스트에서만 필요한 메서드가 생겼는데, 프로덕션 코드에서는 필요 없다면? 

```java
@Builder
    private ProductCreateRequest(final ProductType type, final ProductSellingStatus sellingStatus, final String name, final int price) {
        this.type = type;
        this.sellingStatus = sellingStatus;
        this.name = name;
        this.price = price;
    }



    public ProductServiceCreateRequest toServiceRequest() {
        return ProductServiceCreateRequest.builder()
                .type(type)
                .sellingStatus(sellingStatus)
                .price(price)
                .name(name)
                .build();
    }
```

Layer간 DTO를 분리하면서, toServiceRequest만 프로덕션에서 이용하고, 테스트에서만 builder 생성자가 이용된다면?

* 사용해도 된다, 그러나 보수적으로 접근해야한다. 
  * 무엇을 테스트하는가? 
  * getter, 기본생성자, 빌더, size 등은 거의 기본적으로 필요한 것들일 수 있다.
  * 어떤 객체가 마땅히 가져도 되는 것이면서 동시에 미래에도 사용될 수 있는 성격의 것들은 가져도 될 것이다. 



