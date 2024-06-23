# META-INF, persitence.xml



Maven 사용시 이용되는 설정값들은 Gradle 사용 시 어떻게 전환? 혹은 이용될까?

* META-INF 내에 persistence.xml 파일을 이용하면 가능하다.
* Persistence.createEntityManagerFactory 메서드를 이용하기 위해는 persistence.xml 파일이 필요하다. 
* Spring-data-jpa를 이용하는 경우엔, EntityManager, EntityManagerFactory를 직접 핸들링 할 이유가 없지만, EntityManager, EntityManagerFactory의 경우엔 다르다. 
* 

