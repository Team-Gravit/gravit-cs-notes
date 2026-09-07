## 컬렉션 내부 구조

Java 컬렉션 프레임워크는 `List`·`Set`·`Map` 인터페이스와 그 구현체들로 이루어져 있으며, 같은 인터페이스라도 **구현체마다 내부 자료구조와 시간 복잡도가 완전히 다르다**. 특히 가장 많이 쓰는 `HashMap`의 해시 충돌 처리와 리사이징, `ArrayList`와 `LinkedList`의 선택 기준은 코딩 테스트와 기술 면접 모두에서 반복해서 묻는 주제다.

<br>

### 1. 컬렉션 프레임워크 계층

```
 Iterable
   └── Collection
         ├── List  ─── ArrayList, LinkedList, (Vector, Stack: 레거시)
         ├── Set   ─── HashSet, LinkedHashSet, TreeSet
         └── Queue ─── PriorityQueue, ArrayDeque(Deque), LinkedList
 Map (Collection과 별도 계층)
   └── HashMap, LinkedHashMap, TreeMap, (Hashtable: 레거시), ConcurrentHashMap
```

- `Map`은 `Collection`을 상속하지 않지만 컬렉션 프레임워크의 일부로 취급함
- `HashSet`은 내부적으로 **`HashMap`을 감싸서** 키만 사용하며, `TreeSet`도 `TreeMap`을 감쌈. 따라서 `HashMap`을 이해하면 `HashSet`도 함께 이해됨
- `Vector`·`Hashtable`은 모든 메서드에 `synchronized`가 걸린 레거시 구현체로, 동시성이 필요하면 **unit09**의 동시성 컬렉션을 사용함

<br>

### 2. HashMap 내부 구조

### 2-1. 버킷 배열과 인덱스 계산

`HashMap`은 `Node<K,V>[] table`이라는 **버킷 배열**을 가지며, 키의 해시 값으로 배열 인덱스를 계산해 저장한다. 기본 초기 용량은 **16**, 용량은 항상 **2의 거듭제곱**으로 유지된다.

```java
// HotSpot JDK의 HashMap.hash() 개념 코드
static int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);   // 상위 16비트를 하위에 섞음
}
// 인덱스 = (table.length - 1) & hash   → 용량이 2^n이므로 나머지 연산 대신 비트 AND로 계산
```

- 배열 길이가 2의 거듭제곱이면 `(n - 1) & hash`가 `hash % n`과 같은 결과를 훨씬 싸게 냄
- 하지만 이 방식은 **해시의 하위 비트만** 인덱스에 쓰므로, 상위 비트를 XOR로 섞어 충돌을 줄임 (사용자 `hashCode`가 나쁠 때의 보완 장치)
- `null` 키는 해시 0으로 항상 0번 버킷에 저장되어 하나만 허용됨

<br>

### 2-2. 해시 충돌 처리 — 체이닝과 트리화

서로 다른 키가 같은 버킷 인덱스를 얻는 것이 **해시 충돌**이다. `HashMap`은 **분리 연결법(Separate Chaining)**으로 같은 버킷의 엔트리를 연결 리스트로 잇고, Java 8부터는 리스트가 길어지면 **레드-블랙 트리**로 바꾼다.

```
 table[]
 ┌───┐
 │ 0 │ → (k1,v1)
 ├───┤
 │ 1 │ → null
 ├───┤
 │ 2 │ → (k5,v5) → (k9,v9) → (k13,v13)        ← 체이닝: 같은 버킷의 연결 리스트
 ├───┤
 │ 3 │ → [레드-블랙 트리]                      ← 노드 8개 초과 && 용량 64 이상이면 트리화
 └───┘
```

| **조건**                       | **값(기본)** | **의미**                                                         |
| ------------------------------ | ------------ | ---------------------------------------------------------------- |
| **TREEIFY_THRESHOLD**          | **8**        | 한 버킷의 노드가 8개를 넘으면 트리로 변환 (단, 용량 64 이상일 때)     |
| **MIN_TREEIFY_CAPACITY**       | **64**       | 용량이 64 미만이면 트리화 대신 **리사이징**을 먼저 수행               |
| **UNTREEIFY_THRESHOLD**        | **6**        | 리사이징으로 노드가 6개 이하로 줄면 다시 연결 리스트로 되돌림          |

- 트리화 덕분에 최악의 경우 조회가 `O(n)`에서 **`O(log n)`**으로 개선됨. 악의적으로 해시가 같은 키를 대량 삽입하는 **해시 플러딩(Hash DoS)** 공격 대응 목적이기도 함
- 트리 안에서 키를 정렬하려면 비교가 필요하므로 `Comparable`을 구현한 키가 유리하며, 아니면 클래스 이름·`identityHashCode`로 순서를 정함

> 💡 좋은 `hashCode`를 쓰면 버킷당 노드 수가 포아송 분포를 따라 8개를 넘을 확률이 천만분의 1 이하다. 즉, 트리화가 자주 일어난다면 **키의 `hashCode` 구현이 잘못된 것**이다. `equals`/`hashCode` 규약은 **unit03** 참고.

<br>

### 2-3. 리사이징(Resize)

엔트리 수가 **용량 × 로드 팩터(기본 0.75)**를 넘으면 배열을 **2배**로 늘리고 모든 엔트리를 재배치한다. 예를 들어 용량 16이면 13번째 엔트리를 넣을 때 32로 확장된다.

```
 리사이징 전 (n=16)             리사이징 후 (n=32)
 hash & 15 = 5 인 노드들   →    hash & 16 == 0  ? 그대로 5번 버킷  (lo 리스트)
                                hash & 16 != 0  ? 5+16=21번 버킷 (hi 리스트)
 ▶ 각 노드를 재해싱하지 않고 "새로 추가된 비트 하나"만 검사해 두 그룹으로 나눔
```

- 리사이징은 모든 노드를 순회하므로 **`O(n)`**이며, 이때 큰 맵이면 순간적인 지연이 발생함
- 저장할 개수를 미리 알면 `new HashMap<>(expectedSize / 0.75 + 1)`처럼 **초기 용량을 지정**해 리사이징을 피함
- 로드 팩터를 높이면 메모리는 아끼지만 충돌이 늘고, 낮추면 반대. 0.75는 시간·공간의 절충값임

```java
// 안티패턴: 10만 건을 넣을 것을 알면서 기본 용량 사용 → 리사이징 약 13회 발생
Map<String, User> users = new HashMap<>();

// 개선: 예상 크기를 고려해 초기 용량 지정 (Java 19+에는 HashMap.newHashMap(expected)도 있음)
Map<String, User> users = new HashMap<>((int) (100_000 / 0.75f) + 1);
```

> ⚠️ `HashMap`은 스레드 안전하지 않다. Java 7까지는 여러 스레드가 동시에 리사이징하면 연결 리스트에 순환이 생겨 `get()`이 **무한 루프**에 빠지는 유명한 버그가 있었다. Java 8에서 삽입 방식을 바꿔 순환은 사라졌지만, 여전히 데이터 유실이 발생하므로 동시 접근에는 `ConcurrentHashMap`(unit09)을 써야 한다.

<br>

### 3. 순서·정렬이 필요한 Map/Set

| **구현체**              | **내부 구조**                          | **순서**                     | **조회·삽입**   | **적합한 상황**                          |
| ----------------------- | -------------------------------------- | ---------------------------- | --------------- | ---------------------------------------- |
| **HashMap / HashSet**   | 해시 버킷 배열 + 체이닝·트리            | **보장 안 됨**                | 평균 `O(1)`     | 순서가 무의미한 일반적인 키-값 저장          |
| **LinkedHashMap / LinkedHashSet** | 해시 테이블 + 이중 연결 리스트  | **삽입 순서**(또는 접근 순서) | 평균 `O(1)`     | 순서 유지 캐시, LRU 캐시(`accessOrder=true`) |
| **TreeMap / TreeSet**   | **레드-블랙 트리**                      | **키 정렬 순서**              | `O(log n)`      | 범위 검색(`subMap`, `ceilingKey`), 정렬 순회 |

- `TreeMap`은 키가 `Comparable`이거나 생성자에 `Comparator`를 넘겨야 하며, `null` 키를 허용하지 않음
- `HashMap`의 순회 순서는 리사이징 후 바뀔 수 있으므로 **순서에 의존하는 코드는 버그**임

<br>

### 4. List 구현체 선택 — ArrayList vs LinkedList

| **연산**                          | **ArrayList (동적 배열)**                | **LinkedList (이중 연결 리스트)**             |
| --------------------------------- | ---------------------------------------- | --------------------------------------------- |
| **인덱스 접근 `get(i)`**            | **`O(1)`**                               | `O(n)` — 앞·뒤 중 가까운 쪽에서 순회            |
| **끝에 추가 `add(e)`**              | 분할 상환 `O(1)` (용량 초과 시 복사)        | `O(1)`                                        |
| **중간 삽입·삭제 `add(i, e)`**       | `O(n)` — 뒤 원소 전부 이동                 | `O(n)` — 위치를 찾는 순회 비용 (노드 연결 자체는 `O(1)`) |
| **이터레이터로 순회 중 삭제**         | `O(n)`                                   | **`O(1)`**                                    |
| **메모리**                         | 원소당 참조 1개 + 여유 공간                | 원소당 노드 객체(참조 3개) → **오버헤드 큼**      |
| **캐시 지역성**                     | **연속 메모리라 매우 좋음**                | 노드가 힙에 흩어져 나쁨                          |

```java
// ArrayList의 용량 증가: 처음 add 시 10, 이후 1.5배씩 확장 (Arrays.copyOf로 복사)
List<Integer> list = new ArrayList<>();       // 내부 배열은 비어 있음 (지연 할당)
for (int i = 0; i < 100; i++) list.add(i);    // 10 → 15 → 22 → 33 → 49 → 73 → 109 순으로 확장

// 크기를 알면 미리 확보
List<Integer> sized = new ArrayList<>(100);
```

- 실무에서는 **거의 항상 `ArrayList`**를 쓴다. `LinkedList`가 이론상 유리한 "중간 삽입·삭제"도 위치를 찾는 순회 비용 때문에 실제로는 `ArrayList`가 빠른 경우가 많음
- 양 끝 삽입·삭제가 핵심인 큐·스택 용도라면 `LinkedList`보다 **`ArrayDeque`**가 빠르고 메모리 효율도 좋음 (`Stack` 클래스는 레거시라 사용하지 않음)

> 💡 "`LinkedList`를 언제 쓰나요?"에 대한 현실적인 답은 "`ListIterator`로 순회하면서 그 자리에서 삽입·삭제를 반복해야 할 때 정도이며, 그마저도 `ArrayDeque`나 `ArrayList`로 대체되는 경우가 대부분"이다. 자료구조의 이론적 복잡도와 캐시 지역성이 포함된 실제 성능을 구분해 답하면 좋다.

<br>

### 4-1. 불변 리스트와 고정 크기 리스트

```java
List<String> a = Arrays.asList("x", "y");   // 배열을 감싼 고정 크기 리스트: set() 가능, add()/remove() 불가
List<String> b = List.of("x", "y");         // Java 9+ 완전 불변: 모든 변경 연산 불가, null 원소 불가
List<String> c = Collections.unmodifiableList(new ArrayList<>(a)); // 뷰(view): 원본이 바뀌면 함께 바뀜

a.set(0, "z");    // 가능 — 원본 배열도 함께 바뀜
b.add("w");       // UnsupportedOperationException
```

- `List.of`·`Map.of`·`Set.of`는 **불변 컬렉션**으로 반환 값을 방어적으로 노출할 때 유용함 (**unit10** 참고)
- `Stream.toList()`(Java 16+)도 불변 리스트를 반환하므로 결과를 수정하려면 `Collectors.toCollection(ArrayList::new)`를 사용함

<br>

### 5. 순회 중 수정과 fail-fast

`ArrayList`·`HashMap` 등의 이터레이터는 **fail-fast**로 동작한다. 컬렉션은 구조 변경 횟수(`modCount`)를 기록하고, 이터레이터가 다음 원소를 꺼낼 때 이 값이 바뀌어 있으면 `ConcurrentModificationException`을 던진다.

```java
// 안티패턴: for-each 순회 중 컬렉션 직접 수정
List<Integer> nums = new ArrayList<>(List.of(1, 2, 3, 4));
for (Integer n : nums) {
    if (n % 2 == 0) nums.remove(n);       // ConcurrentModificationException
}

// 개선 1: 이터레이터의 remove 사용
for (Iterator<Integer> it = nums.iterator(); it.hasNext(); ) {
    if (it.next() % 2 == 0) it.remove();
}
// 개선 2: removeIf (내부적으로 안전하게 처리)
nums.removeIf(n -> n % 2 == 0);
```

- fail-fast는 **버그를 빨리 드러내기 위한 장치**일 뿐 동시성 보장이 아님. 멀티스레드 환경에서는 예외가 나지 않고 조용히 깨질 수도 있음
- `CopyOnWriteArrayList`·`ConcurrentHashMap`의 이터레이터는 스냅샷 또는 약한 일관성(weakly consistent)으로 동작해 예외를 던지지 않음 (**unit09**)

<br>

### 6. 정리 — 구현체 선택 기준

| **요구 사항**                                     | **선택**                          | **이유**                                       |
| ------------------------------------------------- | --------------------------------- | ---------------------------------------------- |
| **키로 빠르게 찾기, 순서 무관**                     | `HashMap` / `HashSet`             | 평균 `O(1)`, 초기 용량 지정으로 리사이징 회피       |
| **삽입 순서 유지, LRU 캐시**                        | `LinkedHashMap`                   | 해시 + 연결 리스트, `removeEldestEntry`           |
| **정렬·범위 검색**                                  | `TreeMap` / `TreeSet`             | 레드-블랙 트리, `O(log n)`                        |
| **인덱스 접근·순회 위주 리스트**                     | `ArrayList`                       | 연속 메모리, 캐시 지역성                          |
| **양 끝 삽입·삭제(스택·큐)**                         | `ArrayDeque`                      | 순환 배열, `LinkedList`·`Stack`보다 빠름          |
| **우선순위 기반 꺼내기**                             | `PriorityQueue`                   | 이진 힙, 삽입·삭제 `O(log n)`                     |
| **멀티스레드 공유**                                  | `ConcurrentHashMap` 등 (unit09)   | `HashMap`은 동시 수정 시 데이터 유실               |

- `HashMap`은 **해시 → 버킷 인덱스 → 체이닝(8개 초과 시 트리) → 0.75 초과 시 2배 리사이징** 흐름으로 요약할 수 있어야 함
- `LinkedList`는 이론과 달리 실무에서 거의 선택되지 않으며, 그 이유(순회 비용·캐시 지역성·노드 오버헤드)를 설명할 수 있어야 함
