# 고급 매핑

### 상속관계 매핑

* RDB는 상속 관계 X

* 슈퍼타입 서브타입 관계라는 모델링 기법이 객체 상속과 유사

* 상속관계 매핑 : 객체의 상속과 구조와 DB의 슈퍼타입 서브타입 관계를 매핑

* 슈퍼타입 서브타입 구현 방법(DB관점에서 봤을 시)
  1. 조인전략
  
     * @Inheritance(strategy = InheritanceType.JOINED)
     * @DiscriminatorColumn
       * 있어야 해당 타입을 상속받는 Entity의 타입 구분 가능
       * 관례상 컬럼에 DTYPE이라는 이름으로 들어감
       * 없어도 상관은 없음
  
  2. 단일테이블전략
  
     * @Inheritance(strategy = InheritanceType.SINGLE_TABLED)
       * 하나의 테이블에서 관리하므로 insert, select 쿼리 시 한 번에 나감
       * 성능 걱정할 필요는 없음
  
  3. 구현 클래스마다 테이블 전략
  
     * @Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
  
     * 단순 조회는 심플하지만, 다형성을 이용하여 부모타입으로 조회 시 문제가 됨.
  
       ```java
       package hellojpa;
       
       import jakarta.persistence.*;
       
       import java.util.List;
       
       public class JpaMain {
       
           public static void main(String[] args) {
       
               EntityManagerFactory emf = Persistence.createEntityManagerFactory("hello");
               EntityManager em = emf.createEntityManager();
       
               EntityTransaction tx = em.getTransaction();
               tx.begin();
               //code
       
               try {
                   // 저장
       
       
                   Movie movie = new Movie();
                   movie.setDirector("aaa");
                   movie.setActor("bbb");
                   movie.setName("바람과 함께 사라지다.");
                   movie.setPrice(1000);
       
                   em.persist(movie);
       
                   em.flush();
                   em.clear();
       
       //            em.find(Movie.class, movie.getId());
                   Item item = em.find(Item.class, movie.getId());
                   
       
       
                   tx.commit();
               } catch (Exception e) {
                   tx.rollback();
               } finally {
                   em.close();
               }
               emf.close();
           }
       }
       
       // find 시 날라가는 쿼리
       /**
       *Hibernate: 
           select
               i1_0.id,
               i1_0.clazz_,
               i1_0.name,
               i1_0.price,
               i1_0.artist,
               i1_0.author,
               i1_0.isbn,
               i1_0.Actor,
               i1_0.director 
           from
               (select
                   price,
                   id,
                   artist,
                   name,
                   null as author,
                   null as isbn,
                   null as Actor,
                   null as director,
                   1 as clazz_ 
               from
                   Album 
               union
               all select
                   price,
                   id,
                   null as artist,
                   name,
                   author,
                   isbn,
                   null as Actor,
                   null as director,
                   2 as clazz_ 
               from
                   Book 
               union
               all select
                   price,
                   id,
                   null as artist,
                   name,
                   null as author,
                   null as isbn,
                   Actor,
                   director,
                   3 as clazz_ 
               from
                   Movie
           ) i1_0 
       where
           i1_0.id=?
       */
       
       ```
  
       위와 같이 다형성을 이용해서 조회 시, item을 찾을때 쿼리가 굉장히 복잡해진다. 
  
       
  
* JPA는 슈퍼타입 서브타입을 DB가 어떻게 구현하든 다 상관없음

---

### 상속관계 매핑 장단점

1. 조인전략
   * 장점
     * 테이블 정규화
     * 외래 키 참조 무결성제약조건 활용가능
       * <-
     * 저장공간 효율화
   * 단점
     * 조회 시 조인 많이 사용, 성능 저하
     * 쿼리 복잡, 저장 시 insert 쿼리가 2번 나간다.
       * <- 그렇게 성능저하가 많지는 않다. 
2. 단일테이블
   * 장점
     * 조인이 필요없으므로 조회 성능이 빠르고 쿼리가 단순
   * 단점
     * 자식 엔티티가 매핑한 컬럼은 모두 null 허용
     * 테이블이 커질 수 있고 조회 성능이 느려질 수 있음
3. 구현 클래스마다 테이블 전략
   * 쓰면 안된다.
   * 조회성능이 너무 느리고 자식 테이블을 통합하여 쿼리하기 어려움
   * 변경에 너무 힘들다. 



---

### @MappedSuperClass

* 공통 매핑 정보가 필요할때 사용
* Entity X, 테이블과 매핑 X
* 부모클래스를 상속받는 자식 클래스에 매핑 정보만 제공
* 조회, 검색 불가
* 추상클래스 권장
* CF) @Entity 클래스는 엔티티나 @MappedSuperClass로 지정한 클래스만 상속가능





