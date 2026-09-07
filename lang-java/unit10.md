## 불변 객체와 값 설계

**불변 객체(Immutable Object)**는 생성 이후 상태가 절대 바뀌지 않는 객체다. 상태가 바뀌지 않으면 동기화 없이 스레드 간에 공유할 수 있고, 해시 컬렉션의 키로 안전하며, 어디서 값이 바뀌었는지 추적할 필요가 없어진다. `final`·`record`·**방어적 복사**는 Java에서 불변성을 구현하는 세 축이며, "값(value)"을 표현하는 클래스를 어떻게 설계해야 하는지가 이 유닛의 주제다.

<br>

### 1. 불변 객체가 중요한 이유

| **이점**                    | **설명**                                                                          | **관련 유닛** |
| --------------------------- | --------------------------------------------------------------------------------- | ------------- |
| **스레드 안전**              | 상태 변경이 없으므로 경쟁 상태·가시성 문제가 원천적으로 없음. 락이 필요 없음             | unit08, 09    |
| **안전한 공유·캐싱**          | 같은 인스턴스를 마음껏 공유·재사용 가능 (`Integer` 캐시, `String` 상수 풀)               | unit01        |
| **해시 키로 안전**            | `hashCode`가 변하지 않으므로 `HashMap`·`HashSet`에서 잃어버릴 일이 없음                 | unit03        |
| **실패 원자성**              | 메서드 도중 예외가 나도 객체가 반쯤 바뀐 상태로 남지 않음                                | unit05        |
| **추론 용이**                | 값을 받은 시점의 상태가 계속 유지되므로 "누가 언제 바꿨나"를 추적할 필요가 없음            | -             |

```
 가변 객체 공유                              불변 객체 공유
 A ──▶ ┌────────┐ ◀── B                     A ──▶ ┌────────┐ ◀── B
       │ x = 10 │  B가 x=20으로 변경               │ x = 10 │  누구도 변경 불가
       └────────┘  → A는 영문도 모른 채 20을 봄      └────────┘  변경이 필요하면 새 객체 생성
```

- 단점은 값이 바뀔 때마다 **새 객체를 생성**해야 한다는 것. 문자열을 반복문에서 `+`로 이어 붙이면 매번 새 `String`이 생기므로 이런 경우에는 `StringBuilder` 같은 **가변 동반 클래스**를 씀

<br>

### 2. final의 정확한 의미

`final`은 "한 번만 대입할 수 있다"는 뜻이며, 붙는 위치에 따라 의미가 다르다.

| **대상**          | **의미**                                             | **불변성과의 관계**                                  |
| ----------------- | ---------------------------------------------------- | ---------------------------------------------------- |
| **변수·필드**      | 재대입 불가                                            | **참조가 고정될 뿐, 가리키는 객체는 바뀔 수 있음**       |
| **메서드**         | 하위 클래스에서 재정의 불가                             | 불변 클래스의 동작을 하위 클래스가 깨지 못하게 막음        |
| **클래스**         | 상속 불가                                              | 가변 하위 클래스가 불변 계약을 위반하는 것을 차단          |

```java
final List<String> tags = new ArrayList<>();
tags.add("java");                 // 가능 — 참조는 final이지만 리스트 내용은 가변
tags = new ArrayList<>();         // 컴파일 오류 — 재대입 불가
```

❗️**`final` ≠ 불변**: `final` 필드를 가진 클래스라도 그 필드가 가변 객체(`List`, `Date`, 배열)를 가리키면 클래스는 불변이 아니다. 불변성은 필드 하나의 속성이 아니라 **클래스 설계 전체**의 속성이다.

> 💡 `final` 필드는 JMM에서 특별 대우를 받는다. 생성자가 끝난 뒤 객체 참조가 다른 스레드에 전달되면, 그 스레드는 **`final` 필드의 초기화된 값을 반드시 본다**(안전한 발행). 일반 필드는 unit08의 DCL 예시처럼 초기화 전 값이 보일 수 있다. 불변 객체가 동기화 없이 공유 가능한 근거가 이 규칙이다.

<br>

### 3. 불변 클래스 작성 규칙

Effective Java가 정리한 다섯 가지 규칙이다.

- **상태 변경 메서드(setter)를 제공하지 않음**
- **클래스를 확장할 수 없게 함** — `final class` 또는 생성자를 `private`으로 하고 정적 팩토리 제공
- **모든 필드를 `private final`로 선언**
- **가변 컴포넌트를 외부에 노출하지 않음** — 생성자에서 받을 때와 접근자로 내줄 때 **방어적 복사**
- **변경이 필요하면 새 객체를 반환** — `withX()` 스타일 메서드

```java
public final class Money {
    private final long amount;
    private final Currency currency;

    private Money(long amount, Currency currency) {
        if (amount < 0) throw new IllegalArgumentException("금액은 음수일 수 없음: " + amount);
        this.amount = amount;
        this.currency = Objects.requireNonNull(currency);
    }

    public static Money of(long amount, Currency currency) { return new Money(amount, currency); }

    public Money plus(Money other) {                        // 자신을 바꾸지 않고 새 객체 반환
        if (!currency.equals(other.currency)) throw new IllegalArgumentException("통화 불일치");
        return new Money(amount + other.amount, currency);
    }

    public long amount() { return amount; }
    public Currency currency() { return currency; }

    @Override public boolean equals(Object o) {
        return o instanceof Money m && amount == m.amount && currency.equals(m.currency);
    }
    @Override public int hashCode() { return Objects.hash(amount, currency); }
}
```

- 값 객체는 **동치성**을 값 기준으로 정의해야 하므로 `equals`·`hashCode`를 반드시 함께 재정의함 (unit03)
- `BigDecimal`·`String`·`LocalDateTime`·래퍼 타입이 JDK의 대표적 불변 클래스임. `java.util.Date`·`Calendar`는 가변이라 새 코드에서는 `java.time`을 씀

<br>

### 4. 방어적 복사(Defensive Copy)

불변 클래스가 **가변 객체를 필드로 가질 때**, 외부의 참조를 그대로 저장하거나 그대로 내주면 외부에서 내부 상태를 바꿀 수 있다. 이를 막기 위해 **들어올 때와 나갈 때 모두 복사**한다.

```java
// 안티패턴: 참조를 그대로 저장·반환 → 외부에서 내부 리스트를 수정 가능
public final class Order {
    private final List<Item> items;
    public Order(List<Item> items) { this.items = items; }
    public List<Item> items() { return items; }
}

List<Item> src = new ArrayList<>(List.of(item1));
Order order = new Order(src);
src.add(item2);                    // 생성자에 넘긴 리스트를 바꾸면 Order 내부도 바뀜
order.items().clear();             // 접근자로 얻은 리스트를 비우면 Order가 비워짐
```

```java
// 개선: 들어올 때 복사(List.copyOf: Java 10+, 불변 복사본), 나갈 때는 이미 불변이므로 그대로 반환
public final class Order {
    private final List<Item> items;

    public Order(List<Item> items) {
        this.items = List.copyOf(items);           // null 원소 불가, 원본과 분리된 불변 리스트
    }
    public List<Item> items() { return items; }    // 수정 시도 → UnsupportedOperationException
}
```

| **방법**                                     | **원본과 분리** | **결과의 불변성**              | **비고**                                                   |
| -------------------------------------------- | --------------- | ----------------------------- | ---------------------------------------------------------- |
| **`new ArrayList<>(src)`**                    | 예              | 가변                           | 내부 작업용 복사. 접근자로 그대로 내주면 다시 노출됨            |
| **`Collections.unmodifiableList(src)`**       | **아니오**(뷰)   | 뷰만 불변, **원본 바뀌면 함께 변함** | 내부 리스트를 읽기 전용으로 노출할 때. 외부 원본을 감싸면 위험      |
| **`List.copyOf(src)`** / `Set.copyOf` / `Map.copyOf` | 예        | **불변**                        | 가장 안전. 이미 불변 컬렉션이면 복사 없이 재사용                  |
| **`array.clone()`**                           | 예              | 가변 (배열은 불변 불가)          | 배열 필드는 들어올 때·나갈 때 모두 `clone()` 필요               |

> ⚠️ 방어적 복사는 **검사보다 먼저** 해야 한다. 매개변수를 검사한 뒤 복사하면, 검사와 복사 사이에 다른 스레드가 원본을 바꾸는 **TOCTOU(Time-Of-Check to Time-Of-Use)** 공격·버그가 가능하다. "복사 → 복사본 검사" 순서가 원칙이다. 또한 복사는 **얕은 복사**이므로 원소 자체가 가변이면 원소도 불변 타입이어야 진짜 불변이 된다.

<br>

### 5. record — 값 클래스의 표준

**record(Java 16 정식)**는 "데이터를 담는 불변 캐리어"를 선언하는 문법으로, 컴포넌트 필드(`private final`)·생성자·접근자·`equals`·`hashCode`·`toString`을 컴파일러가 생성한다. 3절의 규칙 대부분이 자동으로 지켜진다.

```java
public record Money(long amount, Currency currency) {

    public Money {                                          // 컴팩트 생성자: 검증·정규화
        if (amount < 0) throw new IllegalArgumentException("금액은 음수일 수 없음: " + amount);
        Objects.requireNonNull(currency);
    }

    public static Money zero(Currency c) { return new Money(0, c); }

    public Money plus(Money other) {
        if (!currency.equals(other.currency)) throw new IllegalArgumentException("통화 불일치");
        return new Money(amount + other.amount, currency);
    }
}
```

**record의 특징과 제약**

| **항목**                | **내용**                                                                          |
| ----------------------- | --------------------------------------------------------------------------------- |
| **자동 생성**            | 정식(canonical) 생성자, 컴포넌트명과 같은 접근자(`amount()`), `equals`/`hashCode`/`toString` |
| **불변성**              | 모든 컴포넌트는 `private final`. **인스턴스 필드 추가 불가** (static 필드는 가능)          |
| **상속**                | 암묵적으로 `final`, 다른 클래스 상속 불가. **인터페이스 구현은 가능**                        |
| **컴팩트 생성자**         | 매개변수 선언 없이 검증·정규화 로직만 작성. 필드 대입은 끝에서 자동 수행                      |
| **얕은 불변**            | 컴포넌트가 가변 객체면 record 자체는 그 객체의 변경을 막지 못함 → **방어적 복사 필요**           |

```java
// record도 가변 컴포넌트는 방어적 복사가 필요하다
public record Order(String id, List<Item> items) {
    public Order {
        items = List.copyOf(items);       // 컴팩트 생성자에서 매개변수를 복사본으로 교체
    }
    // items() 접근자는 불변 리스트를 반환하므로 재정의 불필요
}
```

- record는 **값의 의미론(value semantics)**을 강제하므로 JPA 엔티티처럼 식별자로 동일성을 판단하고 상태가 변하는 객체에는 부적합함. DTO·값 객체·좌표·복합 키·API 응답 모델에 적합
- **sealed 인터페이스(Java 17)**와 결합하면 "정해진 형태 중 하나"를 표현하는 대수적 데이터 타입을 만들 수 있고, `switch` 패턴 매칭(Java 21)으로 모든 경우를 컴파일러가 검사함

```java
public sealed interface PaymentResult permits Approved, Declined {}
public record Approved(String txId, Money amount) implements PaymentResult {}
public record Declined(String reason) implements PaymentResult {}

String describe(PaymentResult r) {
    return switch (r) {                                   // Java 21: 누락된 case가 있으면 컴파일 오류
        case Approved a -> "승인 " + a.txId();
        case Declined d -> "거절: " + d.reason();
    };
}
```

<br>

### 6. 값 설계의 판단 기준

| **상황**                                                  | **선택**                                | **이유**                                                |
| --------------------------------------------------------- | --------------------------------------- | ------------------------------------------------------- |
| **값 자체가 의미**(금액, 좌표, 기간, 이메일)                  | **record** 또는 불변 클래스               | 동치성이 값 기준, 공유·캐싱 안전                            |
| **식별자로 구분되고 생명주기 동안 상태가 변함**(회원, 주문)     | 가변 엔티티 클래스                        | 같은 식별자면 값이 달라도 같은 객체. record 부적합            |
| **대량 반복 갱신**(문자열 조립, 대형 배열 누적)                | 가변 동반 클래스(`StringBuilder`) 후 불변으로 변환 | 객체 생성 비용 회피                                  |
| **컬렉션을 외부에 반환**                                     | `List.copyOf` 또는 `Stream.toList()`       | 호출자가 내부 상태를 바꾸지 못하게                           |
| **여러 스레드가 공유하는 설정·스냅샷**                         | 불변 객체 + `volatile`/`AtomicReference` 교체 | 읽기 락 불필요, 교체만 원자적으로                          |

**흔한 실수**

- 접근자에서 내부 배열·컬렉션을 그대로 반환 → 캡슐화 붕괴
- "`setter`만 없으면 불변"이라고 생각 → 가변 필드 참조 노출을 놓침
- 불변 객체에 `withX()`를 만들지 않고 편의상 setter를 추가 → 불변 계약 파기
- record에 Lombok `@Data`처럼 가변 스타일을 섞으려는 시도 → record는 애초에 불변 캐리어임

> 💡 "불변 객체는 매번 새로 만들어서 느리지 않나요?"라는 질문에는 두 가지로 답한다. 첫째, 대부분의 값 객체는 작고 수명이 짧아 Young 영역에서 즉시 회수되므로 비용이 미미하다(unit02). 둘째, JIT의 탈출 분석이 할당 자체를 제거하기도 한다(unit01). 진짜 병목은 프로파일링으로 확인된 극소수 지점에서만 가변 동반 클래스로 해결한다.

<br>

### 7. 정리

| **항목**                | **핵심**                                                                                            |
| ----------------------- | --------------------------------------------------------------------------------------------------- |
| **불변의 가치**          | 동기화 없는 공유, 안전한 해시 키, 실패 원자성, 추론 용이                                                   |
| **`final`의 의미**       | 재대입 금지일 뿐 객체 내용의 불변을 뜻하지 않음. 클래스 `final`은 하위 클래스의 계약 위반 차단                 |
| **불변 클래스 규칙**      | setter 없음, 확장 불가, 모든 필드 `private final`, 가변 컴포넌트 비노출, 변경은 새 객체 반환                  |
| **방어적 복사**          | 들어올 때·나갈 때 모두 복사. `List.copyOf`가 가장 안전. **복사 후 검사**(TOCTOU 방지)                        |
| **record**              | Java 16+ 불변 데이터 캐리어. 컴팩트 생성자로 검증, 가변 컴포넌트는 여전히 복사 필요. 엔티티에는 부적합          |
| **sealed + record**     | 정해진 형태의 합 타입을 표현하고 `switch` 패턴 매칭으로 완전성 검사                                           |

- `equals`/`hashCode` 규약은 **unit03**, 불변 객체가 동시성 문제를 없애는 원리는 **unit08**·**unit09**와 함께 볼 것
