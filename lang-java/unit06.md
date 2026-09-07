## 제네릭과 타입 소거

**제네릭(Generics)**은 클래스·메서드가 다룰 타입을 매개변수화해 **컴파일 시점에 타입 안전성을 검사**하고 형변환을 없애 주는 기능이다. 그러나 Java 제네릭은 하위 호환을 위해 **타입 소거(Type Erasure)** 방식으로 구현되어 런타임에는 타입 정보가 사라지며, 이 사실이 `new T()` 불가·`instanceof` 제한·와일드카드의 필요성 같은 Java 특유의 제약을 만든다.

<br>

### 1. 제네릭이 해결하는 문제

Java 5 이전에는 컬렉션이 `Object`를 담았기 때문에 꺼낼 때마다 형변환이 필요했고, 잘못된 타입을 넣어도 **런타임에야** `ClassCastException`으로 드러났다.

```java
// 제네릭 이전 (raw type): 컴파일은 통과, 실행 중 실패
List names = new ArrayList();
names.add("kim");
names.add(42);                              // 아무 경고 없이 들어감
String first = (String) names.get(1);       // ClassCastException — 런타임에 발견

// 제네릭: 잘못된 타입은 컴파일 오류
List<String> names = new ArrayList<>();
names.add("kim");
names.add(42);                              // 컴파일 오류 — 작성 시점에 발견
String first = names.get(0);                // 형변환 불필요
```

- 오류 발견 시점을 **런타임 → 컴파일 타임**으로 앞당기는 것이 제네릭의 본질적 가치임
- `List<String>`에서 `List`는 **제네릭 타입**, `String`은 **타입 인자**, 선언부의 `E`는 **타입 매개변수**라고 부름

> 💡 raw 타입(`List`)은 제네릭 이전 코드와의 호환을 위해 남아 있을 뿐이며, 새 코드에서는 쓰면 안 된다. 컴파일러가 `unchecked` 경고를 내는 코드는 "런타임 `ClassCastException` 가능성이 있다"는 뜻이므로 경고를 없앨 수 없다면 `@SuppressWarnings("unchecked")`와 함께 **안전한 이유를 주석으로** 남긴다.

<br>

### 2. 타입 소거(Type Erasure)

### 2-1. 컴파일러가 하는 일

컴파일러는 제네릭 코드의 타입을 검사한 뒤, 바이트코드에서는 타입 매개변수를 **한정 타입(bound)**으로 바꾸고 필요한 곳에 **형변환을 삽입**한다. 한정이 없으면 `Object`, `<T extends Comparable<T>>`면 `Comparable`로 바뀐다.

```java
// 작성한 코드
public class Box<T> {
    private T value;
    public void set(T value) { this.value = value; }
    public T get() { return value; }
}
Box<String> box = new Box<>();
box.set("a");
String s = box.get();
```

```java
// 소거 후 컴파일러가 실질적으로 만든 코드
public class Box {
    private Object value;
    public void set(Object value) { this.value = value; }
    public Object get() { return value; }
}
Box box = new Box();
box.set("a");
String s = (String) box.get();      // 컴파일러가 형변환 삽입
```

```
 소스 코드 (컴파일 타임)                  바이트코드 (런타임)
 List<String>  ─────┐
 List<Integer> ─────┼──── 타입 소거 ────▶ List  (하나의 클래스, Class 객체도 하나)
 List<User>    ─────┘
```

- 소거 덕분에 제네릭 도입 전에 컴파일된 라이브러리와 **이진 호환**이 유지되었고, 타입 인자마다 클래스를 새로 만들지 않아 클래스 수가 늘지 않음
- 반대로 런타임에는 `List<String>`과 `List<Integer>`가 **완전히 같은 타입**이라 아래 제약이 생김

<br>

### 2-2. 브리지 메서드

제네릭 클래스를 상속하며 타입 인자를 구체화하면, 소거된 시그니처와 재정의한 시그니처가 달라 다형성이 깨진다. 컴파일러는 이를 잇기 위해 **브리지 메서드(bridge method)**를 자동 생성한다.

```java
class StringBox extends Box<String> {
    @Override
    public void set(String value) { ... }          // 소거된 부모의 set(Object)와 시그니처가 다름

    // 컴파일러가 생성하는 브리지 메서드 (synthetic)
    // public void set(Object value) { set((String) value); }
}
```

- 리플렉션으로 메서드 목록을 보면 `set(Object)`와 `set(String)`이 둘 다 보이는 이유임
- `StringBox`를 raw 타입으로 쓰며 `set(42)`를 호출하면 브리지 메서드의 형변환에서 `ClassCastException`이 남 — 컴파일러가 막지 못하는 지점을 브리지가 보완함

<br>

### 3. 런타임 제약 — 소거 때문에 안 되는 것들

| **제약**                                    | **예시**                                    | **이유·우회 방법**                                                  |
| ------------------------------------------- | ------------------------------------------- | ------------------------------------------------------------------- |
| **타입 매개변수로 인스턴스 생성 불가**         | `new T()`, `new T[10]`                       | 런타임에 `T`가 뭔지 모름 → `Class<T>` 또는 `Supplier<T>`를 받아 생성     |
| **기본 타입을 타입 인자로 사용 불가**          | `List<int>`                                  | 소거 후 `Object`여야 하므로 → `List<Integer>` (박싱 비용 발생)          |
| **매개변수화 타입에 `instanceof` 불가**        | `obj instanceof List<String>`                | 런타임에 구분 불가 → `obj instanceof List<?>`만 가능                    |
| **제네릭 배열 생성 불가**                      | `new List<String>[3]`                        | 배열은 런타임 타입 검사를 하는데 소거로 불가 → `List<?>[]` 후 형변환      |
| **static 문맥에서 타입 매개변수 사용 불가**     | `static T value;`                            | `T`는 인스턴스마다 다르지만 static은 클래스당 하나                       |
| **소거 후 같은 시그니처로 오버로딩 불가**       | `void f(List<String>)`와 `void f(List<Integer>)` | 둘 다 `f(List)`로 소거되어 충돌                                    |
| **제네릭 예외 클래스 불가**                    | `class MyEx<T> extends Exception`            | `catch` 절은 런타임 타입 검사가 필요                                    |

```java
// 안티패턴: 컴파일 불가
class Repository<T> {
    T create() { return new T(); }                 // 오류
}

// 개선: 타입 토큰(Class<T>) 또는 팩토리를 주입
class Repository<T> {
    private final Supplier<T> factory;
    Repository(Supplier<T> factory) { this.factory = factory; }
    T create() { return factory.get(); }
}
Repository<User> repo = new Repository<>(User::new);
```

> ⚠️ 소거는 "완전한 삭제"가 아니다. 클래스·필드·메서드 선언의 제네릭 시그니처는 클래스 파일의 `Signature` 속성에 남아 리플렉션(`getGenericSuperclass()` 등)으로 읽을 수 있다. Jackson의 `TypeReference<List<User>>() {}`처럼 **익명 하위 클래스를 만들어 부모의 타입 인자를 읽어 내는 기법(슈퍼 타입 토큰)**이 동작하는 이유다. 사라지는 것은 **인스턴스와 지역 변수의 타입 인자**다.

<br>

### 4. 불공변과 와일드카드

### 4-1. 배열은 공변, 제네릭은 불공변

`Integer`가 `Number`의 하위 타입이어도 `List<Integer>`는 `List<Number>`의 하위 타입이 **아니다**(불공변, invariant). 반면 배열은 `Integer[]`가 `Number[]`의 하위 타입이다(공변, covariant).

```java
Number[] nums = new Integer[2];
nums[0] = 3.14;                       // 컴파일 통과, 런타임 ArrayStoreException — 배열은 런타임에 검사

List<Number> list = new ArrayList<Integer>();   // 컴파일 오류 — 제네릭은 컴파일 타임에 차단
```

- 제네릭이 불공변인 이유는 소거 후 런타임 검사가 불가능하므로 **컴파일 타임에 잘못된 삽입을 원천 차단**해야 하기 때문임
- 그러나 불공변만으로는 "`Number`의 모든 하위 타입 리스트를 받는 메서드"를 만들 수 없어 **와일드카드**가 필요함

<br>

### 4-2. 와일드카드 세 가지와 PECS

| **형태**              | **의미**                             | **읽기(get)**                | **쓰기(add)**                          | **용도**                    |
| --------------------- | ------------------------------------ | ---------------------------- | -------------------------------------- | --------------------------- |
| **`List<?>`**          | 어떤 타입인지 모르는 리스트            | `Object`로만 받음             | `null` 외 불가                          | 타입에 무관한 연산(size 등)   |
| **`List<? extends T>`** | `T` 또는 `T`의 하위 타입 리스트        | **`T`로 안전하게 읽음**        | 불가 (실제 타입을 모르므로)               | **생산자(Producer)**        |
| **`List<? super T>`**   | `T` 또는 `T`의 상위 타입 리스트        | `Object`로만 받음             | **`T`(및 하위)를 안전하게 넣음**          | **소비자(Consumer)**        |

**PECS(Producer-Extends, Consumer-Super)** — 매개변수가 데이터를 **내주면** `extends`, **받아들이면** `super`를 쓴다.

```java
// 안티패턴: 불공변 때문에 List<Integer>를 넘길 수 없음
static double sum(List<Number> nums) { ... }
sum(List.of(1, 2, 3));                       // 컴파일 오류

// 개선: 생산자 → extends
static double sum(List<? extends Number> nums) {
    double total = 0;
    for (Number n : nums) total += n.doubleValue();   // 읽기는 Number로 안전
    return total;
}
sum(List.of(1, 2, 3));                       // List<Integer> 가능

// 소비자 → super : Integer를 넣을 수 있는 어떤 리스트든 받음
static void fill(List<? super Integer> dst) {
    for (int i = 0; i < 3; i++) dst.add(i);
}
fill(new ArrayList<Number>());               // 가능
fill(new ArrayList<Object>());               // 가능
```

```
 List<? extends Number>                  List<? super Integer>
   ┌────────┐                              ┌────────┐
   │ Number │ ◀── 읽을 때 상한: Number       │ Object │
   │ Integer│                              │ Number │ ◀── 넣을 때 하한: Integer
   │ Double │  (넣기는 불가)                  │ Integer│
   └────────┘                              └────────┘
```

- JDK의 `Collections.copy(List<? super T> dest, List<? extends T> src)`가 PECS의 교과서적 예시임
- 읽고 쓰기를 모두 해야 하면 와일드카드 대신 **타입 매개변수 `<T>`**를 사용함

<br>

### 5. 힙 오염과 제네릭 가변 인자

제네릭 변수가 **선언과 다른 타입의 객체를 가리키는 상태**를 **힙 오염(Heap Pollution)**이라 한다. raw 타입 혼용, unchecked 형변환, 제네릭 가변 인자에서 발생한다.

```java
// 가변 인자 T...는 내부적으로 배열 T[]를 만드는데, 제네릭 배열 생성이 불가하므로 Object[]로 소거됨
static void dangerous(List<String>... lists) {
    Object[] arr = lists;                  // 배열 공변으로 허용
    arr[0] = List.of(42);                  // List<Integer>가 들어감 — 힙 오염
    String s = lists[0].get(0);            // ClassCastException
}
```

- 가변 인자 배열에 **아무것도 저장하지 않고 외부로 노출하지 않으면** 안전하며, 그 경우 `@SafeVarargs`로 경고를 억제함 (`Arrays.asList`, `List.of`가 그렇게 선언됨)
- `@SafeVarargs`는 재정의될 수 없는 메서드(`static`, `final`, `private`, 생성자)에만 붙일 수 있음

> 💡 `<T extends Comparable<? super T>>`처럼 복잡해 보이는 한정은 "`T`가 자기 자신 또는 상위 타입과 비교 가능하면 된다"는 뜻이다. `Collections.sort`·`Collections.max`의 시그니처가 이 형태이며, 부모가 `Comparable`을 구현하고 자식은 구현하지 않은 경우까지 포용하기 위한 것이다.

<br>

### 6. 정리

| **항목**                    | **핵심**                                                                                          |
| --------------------------- | ------------------------------------------------------------------------------------------------- |
| **제네릭의 목적**            | 타입 오류를 **컴파일 타임**에 발견하고 형변환을 제거                                                   |
| **타입 소거**                | 컴파일 후 타입 매개변수는 한정 타입(기본 `Object`)으로 대체되고 형변환이 삽입됨. 하위 호환이 이유           |
| **런타임 제약**              | `new T()`·`List<int>`·`instanceof List<String>`·제네릭 배열·static `T`·소거 후 동일 시그니처 오버로딩 불가 |
| **브리지 메서드**            | 소거된 부모 시그니처와 자식 재정의를 잇기 위해 컴파일러가 생성                                            |
| **불공변**                   | `List<Integer>`는 `List<Number>`가 아님. 배열은 공변이라 `ArrayStoreException`이 런타임에 발생            |
| **PECS**                     | 생산자(읽기)는 `? extends T`, 소비자(쓰기)는 `? super T`, 둘 다면 `<T>`                                 |
| **힙 오염**                  | raw 타입·unchecked 형변환·제네릭 가변 인자에서 발생. `@SafeVarargs`는 안전을 확인한 뒤에만                 |

- 함수형 인터페이스(`Function<T, R>` 등)의 시그니처에 와일드카드가 어떻게 쓰이는지는 **unit07**에서 이어짐
