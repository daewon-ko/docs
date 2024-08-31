#  @PostConstruct, @Transactional은 왜 같이 사용할수 없는가?



### 배경

@PostConstruct, @Transcational의 경우 Spring 컨테이너의 라이프 사이클때문에 아래와 같이 함께 사용할 수 없다는 이야기를 들었다. 이유가 궁금했다. 



~~~java
package controller;

import jakarta.annotation.PostConstruct;
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;
import study.querydsl.entity.Member;
import study.querydsl.entity.Team;

@Profile("local")
@Component
@RequiredArgsConstructor
public class InitMember {

    private final InitMemberService initMemberService;

    @PostConstruct
    @Transactional
    public void init() {
        initMemberService.init();
          Team teamA = new Team("Team A");
            Team teamB = new Team("Team B");
            em.persist(teamA);
            em.persist(teamB);

            for (int i = 0; i < 100; i++) {
                Team selectedTeam = i % 2 == 0 ? teamA : teamB;
                em.persist(new Member("member" + i, i, selectedTeam));
            }
    }

    @Component
    static class InitMemberService{
        @PersistenceContext
        private EntityManager em;



    }

}


~~~



즉, 위와 같이 실행할경우 문제가 발생할 수 있다 설명이다. 



### 스프링 컨테이너 및 어노테이션의 적용시점

스프링 컨테이너는 애플리케이션 내의 빈을 생성하고, 의존성을 주입한 후 초기화한다. @PostConstructor의 경우, 스프링 빈이 모두 초기화 된 이후에 호출된다. 즉, @PostConsructor가 호출되는 시점은 빈이 모두 초기화되기 전일 수 있다. 



@Transactional의 경우, 해당 어노테이션이 붙은 메서드의 트랜잭션이 보장되지만, 해당 트랜잭션이 보장되기 위해선 프록시 객체를 요구한다. 그러나 문제는, 이러한 프록시 기술의 사용을 위해서는 빈이 완전히 초기화된 이후에 사용가능하다는 점이다. 따라서 @PostConstructor와 @Transactional의 경우 같이 사용할 경우에 문제가 발생할 수 있다. 따라서 아래와 같이 분리하는 것이 적절하다. 



~~~java
package controller;

import jakarta.annotation.PostConstruct;
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;
import study.querydsl.entity.Member;
import study.querydsl.entity.Team;

@Profile("local")
@Component
@RequiredArgsConstructor
public class InitMember {

    private final InitMemberService initMemberService;

    @PostConstruct
    public void init() {
        initMemberService.init();
    }

    @Component
    static class InitMemberService{
        @PersistenceContext
        private EntityManager em;
        @Transactional
        public void init() {
            Team teamA = new Team("Team A");
            Team teamB = new Team("Team B");
            em.persist(teamA);
            em.persist(teamB);

            for (int i = 0; i < 100; i++) {
                Team selectedTeam = i % 2 == 0 ? teamA : teamB;
                em.persist(new Member("member" + i, i, selectedTeam));
            }
        }

    }

}

~~~



