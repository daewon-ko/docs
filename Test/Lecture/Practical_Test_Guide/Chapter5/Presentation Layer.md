# Presentation Layer

### Mock

- 의존 관계를 지니고 있는 것들을 가짜로 처리
- MockMvc 프레임 워크
  - 스프링에서 제공
  - Mock(가짜) 객체를 사용해 스프링 MVC동작을 재현할 수 있는 테스트 프레임워크

### 요구사항

> 1. 관리자 페이지에서 신규 상품을 등록할 수 있다.
>
> 2. 상품명, 상품타입, 판매상태, 가격 등을 입력받는다.
>
>    

* 관리자 페이지에서 상품등록 하는 기능의 경우또한 동시성문제 발생가능(재고문제와 동일)
  - 어떻게 풀 수 있을까?
    - 상품을 여러 명이 등록하는 경우?
    - productNumber DB field에 Unique Index를 걸고, (제약조건), 재시도하는 로직을 추가할 수 있다.
      - Unique한 조건때문에 튕기게 되면, 3회 이상 재시도를 하게 한다든가? (양이 많지 않아서 크리티컬하지 않은 경우 이렇게 Application 단에서 처리도 가능)
    - 크리티컬한 케이스 → 동시접속자가 너무 많은경우
      - productNumber 자체를 증가하는 값이아니라, UUID와 같은 경우를 활용가능하다.
        - 상품 번호 자체가 동시성과 무관하게 풀 수 있다.

### @Transactional

- readOnly Option → 기본이 False
- true값을 주면, 읽기 전용 트랜잭션이 열림
- Master DB → 쓰기전용
- (Replica) Slave DB → 읽기전용

→ Write Db에 접근할 수 있는 URL과 읽기 DB에 접근할 수 있는 URL을 분리한다.

→ readOnly = true가 걸리면 → 읽기전용으로 보내자.(slave)

또한, readonly= false가 기본값이면, 커맨드 작업에 대한 것을 마스터 DB로 보내자.

→ 이에 따라 장애격리를 할 수 있는 포인트가 된다.

AWS Aurora → DB 클러스트 모드 같은 것을 사용하면, 같은 EndPoint로 보내도,  Read-only값으로 비교를 해준다.

또는 스프링에서 Read-only값을 보고 end-point를 달리 줄 수도 있다.



### @WebMvcTest

* Controller 관련 Bean만 띄워주는 Test



### Validation

* Domain 정책이 변경되어서 상품 이름은 20자 이상이라는 조건이 있다면, 
  해당 Valdation을 Presentation Layer(ProductCreateRequest에서 Validation Annotation으로 검증)하는 것이 적절할까? 
  * @NotBlank, @NotEmpty등은 유효하다. 
  * 그러나 해당 조건(글자 수)의 경우 서비스 레이어에서 검증하든, 도메인 객체를 생성하는 시점에서 생성자에서 검증을 하든 안쪽 Layer에서 검증하는 것이 좋다. 
  * 즉 Validation의 성격이 다르다. 