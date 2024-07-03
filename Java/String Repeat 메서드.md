# String repeat 메서드

* 자바 11부터 이용
* 아래와 같이 사용

```java
    public static void recursive(int n, int depth) {
        String underBar = "____".repeat(depth);

        System.out.println(underBar + "\"재귀함수가 뭔가요?\"");
        
        if (n == depth) { // 기본 단계
            System.out.println(underBar + "\"재귀함수는 자기 자신을 호출하는 함수라네\"");
        } else { // 귀납 단계
            System.out.println(underBar + "\"잘 들어보게. 옛날옛날 한 산 꼭대기에 이세상 모든 지식을 통달한 선인이 있었어.\"");
            System.out.println(underBar + "마을 사람들은 모두 그 선인에게 수많은 질문을 했고, 모두 지혜롭게 대답해 주었지.");
            System.out.println(underBar + "그의 답은 대부분 옳았다고 하네. 그런데 어느 날, 그 선인에게 한 선비가 찾아와서 물었어.\"");
            recursive(n, depth + 1);
        }
        
        System.out.println(underBar + "라고 답변하였지.");
    }
```

> 백준 재귀함수가 뭔가요? 문제 

underBar를 재귀적으로 정의해야하는 상황에서 

underBar+="____";와 같이 작성하지 않고 위와 같이 repeat 메서드를 사용할 수 있다. 