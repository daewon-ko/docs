# List의 subList 메서드

### 배경
코테를 풀던 도중, List 자료구조에서 특정 요소만 추출한 새로운 List를 만들고 싶었다.

처음에 생각한 것은, 새로운 List를 만드는게 아니라 remove()메서드를 이용하는 방법이었는데 너무 원시적인 것 같아서 구글링을 해봤다.

### 사용예시

```java
List<Integer> list = List.of(1,2,3,4,5);

// 0번 Index부터 2번 Index까지 추출한다.
List<Integer> sub = list.subList(0,3); 
for(Integer i : sub){
	System.out.printf(i+" "); // 1 2 3
}
```



참고

https://docs.oracle.com/javase/8/docs/api/java/util/List.html