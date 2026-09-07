## 스트림과 함수형 인터페이스

Java 8에서 도입된 **람다(Lambda)와 함수형 인터페이스**는 동작(코드)을 값처럼 전달할 수 있게 했고, 그 위에 세워진 **스트림(Stream) API**는 컬렉션 처리를 "어떻게(how)"가 아니라 "무엇을(what)" 중심의 선언형 파이프라인으로 바꾸었다. 스트림이 **지연 평가**되는 원리와 병렬 스트림이 언제 오히려 느려지는지를 모르면 읽기 쉬운 코드가 성능 문제의 원인이 된다.

<br>

### 1. 함수형 인터페이스와 람다

**함수형 인터페이스(Functional Interface)**는 **추상 메서드가 정확히 하나**인 인터페이스다. 람다식은 이 인터페이스의 인스턴스로 컴파일되며, `@FunctionalInterface`를 붙이면 추상 메서드가 둘 이상일 때 컴파일 오류로 잡아 준다.

| **인터페이스**            | **시그니처**          | **의미**                          | **대표 사용처**                        |
| ------------------------- | --------------------- | --------------------------------- | -------------------------------------- |
| **`Supplier<T>`**          | `() → T`              | 값을 **생산**                       | 지연 생성, `Optional.orElseGet`         |
| **`Consumer<T>`**          | `T → void`            | 값을 **소비**                       | `forEach`                              |
| **`Function<T, R>`**       | `T → R`               | 값을 **변환**                       | `map`                                  |
| **`Predicate<T>`**         | `T → boolean`         | 조건 **판정**                       | `filter`, `removeIf`                   |
| **`UnaryOperator<T>`**     | `T → T`               | 같은 타입으로 변환                   | `replaceAll`                           |
| **`BinaryOperator<T>`**    | `(T, T) → T`          | 두 값을 하나로                      | `reduce`                               |
| **`BiFunction<T, U, R>`**  | `(T, U) → R`          | 두 인자 변환                        | `Map.merge`, `compute`                 |

```java
// 익명 클래스 → 람다 → 메서드 참조
Comparator<String> byLength = new Comparator<>() {
    @Override public int compare(String a, String b) { return Integer.compare(a.length(), b.length()); }
};
Comparator<String> byLength2 = (a, b) -> Integer.compare(a.length(), b.length());
Comparator<String> byLength3 = Comparator.comparingInt(String::length);
```

- 람다가 캡처하는 지역 변수는 **effectively final**(사실상 final)이어야 함. 람다는 값을 복사해 가지며, 스택 프레임이 사라진 뒤에도 실행될 수 있기 때문
- `IntFunction`, `ToIntFunction`, `IntPredicate` 등 **기본형 특화 인터페이스**는 오토박싱 비용을 피하기 위해 존재함
- 람다는 익명 클래스와 달리 `this`가 **바깥 인스턴스**를 가리키며, 클래스 파일을 생성하지 않고 `invokedynamic`으로 런타임에 구현체를 만듦

> 💡 함수형 인터페이스는 Checked 예외를 선언하지 않으므로 람다 본문에서 `IOException` 같은 예외를 던지면 컴파일되지 않는다. 람다 안에서 `try-catch`로 감싸 Unchecked 예외로 번역하거나, 예외를 선언한 자체 함수형 인터페이스를 정의해야 한다. Checked/Unchecked 설계 기준은 **unit05** 참고.

<br>

### 2. 스트림 파이프라인 구조

스트림은 **소스 → 중간 연산(0개 이상) → 최종 연산(1개)**으로 구성된다. 중간 연산은 새 스트림을 반환해 체이닝되고, 최종 연산이 호출되어야 비로소 전체 파이프라인이 실행된다.

```
 소스                중간 연산 (lazy, 스트림 반환)                최종 연산 (eager, 결과 반환)
 list.stream() ──▶ filter(p) ──▶ map(f) ──▶ sorted() ──▶ limit(3) ──▶ collect(toList())
                   ─────────── 실행 계획만 쌓임 ───────────            ▲ 여기서 실제 실행
```

| **분류**                | **연산 예시**                                                    | **특징**                                     |
| ----------------------- | ---------------------------------------------------------------- | -------------------------------------------- |
| **중간 연산 (무상태)**    | `filter`, `map`, `flatMap`, `peek`, `mapToInt`                    | 원소 하나씩 독립적으로 처리 가능               |
| **중간 연산 (상태 있음)** | `sorted`, `distinct`, `limit`, `skip`                             | 이전 원소 정보가 필요 → 전체 또는 일부를 버퍼링 |
| **최종 연산**            | `collect`, `forEach`, `reduce`, `count`, `findFirst`, `anyMatch`  | 파이프라인 실행을 촉발, 스트림 소비             |
| **단락(short-circuit)**  | `findFirst`, `anyMatch`, `limit`, `allMatch`                      | 결과가 확정되면 나머지 원소를 처리하지 않음     |

```java
record Order(String customer, int amount, boolean paid) {}

Map<String, Integer> totalByCustomer = orders.stream()
        .filter(Order::paid)
        .collect(Collectors.groupingBy(
                Order::customer,
                Collectors.summingInt(Order::amount)));

List<String> top3 = orders.stream()
        .sorted(Comparator.comparingInt(Order::amount).reversed())
        .map(Order::customer)
        .distinct()
        .limit(3)
        .toList();                              // Java 16+: 불변 리스트 반환
```

❗️**스트림은 일회용**: 최종 연산이 끝난 스트림을 다시 사용하면 `IllegalStateException: stream has already been operated upon or closed`가 발생한다. 여러 번 순회하려면 소스 컬렉션에서 스트림을 다시 만들어야 한다.

<br>

### 3. 지연 평가(Lazy Evaluation)

### 3-1. 최종 연산이 있어야 실행된다

```java
Stream<Integer> s = Stream.of(1, 2, 3)
        .peek(n -> System.out.println("peek " + n))
        .map(n -> n * 2);
System.out.println("파이프라인 구성 완료");     // 이 시점까지 peek는 한 번도 출력되지 않음

List<Integer> result = s.toList();             // 여기서 비로소 peek 1, peek 2, peek 3 출력
```

- 중간 연산은 "무엇을 할지"만 기록하고, 최종 연산이 호출되면 파이프라인을 **한 번에 융합(fusion)**해 실행함
- 덕분에 `Stream.iterate(1, n -> n + 1)`처럼 **무한 스트림**도 `limit`·`findFirst`와 결합하면 정상 동작함

<br>

### 3-2. 원소 단위 수직 처리와 단락

스트림은 "1단계를 전부 끝내고 2단계로" 가는 것이 아니라, **원소 하나가 파이프라인을 끝까지 통과한 뒤 다음 원소**를 처리한다. 이 순서 덕분에 단락 연산이 불필요한 계산을 건너뛴다.

```java
Stream.of("a", "bb", "ccc", "dddd")
        .filter(w -> { System.out.println("filter " + w); return w.length() >= 2; })
        .map(w -> { System.out.println("map " + w);    return w.toUpperCase(); })
        .findFirst();
// 출력: filter a → filter bb → map bb   ("ccc", "dddd"는 아예 처리되지 않음)
```

```
 원소 a  : filter ✗ ──────────────────────▶ 버림
 원소 bb : filter ✓ ──▶ map ──▶ findFirst ✓ ──▶ 종료 (단락)
 원소 ccc: (처리 안 함)
```

> ⚠️ `sorted()`·`distinct()` 같은 **상태 있는 연산은 배리어(barrier)**가 된다. `sorted()`는 모든 원소를 받아야 정렬할 수 있으므로, `sorted().findFirst()`는 전체를 정렬한 뒤에야 첫 원소를 낸다. 무한 스트림에 `sorted()`를 걸면 끝나지 않으며, 지연 평가의 이점을 살리려면 `filter`·`limit`을 **가능한 앞쪽**에 두어야 한다.

<br>

### 4. 흔한 함정과 올바른 사용

```java
// 안티패턴 1: 람다 안에서 외부 상태 변경 (부수 효과) — 병렬화 시 즉시 깨짐
List<String> result = new ArrayList<>();
names.stream().filter(n -> n.startsWith("k")).forEach(result::add);

// 개선: 결과는 collect로 만든다
List<String> result = names.stream().filter(n -> n.startsWith("k")).toList();
```

```java
// 안티패턴 2: toMap의 중복 키 → IllegalStateException: Duplicate key
Map<String, Order> byCustomer = orders.stream()
        .collect(Collectors.toMap(Order::customer, o -> o));

// 개선: 병합 함수 지정 (또는 groupingBy 사용)
Map<String, Order> latest = orders.stream()
        .collect(Collectors.toMap(Order::customer, o -> o,
                (a, b) -> a.amount() >= b.amount() ? a : b));
```

- `Optional`은 반환 타입 용도로만 쓰고, 필드·매개변수·컬렉션 원소로 쓰지 않음. `orElse(expensive())`는 값이 있어도 인자를 평가하므로 비싼 연산은 `orElseGet(() -> expensive())`로 지연시킴
- 기본형 처리는 `mapToInt`·`IntStream`으로 박싱을 피함. `Stream<Integer>`의 `reduce(0, Integer::sum)`은 원소마다 언박싱·박싱이 일어남
- 단순 반복은 for 문이 더 명확하고 빠른 경우가 많으며, 스트림은 **변환·필터·집계가 연쇄되는 로직**에 가치가 있음. 디버깅이 어렵고 스택 트레이스가 길어지는 점도 고려함

<br>

### 5. 병렬 스트림의 함정

`parallel()` 한 줄로 병렬화되지만, 내부 동작을 모르면 **느려지거나 다른 작업까지 마비**시킬 수 있다.

```
 list.parallelStream()
   └─ Spliterator로 데이터를 분할 ─▶ ForkJoinPool.commonPool()의 워커 스레드에 분배
        ┌──── 청크 1 ──── 워커 1 ────┐
        ├──── 청크 2 ──── 워커 2 ────┼──▶ 부분 결과 결합(combine) ──▶ 최종 결과
        └──── 청크 3 ──── 호출 스레드 ─┘
   commonPool 크기 = CPU 코어 수 - 1 (기본), JVM 전체에서 하나를 공유
```

| **함정**                          | **설명**                                                                                    | **대응**                                                    |
| --------------------------------- | ------------------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| **공용 풀 공유**                   | 모든 병렬 스트림·`CompletableFuture` 기본 실행이 **하나의 commonPool**을 씀. 한 작업이 점유하면 다른 곳이 지연 | 격리가 필요하면 별도 `ForkJoinPool`에서 실행(비공식 관행)      |
| **블로킹 I/O**                     | 워커가 DB·HTTP 대기에 묶이면 코어 수만큼밖에 없는 풀이 **고갈**                                 | 병렬 스트림은 **CPU 연산 전용**. I/O는 `ExecutorService`(unit09) |
| **분할이 어려운 소스**              | `LinkedList`·`Stream.iterate`·`BufferedReader.lines()`는 크기를 모르거나 순차 접근이라 분할 비용이 큼 | `ArrayList`·배열·`IntStream.range`처럼 분할이 쉬운 소스 사용     |
| **작은 데이터·가벼운 연산**          | 스레드 분배·결합 오버헤드가 이득보다 큼                                                        | 원소 수 × 원소당 비용이 충분히 클 때만(대략 수만 건 이상) 고려    |
| **순서 의존 연산**                  | `limit`·`findFirst`·`forEachOrdered`는 순서 유지 비용 발생                                     | 순서가 무의미하면 `unordered()`·`findAny`·`forEach`            |
| **스레드 안전하지 않은 부수 효과**    | `forEach`로 `ArrayList`에 추가 → 유실·예외                                                    | `collect` 사용. 컬렉터는 병렬 결합을 안전하게 처리               |
| **결합 법칙 위반 `reduce`**          | `(a, b) -> a - b`처럼 순서에 따라 결과가 달라지는 누적은 병렬 시 **틀린 답**                     | 항등원·결합 법칙을 만족하는 연산만 사용                          |

```java
// 안티패턴: 병렬 스트림 안에서 블로킹 호출 — commonPool 고갈로 애플리케이션 전체가 느려짐
List<Profile> profiles = userIds.parallelStream()
        .map(id -> httpClient.fetchProfile(id))     // 네트워크 대기
        .toList();

// 개선: I/O는 전용 스레드 풀과 CompletableFuture로 (unit09)
ExecutorService io = Executors.newFixedThreadPool(32);
List<CompletableFuture<Profile>> futures = userIds.stream()
        .map(id -> CompletableFuture.supplyAsync(() -> httpClient.fetchProfile(id), io))
        .toList();
List<Profile> profiles = futures.stream().map(CompletableFuture::join).toList();
```

> 💡 병렬 스트림의 사용 조건을 한 줄로 요약하면 "**분할이 쉬운 소스 + CPU 바운드 연산 + 충분한 데이터 + 부수 효과 없음**"이다. 넷 중 하나라도 빠지면 순차 스트림이 낫다. 면접에서는 "왜 기본으로 병렬을 쓰면 안 되는가"를 commonPool 공유와 블로킹 I/O로 설명할 수 있어야 한다.

<br>

### 6. 정리

| **항목**                 | **핵심**                                                                                 |
| ------------------------ | ---------------------------------------------------------------------------------------- |
| **함수형 인터페이스**      | 추상 메서드 하나. 람다는 그 구현체이며 캡처 변수는 effectively final                          |
| **파이프라인**            | 소스 → 중간 연산(지연) → 최종 연산(실행). 스트림은 **일회용**                                  |
| **지연 평가**             | 최종 연산 전까지 실행 안 됨. 원소 단위 수직 처리로 **단락 연산이 불필요한 계산을 생략**            |
| **상태 있는 연산**         | `sorted`·`distinct`는 배리어. `filter`·`limit`을 앞에 두어 처리량을 줄임                      |
| **부수 효과 금지**         | 결과는 `collect`로 생성. 외부 컬렉션 변경은 병렬화 시 깨짐                                     |
| **병렬 스트림**            | commonPool 공유·블로킹 I/O·분할 비용·순서 비용·결합 법칙을 확인한 뒤에만. I/O는 전용 풀로         |

- 병렬 실행에 쓰이는 `ForkJoinPool`·`CompletableFuture`와 스레드 풀 설계는 **unit09**, 람다가 캡처한 값을 여러 스레드가 볼 때의 가시성 문제는 **unit08** 참고
