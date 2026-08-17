## 덱(Deque)과 우선순위 큐(Priority Queue)

**덱**(Deque, Double-Ended Queue)은 **양쪽 끝**에서 삽입과 삭제가 모두 가능한 자료구조이고, **우선순위 큐**(Priority Queue)는 들어온 순서와 관계없이 **우선순위가 높은 데이터**가 먼저 나가는 자료구조다. 두 자료구조 모두 큐(unit03)를 확장한 형태다.

<br>

### 1. 덱의 정의

덱은 큐의 **Front**와 **Rear** 양쪽 모두에서 **삽입과 삭제를 허용**하는 자료구조다. 한쪽 끝만 사용하면 스택처럼, 서로 반대쪽 끝을 사용하면 큐처럼 동작하므로 **스택과 큐를 일반화한 자료구조**라고 할 수 있다.

```
              Front                     Rear
                ↓                         ↓
   삽입 →   ┌────┬────┬────┬────┐   ← 삽입
            │ 10 │ 20 │ 30 │ 40 │
   삭제 ←   └────┴────┴────┴────┘   → 삭제
```

<br>

### 2. 덱의 주요 연산과 시간 복잡도

| **연산**          | **설명**                       | **시간 복잡도** |
| ----------------- | ------------------------------ | --------------- |
| **addFirst()**    | Front에 데이터 삽입            | O(1)            |
| **addLast()**     | Rear에 데이터 삽입             | O(1)            |
| **removeFirst()** | Front에서 데이터 제거 및 반환  | O(1)            |
| **removeLast()**  | Rear에서 데이터 제거 및 반환   | O(1)            |
| **peekFirst()**   | Front 데이터 조회              | O(1)            |
| **peekLast()**    | Rear 데이터 조회               | O(1)            |

❗️**중간 원소 접근**: 덱은 양 끝 연산에 최적화된 자료구조이므로, 중간 위치의 원소를 탐색하거나 삭제하는 연산은 O(n)이 걸린다.

<br>

### 3. 스택과 큐의 대체

덱은 사용하는 끝을 제한하는 방식으로 스택과 큐를 모두 대체할 수 있다.

| **대체 대상** | **삽입 연산** | **삭제 연산**     | **동작 방식** |
| ------------- | ------------- | ----------------- | ------------- |
| **스택**      | addLast()     | removeLast()      | **LIFO**      |
| **큐**        | addLast()     | removeFirst()     | **FIFO**      |

> 💡 Python은 **collections.deque**(이중 연결 리스트 기반), Java는 **ArrayDeque**(원형 배열 기반)를 표준 덱 구현체로 제공한다. 특히 Java 공식 문서는 레거시 **Stack** 클래스 대신 **ArrayDeque**를 스택 용도로 사용할 것을 권장한다.

<br>

### 4. 덱의 활용 — 슬라이딩 윈도우 최댓값

크기 k의 윈도우를 한 칸씩 이동시키며 **각 구간의 최댓값**을 구하는 문제는, 덱에 **인덱스를 내림차순 값 순서로 유지**하는 방식으로 **O(n)** 에 해결할 수 있다. 양 끝에서 삽입/삭제가 모두 필요하므로 덱이 아니면 구현할 수 없는 대표적인 사례다.

```python
from collections import deque

def sliding_window_max(nums, k):
    dq = deque()  # 값이 큰 순서를 유지하는 인덱스 저장
    result = []
    for i, num in enumerate(nums):
        if dq and dq[0] <= i - k:      # 윈도우를 벗어난 인덱스 제거 (Front)
            dq.popleft()
        while dq and nums[dq[-1]] < num:  # 현재 값보다 작은 값 제거 (Rear)
            dq.pop()
        dq.append(i)
        if i >= k - 1:
            result.append(nums[dq[0]])  # Front가 항상 윈도우 최댓값
    return result
```

이 외에도 **회문(팰린드롬) 검사**, **BFS의 0-1 가중치 변형(0-1 BFS)** 등에서 덱이 활용된다.

<br>

---

### 5. 우선순위 큐의 개념

**우선순위 큐**는 각 데이터가 **우선순위**를 가지며, 삭제 시 **들어온 순서가 아니라 우선순위가 가장 높은 데이터**가 먼저 나가는 자료구조다.

| **구분**        | **FIFO 큐**            | **우선순위 큐**                |
| --------------- | ---------------------- | ------------------------------ |
| **삭제 기준**   | 먼저 들어온 데이터     | **우선순위가 가장 높은** 데이터 |
| **내부 순서**   | 삽입 순서 유지         | 우선순위 기준 **반정렬** 상태  |
| **대표 구현**   | 배열, 연결리스트       | **힙(Heap)**                   |
| **활용 예시**   | 프로세스 대기열, BFS   | 다익스트라, 작업 스케줄링      |

<br>

### 6. 우선순위 큐의 구현 방식 비교

| **구현 방식**         | **삽입**     | **삭제(최우선 원소)** | **비고**            |
| --------------------- | ------------ | --------------------- | ------------------- |
| **정렬되지 않은 배열/연결리스트** | O(1)         | O(n)                  | 삭제 시 전체 탐색   |
| **정렬된 배열/연결리스트**       | O(n)         | O(1)                  | 삽입 시 위치 탐색   |
| **힙(Heap)**          | **O(log n)** | **O(log n)**          | **표준 구현 방식**  |

힙은 삽입과 삭제가 모두 **O(log n)** 으로 균형 잡혀 있어 우선순위 큐의 **표준 구현 방식**으로 사용된다. 힙의 내부 동작(삽입/삭제 과정, 배열 표현)은 **unit06(힙)** 을 참고한다.

<br>

### 7. Python heapq 사용 예시

Python의 **heapq** 모듈은 리스트를 **최소 힙**으로 다루는 함수를 제공한다.

```python
import heapq

pq = []
heapq.heappush(pq, 3)
heapq.heappush(pq, 1)
heapq.heappush(pq, 2)

print(heapq.heappop(pq))  # 1 (가장 작은 값부터 삭제)
print(pq[0])              # 2 (삭제 없이 최솟값 조회)
```

> ⚠️ **heapq**는 **최소 힙만 지원**한다. 최대 힙이 필요하면 값에 **-1을 곱해 삽입**하고 꺼낼 때 다시 -1을 곱하거나, `(우선순위, 값)` 튜플의 우선순위 부호를 뒤집어 사용해야 한다.

<br>

### 8. Java PriorityQueue 사용 예시

Java의 **PriorityQueue**는 힙 기반 우선순위 큐 구현체로, 기본적으로 **최소 힙**으로 동작한다.

```java
import java.util.PriorityQueue;
import java.util.Comparator;

// 최소 힙 (기본)
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
minHeap.offer(3);
minHeap.offer(1);
minHeap.offer(2);
System.out.println(minHeap.poll()); // 1

// 최대 힙 (Comparator 지정)
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
maxHeap.offer(3);
maxHeap.offer(1);
System.out.println(maxHeap.poll()); // 3
```

> 💡 우선순위 큐는 **다익스트라 최단 경로**(algorithm unit13), **힙 정렬**, 운영체제의 **작업 스케줄링** 등 "가장 우선순위 높은 것부터 처리"가 필요한 모든 곳에서 핵심 자료구조로 사용된다.
