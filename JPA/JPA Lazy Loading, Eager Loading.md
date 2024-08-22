# JPA 즉시로딩과 지연로딩

### 배경

저번 글에 작성했듯이, JPA를 이해하는데 주된 장애물 중 하나가 즉시로딩과 지연로딩의 개념의 차이이다.  그리고 그것에서 비롯되어 각각의 로딩방식이 다양한 연관관계의 방식과 연계되어 쿼리가 생각지도 못하는 쿼리가 날라가는 등.. 로딩방식은 JPA라는 산을 정복하는데 있어서 큰 장애물 중 하나라고 할 수 있다. 생각이 난김에 한 번 정리해보기로 한다. 

### 즉시로딩

즉시로딩은 엔티티를 조회할 때 연관된 엔티티를 함께 조회한다. Team과 Member라는 Entity가 있을때, 

~~~java
@Entity
public class Member{

@ManyToOne(fetch = FetchType.EAGER)
@JoinColumn(name= "team_id")
Team team;

기타 생략..
}
~~~

> em.find(Member.class, "memberId");

위와 같이 조회 시, Member와 연관관계가 있는 Team까지 한번에 조회가 된다. 

대부분의 JPA 구현체의 경우 즉시로딩 사용 시에, 연관된 Entity를 조회하기 위해서 Join을 사용한다. 

![image-20240724112347788](/Users/daewon/Library/Application Support/typora-user-images/image-20240724112347788.png)

위에서 보다시피, @ManyToOne Option 내에 fetch = FetchType.EAGER로 정의한 것 이외에 다른 옵션은 없다. 즉 즉시로딩의 경우, 로그를 보면 알 수 있다시피 기본 옵션이 Outer Join이다. 이 경우에 문제는, 위의 예제의 Member <-> Team의 관계에서 생각해보면, Team이 없는 Member역시 조회된다. 

JPA의 기본스펙이 즉시로딩일 시, Join을 하게 되면 Outer Join이지만, 통상 Outer Join보다는 Inner Join이 성능최적화의 여지가 더 많다. 또한 비즈니스 요구사항에 따라서 위의 예제에서 Team이 없는 Member만을 조회하기 위해서는 반드시 Inner Join이 필요하다. 

Inner Join은 다음과 같이 가능하다.

~~~~~~java
@Entity
public class Member{

@ManyToOne(fetch = FetchType.EAGER)
@JoinColumn(name= "team_id", nullable = false)
Team team;

기타 생략..
}
~~~~~~

위와 같이 FK에 nullable 옵션을 주게되면, 참조무결성 원칙에 의해서 Null이 불가능하므로 Inner Join을 JPA가 수행하게된다. 



### 컬렉션에 즉시로딩 사용시 주의점

* 컬렉션을 하나 이상 즉시로딩하는 것은 권장되지 않는다.

​	컬렉션 조인은 대상 테이블이 '다'인 관계이다. 

​	다대다, 일대다를 생각하면 된다. 위의 예제에서 Team의 관점에서 Member는 '다'이므로 이 예제에 적합하다. 

​	문제는 Team이 Member의 관계뿐 아니라 다른 Entity에 대해서 '다'관계를 형	성한다면, Team 조회 시에 Member의 개수 *  다른 Entity의 개수 만큼 Data를 	조회하는 개수가 급격하게 늘어난다. 이에따라 성능이 저하될 수 있다. 

#### 	따라서 '다'의 관계에 대해서는 2개 이상의 컬렉션에 대해 즉시로딩을 	사용하는 것이 가급적 권장된다.

* 컬렉션 즉시 로딩은 항상 외부조인을 사용한다.



### 지연로딩



Proxy를 통해서 지연로딩이 가능하며, 연관된 Entity의 실제 메서드 등이 사용될때 조회한다. 

> Member member = em.find(Member.class,"memberId");
>
> member.getTeam().getTeamName();

위와 같이, getTeamName()을 조회할 때 Team Entity를 DB에서 조회한다. 



### JPA 기본 페치 전략

* @ManyToOne, @OneToOne : 즉시로딩
* @OneToMany, @ManyToMany : 지연로딩

JPA 기본 페치 전략이다. 

도메인 결합도 및 비즈니스 요구사항에 따라서 Entity 조회 시 함께 조회하는 것이 효율적일 경우에 즉시로딩을 사용하는 방법이 좋지만, 기본적으로 모든 연관관계에 대해서 지연로딩으로 설정 후 최적화 시에 즉시로딩 등을 적용하는 것이 추천된다. 





