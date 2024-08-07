# Assertj의 다양한 용법

#Deepdive/ETC/TESTCODE/ASSERTJ

* **hasValueSatisfying**
```java
assertThat(findChatRoom)
        .isPresent()
        .hasValue(chatRoom)
        .hasValueSatisfying(
                room -> {
                    assertThat(room.getBuyer()).isEqualTo(buyer);
                    assertThat(chatRoom.getSeller()).isEqualTo(seller);
                    assertThat(chatRoom.getProduct()).isEqualTo(product);
                });

```

[AssertJ Docs](https://assertj.github.io/doc/#assertj-core-3.18.0-hasValueSatisfying)
