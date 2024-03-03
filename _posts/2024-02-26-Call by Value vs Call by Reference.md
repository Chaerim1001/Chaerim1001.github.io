---
title: Call by Value vs Call by Reference
date: 2024-02-26
categories: [Java]

tags: [Java]
---

## 메소드의 파라미터 전달 방법
메소드를 호출할 때 파라미터 값을 전달하는 방법은 두 가지가 있습니다.
하나는 `Call by Value` 이고 다른 하나는 `Call by Reference` 입니다.

### Call by Value
`Call by Vaule`는 메소드를 호출할 때 데이터 값을 넘겨주는 방식을 뜻합니다. (`Pass by Value` 라고도 부릅니다.) 

호출자의 변수와 수신자의 변수는 **서로 다른 변수**이며, 서로 다른 변수이기 떄문에 메소드 내에서 해당 변수를 조작하더라도 호출자에서 사용한 변수의 값은 변하지 않습니다.

### Call by Reference
`Call by Reference`는 메소드를 호출할 때 참조(주소)를 직접 전달하는 방식을 뜻합니다. (`Pass by Value` 라고도 부릅니다.)

호출자의 변수와 수신자의 변수는 **같은 변수**이며, 완전히 동일한 변수이기 떄문에 메소드 내에서 해당 변수를 조작하면 호출자에서 사용한 변수도 함께 바뀌게 됩니다.

## Java에서의 파라미터 전달 방법
Java는 오직 `Call by Value`로만 동작합니다.

Java를 사용하면서 메소드 호출 시 파라미터로 전달한 값이 변경된 경험을 한 적이 있으시죠?! 하지만 Java가 `Call by Value`로만 동작한다는 사실은 변함이 없습니다.

왜 그런지 알아볼까요?!

### JVM
Java에서는 변수를 선언하면 JVM의 Stack 영역에 할당됩니다.

![jvm](/assets/img/posts/2024-02-26/jvm.png){: width="500" }

- 원시 타입(Primitive Type)은 Stack 영역에 변수와 함께 저장됩니다.
- 참조 타입(Reference Type) 객체는 Heap 영역에 저장되고 Stack 영역에 있는 변수가 객체의 주소값을 갖고 있습니다.

Reference 자체를 전달하는 것이 아니라 Stack 영역에 가지고 있는 **주소 값**만 전달합니다. 그렇기 때문에 `Call by Reference`가 아닌 `Call by Value` 인거죠!!

### 원시 타입(Primitive Type) 테스트

```java
import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.Test;

class PrimitiveTypeTest {

    @Test
    void test() {
        int a = 1;
        char b = 'b';

        // Before
        assertThat(a).isEqualTo(1);
        assertThat(b).isEqualTo('b');

        modify(a, b);

        // After
        assertThat(a).isEqualTo(1);
        assertThat(b).isEqualTo('b');
    }

    void modify(int a, char b) {
        a = 2;
        b = 'c';
    }
}
```
코드 내용을 그림으로 나타내면 아래와 같습니다.

![stack1](/assets/img/posts/2024-02-26/stack1.png){: width="600" }

### 참조 타입(Reference Type) 테스트

코드를 보면 node 인스턴스의 value는 1이 아닌 값으로 바꼈지만 주소는 전과 동일한 것을 확인할 수 있습니다.

```java
import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.Test;

class ReferenceTypeTest {

    @Test
    void test() {
        Node node = new Node(1);
        int address = identityHashCode(node);

        // Before
        assertThat(node.value).isEqualTo(1);

        modify(node);

        // After
        assertThat(node.value).isNotEqualTo(1);
        assertThat(identityHashCode(node)).isEqualTo(address);

        System.out.println("address: " + identityHashCode(node));
    }

    void modify(Node node) {
        node.value = 100;
    }

    static class Node {
        int value;

        public Node(int value) {
            this.value = value;
        }
    }
}
```

`identityHashCode(node)`를 사용하면 node 객체의 주소 값을 확인할 수 있습니다.

![test](/assets/img/posts/2024-02-26/test.png){: width="600" }

코드 내용을 그림으로 나타내면 아래와 같습니다.

![stack2](/assets/img/posts/2024-02-26/stack2.png){: width="900" }

주소 값이 전달되므로 **같은 객체를 가리키게** 되기 때문에 value 값이 수정되는 것입니다.

<br>

> 참고
> https://www.baeldung.com/java-pass-by-value-or-pass-by-reference
> https://bcp0109.tistory.com/360