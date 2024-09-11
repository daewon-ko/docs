# Gradle 사용시 주의점

### dependencies 추가 시 주의점

* implementation과 api의 차이점에 대해서

  * implementation
    * 해당 모듈에만 의존성을 사용가능
    * 해당 모듈을 의존하는 다른 모듈에서는 사용불가
  * api
    * 해당 모듈뿐 아니라 해당모듈을 의존하는 타 모듈에서도 모듈을 이용가능
  * 즉 아래와 같은 상황에서 openfeign을 의존할때, openfeign을 의존하는 모듈을 또 다른모듈에서 의존하게끔 하려면 implementation이 아닌 api로 작성해줘야함

  ~~~
  dependencies {
      implementation project(':common')
      api 'org.springframework.cloud:spring-cloud-starter-openfeign'
  }
  ~~~

  CF) 상기의 기능은 java-library라는 기능을 통해 가능

  Q) Java-library란?

  > #### `java-library`
  >
  > - Java Library 플러그인은 Java 플러그인과 비슷하지만, API와 내부 구현을 명확하게 구분
  >
  >   할 수 있는 추가 기능을 제공
  >
  >   - `implementation`과 `api` 키워드를 사용해 의존성 범위를 세분화하여, 모듈 간에 어떤 의존성이 공개되고 감춰질지를 관리할 수 있습니다. 이것은 특히 라이브러리 모듈을 개발할 때 유용
  >   - `java-library`는 재사용 가능한 모듈 또는 라이브러리를 만들 때 적합