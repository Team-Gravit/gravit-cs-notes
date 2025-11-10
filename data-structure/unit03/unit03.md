## 스택과 큐(Stack & Queue)

**스택**(Stack)은 한쪽 끝에서만 데이터를 넣고 빼는 **LIFO**(Last In First Out) 구조의 자료구조. 가장 나중에 들어온 데이터가 가장 먼저 나온다.

<br>

**큐**(Queue)는 한쪽에서 삽입하고 반대쪽에서 삭제하는 **FIFO**(First In First Out) 구조의 자료구조. 가장 먼저 들어온 데이터가 가장 먼저 나온다.

<img src="image.png" width="50%">

<br>

### 1. 스택의 특징

- **후입선출(LIFO)** 구조
- **Top**에서만 삽입/삭제 발생
- **스택 포인터**(SP)로 Top 위치 관리

<br>

### 2. 스택의 주요 연산

| **연산** | **설명** | **시간 복잡도** |
| --- | --- | --- |
| **push()** | Top에 데이터 삽입 | O(1) |
| **pop()** | Top에서 데이터 제거 및 반환 | O(1) |
| **peek()** | Top 데이터 조회 | O(1) |
| **isEmpty()** | 스택이 비었는지 확인 | O(1) |

<br>

### 3. 스택 포인터(Stack Pointer)

스택 포인터는 **다음 값이 들어갈 위치**를 가리키며, 초기값은 **-1**이다.

```java
private int sp = -1;

// push
public void push(Object o) {
    if(isFull()) return;
    stack[++sp] = o;
}

// pop
public Object pop() {
    if(isEmpty()) return null;
    return stack[sp--];
}
```

<br>

### 4. 동적 스택

배열 기반 스택은 **크기가 고정**되므로, 가득 차면 배열 크기를 확장한다.

```java
if(isFull()) {
    Object[] arr = new Object[MAX_SIZE * 2];
    System.arraycopy(stack, 0, arr, 0, MAX_SIZE);
    stack = arr;
    MAX_SIZE *= 2;
}
```

<br>

**❗️연결리스트로 구현**하면 크기 제한 없이 사용 가능하다.

<br>

---

### 5. 큐의 특징

- **선입선출(FIFO)** 구조
- **Front**: 데이터가 나가는 위치
- **Rear**: 데이터가 들어오는 위치

<br>

### 6. 큐의 주요 연산

| **연산** | **설명** | **시간 복잡도** |
| --- | --- | --- |
| **enqueue()** | Rear에 데이터 삽입 | O(1) |
| **dequeue()** | Front에서 데이터 제거 및 반환 | O(1) |
| **peek()** | Front 데이터 조회 | O(1) |
| **isEmpty()** | 큐가 비었는지 확인 | O(1) |

<br>

### 7. 선형 큐

**Front**와 **Rear** 포인터로 **삽입**/**삭제** 위치를 관리한다. **Rear**에서 **삽입**(enqueue), **Front**에서 **삭제**(dequeue)가 발생한다.

<br>

**❗️단점**: **Rear**가 배열 끝에 도달하면 앞쪽에 빈 공간이 있어도 삽입이 불가능하다.

<br>

### 8. 원형 큐(Circular Queue)

배열의 처음과 끝을 논리적으로 연결하여 **선형 큐의 메모리 낭비 문제를 해결**한다. **(index + 1) % size**로 순환 구조를 구현하며, **공백**/**포화** 상태를 구분하기 위해 **한 자리를 항상 비워둔다**.

<br>

**❗️단점**: 배열 기반이므로 크기가 고정되어 있다.

<br>

### 9. 연결리스트 큐

노드를 연결하는 방식으로 구현하여 크기 제한이 없다. **Rear**에서 노드를 추가하고 **Front**에서 제거하므로 **삽입**/**삭제**가 O(1)로 효율적이다. **동적 크기 조정이 가능**하여 메모리를 효율적으로 사용할 수 있다.
