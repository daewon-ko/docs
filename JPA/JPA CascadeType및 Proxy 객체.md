# JPA Proxy 및 CascadeType 및 Option(1/2)

### 배경

JPA를 학습하다보면 어려운 개념을 맞닥들이곤 한다.

사람마다 무엇이 어려운가는 차이가 있을 수 있지만, 본인의 경우엔 Entity 매핑 및 CasCade Option, Proxy에의 이해가 잘 가지 않았다. 그래서 이번에 한 번 정리를 하면서 해당 개념에 대해서 조금 더 깊이 있는 이해를 도모해보고자 한다.



### Lazy Loading(지연 로딩)

Entity와 Entity는 연결되어있다. JPA에서는 객체그래프(Entity Graph)라는 것을 통해 연결된들의 정보를 조회할 수 있다. 

그런데, Entity끼리 연결된 객체 그래프의 규모가 클 경우에 하나의 Entity를 조회하는데, 연관된 모든 Entity를 조회하는 것은 비효율적이다. 
따라서 Entity의 모든 정보를 조회하는 것이 아니라, <b>Entity가 실제 사용될때까지 DB에서 조회를 지연하는 방법을 제공하는데 이것을 지연로딩</b>이라고 한다.

그런데, 지연로딩 기능을 사용하기 위해서는 실제 Entity 객체 대신 DB 조회를 지연할 수 있는 가짜 객체가 필요한데 이것을 프록시 객체라고 한다. 

### Proxy

em.find(Meber.class, "meber1"); 와 같이 조회할 시에는 실제 Entity 객체가 반환된다. 

영속성 컨텍스트 내에 해당 객체가 존재하고 있다면 DB를 직접조회하지 않을 수 있지만, 영속성 컨텍스트가 Flush한 이후 혹은 해당 트랜잭션 내에 처음 조회시엔 반드시 DB를 조회하게 된다. 

그러나, em.getReference(Member.class, "member1");와 같이 조회하면 Proxy 객체가 반환된다. 

Proxy 객체는 실제 Entity를 사용하는 시점까지 DB에 접근하지 않는다. 

JPA는 DB를 조회하지 않고, 실제 Entity 객체를 반환하지 않는다. 

다음의 코드를 보자.

~~~java


 public static void main(String[] args) {

        EntityManagerFactory emf = Persistence.createEntityManagerFactory("hello");
        EntityManager em = emf.createEntityManager();

        EntityTransaction tx = em.getTransaction();
        tx.begin();
        //code

        try {

            Member member = new Member();
            member.setUserName("Hello");
            em.persist(member);

            em.flush();
            em.clear();


            Member proxy = em.getReference(Member.class, member.getId());
            System.out.println("proxy.getClass() = " + proxy.getClass());

            tx.commit();

        } catch (Exception e) {
            tx.rollback();
        }

        finally {
            em.close();
        }
        emf.close();
    }


~~~



![image-20240723132301491](/Users/daewon/Library/Application Support/typora-user-images/image-20240723132301491.png)

영속성 컨텍스트를 flush 및 clear한 이후엔 getReference 메서드로 조회 시 HibernateProxy 객체를 반환한다.

그러나, clear를 하지 않는다면 어떻게 될까? 

영속성 컨텍스트를 clear하지 않는다면, 영속성 컨텍스트의 1차 캐시 기능으로 인해서 영속성 컨텍스트에 해당 Entity가 남아있게 된다. 

따라서 위의 경우와 달리 아래와 같이 proxy가 아닌 Member 클래스가 로그에 찍히게 된다.

![스크린샷 2024-07-23 오후 1.27.02](/Users/daewon/SchreenShot/스크린샷 2024-07-23 오후 1.27.02.png)

프록시 클래스는 실제 클래스를 상속받으므로 실제 클래스와 겉모양이 동일한다. 

프록시 객체는 마치 합성과 같이, 실제 원본 객체(Entity)의 참조를 보관하고 있다. 또한, member.getUserName()처럼 실제 메서드가 사용될때, DB를 조회해서 실제 Entity 객체를 생성하며 이것을 프록시 객체의 초기화라고 한다.

~~~~java
            Member member = new Member();
            member.setUserName("Hello");
            em.persist(member);

            em.flush();
            em.clear();



            Member proxy = em.getReference(Member.class, member.getId());
            System.out.println("proxy.getClass() = " + proxy.getClass());
            System.out.println("proxy.getUserName() = " + proxy.getUserName());
            System.out.println("proxy.getClass( = " + proxy.getClass());
            tx.commit();
~~~~



![image-20240723140051820](/Users/daewon/Library/Application Support/typora-user-images/image-20240723140051820.png)

상단의 코드와 로그를 보면 proxy.getClass()를하는 시점까지는 Proxy객체가 확인된다.

그러나,  proxy.getUserName()를 하는 순간, 프록시 객체의 초기화가 요청된다. 

따라서 로그를 보면 조회쿼리가 DB에 날라간다. 

프록시 객체의 초기화는 영속성컨텍스트를 통해서 이뤄지며, 영속성 컨텍스트는 DB를 조회해서 실제 Entity 객체를 생성한다. 

프록시 객체는 생성된 실제 Entity 객체의 참조를 Proxy 내부에 멤버변수에 보관한다. 

따라서 영속성 컨텍스트를 통해 프록시의 초기화가 이뤄진 이후에 다시 한번 proxy.getClass()를 출력해도 똑같이 Proxy객체가 출력된다. 

즉 Proxy 객체가 Entity 객체로 변환되는 것이 아니라, 프록시 객체를 통해서 실제 Entity 객체에 접근할 수 있는 것이다. 

또한 Proxy 객체는 원본 엔티티를 상속받은 객체이므로 타입체크시 주의가 필요하다. 

만약, 영속성 컨텍스트가 close()된 이후에(준영속 상태에서) Proxy객체의 메서드를 수행할 시,

Hibernate의LazyInitializationException이 발생한다. 



### 식별자 조회시, 프록시는 초기화 되지 않는다. 

Proxy 객체는 DB를 조회하지 않았음에도 어떻게 식별자를 갖고 올 수 있는가? 와 같은 의문에 대해서 어떻게 답할 수 있을까? 

해당 과정의 매커니즘에 관한 이해에 선행이 필요하다. 

AbstractLazyInitializer는 멤버변수로 식별자(id)값을 갖고있으며 getIdentifier 메서드를 통해서 id값을 반환한다.

getIdentifier 메서드가 id를 반환하는 조건 중 하나는 hibernate.jpa.compliance.proxy가 true일때지만, 기본값은 false이다. 따라서 일반적으로는 id의 getter를 호출해도 프록시는 초기화 되지 않는다. 

따라서 getId와 같이, 식별자에 대해서 getter를 호출해도 프록시는 호출되지 않는다. 다만, 똑같이 id를 Return하지만, 메서드 명을 findId와 같이 자바빈 규약에서 어긋나는 이름으로 메서드를 작성할 경우, getIdentifier 메서드는 호출되지 못하며 프록시는 자동적으로 초기화된다. 

추가적으로 Proxy 객체가 id값을 직접적으로 참조하고 있는 것이 아니라, 프록시 내부의 인터셉터(ByteBuddyInterceptor)가 id값을 들고있으며 프록시가 들고있는 값은 모두 null이다. 하단의 그림을 참조하자. 

![image-20240724101613531](/Users/daewon/Library/Application Support/typora-user-images/image-20240724101613531.png)





