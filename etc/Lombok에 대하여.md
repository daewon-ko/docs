# Lombok
* Lombok이 @Qualifier와 함께 사용시 아래와 같은 경고문이 발생하는이유는?
> Lombok does not copy the annotation 'org.springframework.beans.factory.annotation.Qualifier' into the constructor 
* Lombok은 @AllArgsConsructor, @NoArgsConstructor와 같이 자동으로 생성자를 만들지만, 그 과정에서 메서드나 필드에 붙어있는 어노테이션을 생성된 파라미터로 복사하지 않는다.
  * @Qualifier는 DI 과정에서 특정 빈을 지정하여 주입하기 위해 사용되는데, Lombok으로 생성자를 만들면, 필드에 붙어있던 @Qualifier 주석까지 생성자 파라미터로 복사되지 않기 때문에 빈을 구분할 수 없다.