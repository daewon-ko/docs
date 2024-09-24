**OrElse / OrElseGet의 차이에 대하여**

\#docs/Java



**배경**

테스트 코드 작성 중, Optional의 값을 반환하는 중 Get으로 꺼내기에는 null처리에서 부담이 있어서 관련 메서드 체이닝을 확인하다가 OrElseGet, OrElse라는 메서드를 확인했다. 무엇으로 테스트 코드를 작성해주는게 좋을까라는 생각이 들다가, 가만히 보니 둘의 차이점을 잘 모르는 것 같았다. 이번 기회에 정리해본다.



**OrElse**

>  public T orElse(T other)

- T의 모든 매개변수를 사용

**OrElseGet**

> public T orElseGet(Supplier<? extends T> Other)



- T를 상속하는 클래스를 반환하는 Supplier 유형의 인터페이스 허용
  - Supplier 메서드의 경우, Optional의 값이 없을때만 인자로 제공된 Supplier 메서드가 실행된다.
  - ![image-20240924221414113](/Users/daewon/Library/Application Support/typora-user-images/image-20240924221414113.png)

**OrElse에서 Null이 아니어도 로직이 호출되는 이유**

- OrElse메서드를 사용하다보면, Optional에서 null이 아닌 값이 조회하는되도 불구하고 orElse내부에서 정의한 메서드 등이 호출되는 경우가 있다.

- orElse는 T를 Return하는 메서드를 호출 시, 내부 로직이 실행된다.
- orElseGet는 내부의 값이 null일경우에만 내부로직이 실행된다.
  - 이는 내부적으로 orElseGet의 경우 supplier.get()메서드를 통해 메서드가 실행되기 때문이다.
- 따라서 동일한 메서드를 실행한다해도 orElseGet이 orElse보다 성능상으로 앞선다.



**주의점**

위의 예시가 메서드 내부에서 출력을하는 예제라 좀 덜 직관적일 수 있으나, 만약 다음과 JPA를 사용하는데 Entity를 변경하는 로직이 들어있다고 가정하면 어떨까? 가장 직관적인 User Entity를 가정해보자

~~~java
public User findByUsername(String username){

return userRepository.findByUsername(username).orElse(createUserWithName(username));}



public User createUserWithName(String username){

User user = new User(username);

userRepository.save(username);

}

~~~



UserRepository에서 조회한 결과가 null이 아님에도 불구하고 createUserWithName 내부의 로직이 실행됨에 따라서 불필요한 User가 생성될 수 있다. 그런데 만약 username이 비즈니스 조건에 따라서 Unique 제약 조건이 걸려있다면 당연히 Exception이 발생할 것이다. 



**결론**

OrElse / OrElseGet의 차이점을 잘 알고써야하지만, 개인적인 생각에는 OrElse내부에는 T객체를 직접 리턴하는 방식으로 작성하고, 메서드 호출은 최대한 지양해야할 것 같다. 메서드 호출을 방식으로 작성하려면 반드시 orElseGet으로 작성하는 것이 본래 코드의 목적에 부합할 듯 싶다. 



참고

[블로그1](https://giron.tistory.com/153)

[블로그2](https://cfdf.tistory.com/34)