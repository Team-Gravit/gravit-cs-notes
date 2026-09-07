## equals와 hashCode 계약

`equals()`는 두 객체가 **논리적으로 같은지(동치성)**를, `hashCode()`는 객체를 **해시 기반 자료구조에서 빠르게 찾기 위한 정수**를 정의한다. 두 메서드는 `Object`에 정의된 **규약(contract)**으로 서로 묶여 있으며, 하나만 재정의하거나 규약을 어기면 `HashMap`·`HashSet`에서 "분명히 넣었는데 못 찾는" 버그가 생긴다.

<br>

### 1. 동일성(Identity)과 동치성(Equality)

| **비교 방식**      | **연산**       | **기준**                          | **예시**                                        |
| ------------------ | -------------- | --------------------------------- | ----------------------------------------------- |
| **동일성**         | `==`           | **같은 객체(같은 참조)**인가        | `a == b` → 두 변수가 같은 힙 주소를 가리킴          |
| **동치성**         | `equals()`     | **논리적으로 같은 값**인가          | `"abc".equals(new String("abc"))` → `true`      |

`Object`의 기본 `equals()`는 `this == obj`로 구현되어 있어 재정의하지 않으면 동일성 비교와 같다. 값을 표현하는 클래스(값 객체, DTO, 식별자 등)는 반드시 동치성을 재정의해야 한다.

```java
String a = new String("java");
String b = new String("java");

System.out.println(a == b);        // false — 서로 다른 객체
System.out.println(a.equals(b));   // true  — 내용이 같음
```

> 💡 `Integer` 비교에서 `==`가 -128~127 범위에서는 `true`, 그 밖에서는 `false`가 나오는 현상은 **`Integer` 캐시** 때문이다. 오토박싱된 래퍼 타입은 항상 `equals()`로 비교해야 하며, 이 문제를 아는지가 "동일성과 동치성을 구분하는가"를 묻는 단골 질문이다.

<br>

### 2. equals() 규약

`Object.equals()`의 Javadoc이 요구하는 다섯 가지 조건이다. `null`이 아닌 모든 참조 `x`, `y`, `z`에 대해 성립해야 한다.

| **규약**              | **조건**                                                                  | **위반 시 문제**                                  |
| --------------------- | ------------------------------------------------------------------------- | ------------------------------------------------- |
| **반사성(Reflexive)**  | `x.equals(x)`는 `true`                                                     | 컬렉션에 넣은 객체를 `contains`로 못 찾음            |
| **대칭성(Symmetric)**  | `x.equals(y)`가 `true`면 `y.equals(x)`도 `true`                            | 비교 순서에 따라 결과가 달라짐                       |
| **추이성(Transitive)** | `x.equals(y)`, `y.equals(z)`가 `true`면 `x.equals(z)`도 `true`             | 상속 계층에서 필드를 추가할 때 흔히 깨짐              |
| **일관성(Consistent)** | 비교에 쓰이는 정보가 바뀌지 않는 한 결과가 항상 같음                         | 네트워크·시간 등 외부 상태에 의존하면 위반            |
| **null 처리**          | `x.equals(null)`은 `false` (예외를 던지면 안 됨)                            | `NullPointerException` 발생                        |

**대칭성 위반 예시** — 다른 타입과의 호환을 시도하다가 흔히 깨진다.

```java
// 안티패턴: String과도 같다고 판단하려다 대칭성 위반
public final class CaseInsensitiveString {
    private final String s;
    public CaseInsensitiveString(String s) { this.s = s; }

    @Override
    public boolean equals(Object o) {
        if (o instanceof CaseInsensitiveString cis) return s.equalsIgnoreCase(cis.s);
        if (o instanceof String str) return s.equalsIgnoreCase(str);   // 문제의 분기
        return false;
    }
}
// cis.equals("java") → true 이지만 "java".equals(cis) → false : 대칭성 위반
```

❗️**추이성과 상속**: `Point`를 상속한 `ColorPoint`가 색 필드를 추가하고 `equals`를 재정의하면, `Point`와 `ColorPoint`를 섞어 비교할 때 추이성이 깨진다. 구체 클래스를 상속해 값 필드를 추가하면서 규약을 완벽히 지킬 방법은 없으므로, **상속 대신 컴포지션**을 쓰거나 값 클래스를 `final`로 만든다.

<br>

### 3. hashCode() 규약과 equals와의 연결

| **규약**                                | **내용**                                                                   |
| --------------------------------------- | -------------------------------------------------------------------------- |
| **일관성**                              | `equals`에 쓰이는 정보가 바뀌지 않으면 같은 실행 중에는 항상 같은 값을 반환      |
| **equals가 같으면 hashCode도 같아야 함** | `x.equals(y)`가 `true`면 반드시 `x.hashCode() == y.hashCode()`               |
| **hashCode가 같아도 equals는 다를 수 있음** | 해시 충돌은 허용됨. 단, 다른 객체가 다른 해시를 낼수록 해시 테이블 성능이 좋아짐 |

핵심은 두 번째 조건이다. 논리적으로 같은 객체가 서로 다른 해시 코드를 내면, 해시 기반 컬렉션은 이들을 **다른 버킷**에 넣어 버려 같은 객체로 인식할 수 없게 된다.

<br>

### 4. 해시 기반 컬렉션에서의 동작 원리

`HashMap.get(key)`는 두 단계로 키를 찾는다. 이 흐름을 알면 "왜 둘 다 재정의해야 하는가"가 자명해진다.

```
 get(key)
   ① key.hashCode() → 해시 값으로 버킷 인덱스 계산   (어느 칸을 볼지 결정)
   ② 그 버킷 안의 엔트리들을 순회하며
      (해시 값이 같고) && key.equals(entry.key) 인 것을 찾음   (진짜 같은 키인지 확인)

 hashCode만 다르면 → ①에서 다른 버킷으로 가 버려 equals는 호출조차 안 됨
 equals만 다르면   → 같은 버킷에 도착해도 ②에서 다른 키로 판정
```

```java
// 안티패턴: equals만 재정의 → HashSet이 중복을 잡지 못함
class Coord {
    final int x, y;
    Coord(int x, int y) { this.x = x; this.y = y; }

    @Override
    public boolean equals(Object o) {
        return o instanceof Coord c && c.x == x && c.y == y;
    }
    // hashCode 미재정의 → Object의 주소 기반 해시가 사용됨
}

Set<Coord> set = new HashSet<>();
set.add(new Coord(1, 2));
set.add(new Coord(1, 2));
System.out.println(set.size());                       // 2 (기대는 1)
System.out.println(set.contains(new Coord(1, 2)));    // false
```

> ⚠️ 위 코드는 컴파일도 되고 예외도 없이 조용히 틀린 결과를 낸다. `equals`를 재정의한 클래스에서 `hashCode`를 재정의하지 않으면 IDE와 정적 분석 도구(SpotBugs, Error Prone 등)가 경고하므로 무시하지 말아야 한다. `HashMap` 내부 구조와 충돌 처리는 **unit04** 참고.

<br>

### 5. 올바른 구현 방법

### 5-1. 직접 구현하는 정석

```java
public final class Coord {
    private final int x;
    private final int y;

    public Coord(int x, int y) { this.x = x; this.y = y; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;                       // ① 동일 참조는 즉시 true (성능)
        if (!(o instanceof Coord other)) return false;    // ② 타입 검사 + null 처리 (패턴 매칭, Java 16+)
        return x == other.x && y == other.y;              // ③ 핵심 필드 비교
    }

    @Override
    public int hashCode() {
        return Objects.hash(x, y);                        // equals에 쓴 필드와 동일한 필드 사용
    }
}
```

- `equals`의 매개변수 타입은 반드시 `Object`여야 함. `equals(Coord o)`로 쓰면 **오버라이딩이 아니라 오버로딩**이 되어 컬렉션에서 호출되지 않음 (`@Override`를 붙이면 컴파일러가 잡아 줌)
- `hashCode`는 `equals`에서 비교한 필드만 사용해야 두 번째 규약이 지켜짐. `Objects.hash()`는 가변 인자 배열과 박싱 비용이 있으므로 성능이 민감한 곳에서는 `31 * result + field` 방식으로 직접 계산함
- 부동소수점 필드는 `Double.compare()`, 배열 필드는 `Arrays.equals()`·`Arrays.hashCode()`로 비교함

<br>

### 5-2. record — Java 16 이후의 권장 방식

**record(Java 16 정식)**는 모든 컴포넌트를 기준으로 `equals`·`hashCode`·`toString`을 컴파일러가 자동 생성하므로 규약 위반 위험이 사라진다.

```java
public record Coord(int x, int y) { }

Set<Coord> set = new HashSet<>();
set.add(new Coord(1, 2));
set.add(new Coord(1, 2));
System.out.println(set.size());   // 1 — 자동 생성된 equals/hashCode가 값 기준으로 동작
```

- 값 객체는 record로 정의하는 것이 가장 안전하며, record가 어울리는 설계 기준은 **unit10** 참고
- Lombok `@EqualsAndHashCode`나 IDE 자동 생성도 같은 효과지만, JPA 엔티티처럼 **식별자만으로 동치성을 정의해야 하는 경우**에는 필드를 명시적으로 골라야 함

<br>

### 6. 가변 객체와 해시 컬렉션의 함정

`hashCode`에 쓰이는 필드가 **컬렉션에 넣은 뒤 변경**되면, 객체는 예전 해시로 계산된 버킷에 남아 있지만 새 해시로는 다른 버킷을 찾게 되어 영영 못 찾는 상태가 된다.

```java
class MutableKey {
    String name;
    MutableKey(String name) { this.name = name; }
    @Override public boolean equals(Object o) {
        return o instanceof MutableKey k && k.name.equals(name);
    }
    @Override public int hashCode() { return name.hashCode(); }
}

MutableKey key = new MutableKey("a");
Set<MutableKey> set = new HashSet<>();
set.add(key);
key.name = "b";                          // 해시 값이 바뀜
System.out.println(set.contains(key));   // false — 버킷 위치가 달라짐
set.remove(key);                         // 제거도 안 됨 → 사실상 누수
```

**대응 원칙**

- `Map`의 키·`Set`의 원소로 쓰는 클래스는 **불변**으로 설계함 (`String`, 래퍼 타입, record가 안전한 이유)
- 어쩔 수 없이 가변 객체를 쓴다면 `hashCode`에 **변하지 않는 식별자 필드만** 사용함
- JPA 엔티티는 DB 식별자가 영속화 이전에 `null`이므로, 비즈니스 키를 쓰거나 `id`가 있을 때만 비교하는 등 별도 전략이 필요함

> 💡 `hashCode`가 상수(예: 항상 `1`)를 반환하면 규약상으로는 합법이지만 모든 객체가 한 버킷에 몰려 `HashMap`이 사실상 연결 리스트(Java 8 이후 트리)로 퇴화한다. 규약을 "지키는 것"과 "좋은 해시 함수"는 다른 문제다.

<br>

### 7. 면접·실무 체크포인트

| **질문**                                              | **핵심 답변**                                                                                  |
| ----------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| **`==`와 `equals()`의 차이는?**                        | `==`는 참조(동일성) 비교, `equals()`는 논리적 값(동치성) 비교. `Object` 기본 구현은 둘이 같음      |
| **왜 둘을 항상 함께 재정의해야 하는가?**                 | 해시 컬렉션은 `hashCode`로 버킷을 찾고 `equals`로 확정함. 하나만 바꾸면 **같은 객체를 못 찾음**     |
| **`hashCode`가 같으면 `equals`도 같은가?**              | 아니다. 충돌이 허용되므로 역은 성립하지 않음. **`equals`가 같으면 `hashCode`가 같아야** 함          |
| **`equals(MyType o)`로 작성하면 어떻게 되는가?**         | 오버로딩이 되어 컬렉션이 `Object` 버전을 호출함. `@Override`로 방지                               |
| **가변 객체를 `HashSet`에 넣으면?**                      | 넣은 뒤 해시 필드가 바뀌면 찾을 수도 지울 수도 없음. 키는 불변으로 설계                            |
| **가장 안전한 구현 방법은?**                             | 값 객체는 **record**(Java 16+). 직접 구현 시 `Object` 매개변수·동일 필드 사용·`instanceof` 패턴 매칭 |

- 해시 충돌 처리와 버킷 구조는 **unit04**, 불변 값 객체 설계는 **unit10**에서 이어짐
