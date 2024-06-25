# Validation

* @Notnull
  * " ","" -> 공백 등은 통과한다.
  * Enum 값 등 검증할때 자주 사용할 수 있다. 
  * Enum은 Null값만 올 수 있기에

* @NotBlank
  * 공백이어도 안되고, 문자가 반드시 존재해야한다.
  * String 같은 경우에 많이 사용

* @NotEmpty
  * " " -> 공백은 통과
* @Positive
  * 가격 등에 Validation 가능