## JPA 쓰기와 식별자 전략

JPA의 쓰기 작업은 영속성 컨텍스트를 거치는 것(persist·변경 감지)과 **컨텍스트를 우회해 DB에 직접 실행하는 것**(벌크 연산)이 섞여 있어, 둘 사이의 불일치를 모르면 갱신한 값이 조회에 반영되지 않는 버그가 생긴다. 여기에 **식별자 생성 전략**은 INSERT 시점과 배치 성능을 좌우하고, **영속성 전이(Cascade)**는 부모-자식 생명주기를 결정하므로 세 주제를 묶어 정리한다.

<br>

### 1. save()의 실제 동작 — persist인가 merge인가

Spring Data JPA의 `SimpleJpaRepository.save()`는 엔티티가 **새 것인지(isNew)** 판단해 `persist()` 또는 `merge()`를 호출한다.

```
save(entity)
   ├─ isNew(entity) == true  → em.persist(entity)         INSERT (쓰기 지연)
   └─ isNew(entity) == false → em.merge(entity)           SELECT 후 INSERT 또는 UPDATE
```

| **isNew 판단 기준**                         | **결과**                                                |
| ------------------------------------------- | ------------------------------------------------------- |
| `@Id` 필드가 `null` (참조형) 또는 `0` (기본형) | 새 엔티티 → **persist**                                 |
| `@Version` 필드가 `null`                    | 새 엔티티 → persist                                     |
| `Persistable<ID>` 구현 시 `isNew()` 반환값   | 개발자가 직접 지정                                       |
| 그 외 (직접 할당한 ID가 있음)                | 기존 엔티티로 간주 → **merge → SELECT 1회 추가**         |

```java
// 함정: UUID·문자열 ID를 직접 할당하면 save()가 merge로 동작 → 불필요한 SELECT 발생
@Entity
public class Coupon implements Persistable<String> {
    @Id private String code;

    @CreatedDate
    private LocalDateTime createdAt;

    @Override
    public boolean isNew() { return createdAt == null; }   // 아직 저장 전이면 persist로 유도
}
```

- `merge()`의 반환값이 영속 객체이고 인자 객체는 준영속으로 남는다는 점, 그리고 `null` 필드까지 덮어쓴다는 점은 unit06 참고

<br>

### 2. 벌크 연산과 영속성 컨텍스트 불일치

JPQL의 `update`/`delete`는 **영속성 컨텍스트를 거치지 않고 DB에 바로 실행**된다. 따라서 1차 캐시에 이미 올라온 엔티티는 벌크 연산 결과를 모른다.

```
① Member m = findById(1)      → 1차 캐시: { 1 → Member(age=20) }
② bulk update: age = age + 1  → DB: age=21   (1차 캐시는 여전히 20)
③ m.getAge()                  → 20  ✗  (DB와 불일치)
④ findById(1)                 → 1차 캐시에서 반환 → 여전히 20  ✗
```

```java
// 안티패턴: 벌크 연산 후 같은 트랜잭션에서 엔티티를 계속 사용
@Modifying
@Query("update Member m set m.age = m.age + 1 where m.age < :age")
int bulkAgePlus(@Param("age") int age);

// 개선: 실행 전 flush + 실행 후 clear로 컨텍스트를 DB와 맞춤
@Modifying(flushAutomatically = true, clearAutomatically = true)
@Query("update Member m set m.age = m.age + 1 where m.age < :age")
int bulkAgePlus(@Param("age") int age);
```

- `clearAutomatically = true`: 벌크 실행 후 `em.clear()` → 이후 조회는 DB에서 새로 읽음. 단, **이전에 조회해 둔 엔티티 변수는 준영속**이 되므로 수정해도 반영되지 않음
- `flushAutomatically = true`: 벌크 실행 전 `em.flush()` → 아직 DB에 나가지 않은 변경이 벌크 연산에 덮이거나 누락되는 것을 방지
- 벌크 연산은 **엔티티 생명주기 콜백·`@Version` 증가·Cascade가 적용되지 않으므로**, 이런 부가 동작이 필요하면 엔티티를 조회해 변경 감지로 처리해야 함

> ⚠️ `deleteAll()`은 엔티티를 하나씩 조회해 `remove()`하므로 N개의 DELETE와 Cascade가 동작하지만, `deleteAllInBatch()`는 JPQL 한 번으로 지우며 Cascade와 영속성 컨텍스트 동기화가 **전혀 일어나지 않는다**. 대량 삭제는 후자를 쓰되 자식 테이블 정리는 직접 책임져야 한다.

<br>

### 3. 식별자 생성 전략

| **전략**       | **동작**                                                | **INSERT 시점**                 | **배치 INSERT** | **비고**                                        |
| -------------- | ------------------------------------------------------- | ------------------------------- | --------------- | ----------------------------------------------- |
| **IDENTITY**   | DB의 AUTO_INCREMENT에 위임                              | **persist() 즉시** (ID를 알아야 하므로) | **불가**   | MySQL에서 가장 흔함                             |
| **SEQUENCE**   | DB 시퀀스에서 ID를 미리 받아옴                          | flush 시 (쓰기 지연 유지)       | **가능**        | PostgreSQL·Oracle·H2. `allocationSize`로 왕복 감소 |
| **TABLE**      | 키 전용 테이블로 시퀀스 흉내                             | flush 시                        | 가능            | 모든 DB 지원하나 **락 경합으로 느림**           |
| **UUID**       | 애플리케이션에서 UUID 생성 (JPA 3.1 / Hibernate 6.2+)   | flush 시                        | 가능            | 분산 환경 유리, 인덱스 크기·정렬 불리           |
| **AUTO**       | DB 방언(Dialect)에 따라 위 전략 중 자동 선택            | 전략에 따름                     | 전략에 따름     | Hibernate 6은 시퀀스 우선, 미지원 DB는 TABLE로 대체 (버전에 따라 다름) |

```java
// SEQUENCE: allocationSize만큼 ID 범위를 한 번에 받아 메모리에서 배분 → DB 왕복 감소
@Entity
@SequenceGenerator(name = "order_seq", sequenceName = "order_seq", allocationSize = 50)
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_seq")
    private Long id;
}
```

- `allocationSize`(기본 50)는 **DB 시퀀스의 `INCREMENT BY`와 일치**해야 함. 다르면 ID 충돌이나 큰 공백이 생김
- **IDENTITY는 쓰기 지연이 깨진다**: persist 시 INSERT를 바로 실행해야 ID를 얻을 수 있어, JDBC 배치로 묶을 수 없음. 대량 INSERT가 필요한 MySQL 환경에서는 JDBC `batchUpdate`나 UUID·직접 채번을 검토함
- `equals/hashCode`를 ID 기반으로 구현하면 persist 전(ID가 `null`)에 `Set`에 넣은 엔티티가 저장 후 해시가 바뀌어 찾을 수 없게 되므로, 비즈니스 키를 쓰거나 ID가 `null`일 때의 처리를 정의해야 함

> 💡 "IDENTITY와 SEQUENCE의 차이"는 **"INSERT 시점이 다르고, 그래서 배치 INSERT 가능 여부가 갈린다"**로 답한다. 왜 IDENTITY는 즉시 INSERT해야 하는지(영속성 컨텍스트는 ID를 키로 엔티티를 관리하므로 persist 시점에 ID가 필요함)까지 설명하면 좋다.

<br>

### 4. 배치 INSERT/UPDATE 설정

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc.batch_size: 100          # JDBC 배치 크기
        order_inserts: true           # 같은 테이블 INSERT를 모아 배치 효율 향상
        order_updates: true
  datasource:
    url: jdbc:mysql://localhost:3306/app?rewriteBatchedStatements=true   # MySQL: 여러 INSERT를 다중 VALUES로 재작성
```

- 쓰기 지연 저장소에 쌓인 SQL을 flush 시 JDBC `addBatch()`로 묶어 전송함 (unit06 참고)
- MySQL은 `rewriteBatchedStatements=true`가 없으면 배치를 받아도 한 건씩 전송하므로 효과가 없음
- 대량 삽입 루프에서는 `batch_size` 단위로 `flush()` + `clear()`를 호출해 1차 캐시 메모리를 정리함

<br>

### 5. 영속성 전이(Cascade)와 고아 객체 제거

**Cascade**는 부모 엔티티에 대한 영속성 작업(persist·remove 등)을 **연관된 자식에게 전파**하는 기능이다.

| **옵션**                | **전파되는 작업**                       | **설명**                                                |
| ----------------------- | --------------------------------------- | ------------------------------------------------------- |
| **PERSIST**             | `persist()`                             | 부모 저장 시 자식도 함께 저장                            |
| **REMOVE**              | `remove()`                              | 부모 삭제 시 자식도 함께 삭제                            |
| **MERGE**               | `merge()`                               | 부모 병합 시 자식도 병합                                 |
| **REFRESH / DETACH**    | `refresh()` / `detach()`                | 상태 동기화·분리 전파                                    |
| **ALL**                 | 위 전부                                 | 부모가 자식의 생명주기를 **완전히 소유**할 때만 사용     |
| **orphanRemoval=true**  | 컬렉션에서 **제거된 자식**을 DELETE      | Cascade와 별개 옵션. `REMOVE`는 부모 삭제 시, orphanRemoval은 **참조가 끊길 때** |

```java
@Entity
public class Order {
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> orderItems = new ArrayList<>();

    public void removeItem(OrderItem item) {
        orderItems.remove(item);       // orphanRemoval → flush 시 DELETE order_item
    }
}

// 사용: 부모만 저장하면 자식까지 INSERT
Order order = new Order();
order.addItem(new OrderItem(book, 2));
orderRepository.save(order);           // INSERT orders, INSERT order_item (cascade PERSIST)
```

**적용 기준**

- 자식이 **부모 없이는 의미가 없고**(주문-주문상품), **다른 엔티티가 자식을 참조하지 않을 때**만 `ALL + orphanRemoval`을 사용함. 이 조합은 사실상 부모가 자식의 리포지토리 역할을 함 (DDD의 애그리거트 개념)
- 자식이 여러 부모에서 공유되거나(회원-주문의 회원 쪽) 독립적인 생명주기를 가지면 Cascade를 쓰지 않음
- `CascadeType.REMOVE`·`orphanRemoval`은 자식을 **하나씩 조회해 DELETE**하므로 대량 삭제 시 성능에 주의

> ⚠️ Cascade는 **연관관계 주인과 무관**하다. `mappedBy` 쪽(비주인)에 걸어도 전파는 동작하지만, 자식의 FK(`item.setOrder(this)`)를 세팅하지 않으면 FK가 `null`인 채 INSERT된다. 양방향 편의 메서드로 양쪽을 모두 연결해야 한다 (unit07 참고).

<br>

### 6. 삭제 관련 API 비교

| **메서드**               | **동작**                                          | **엔티티 없을 때**                                     |
| ------------------------ | ------------------------------------------------- | ------------------------------------------------------ |
| **delete(entity)**       | 준영속이면 merge 후 `remove()`                    | -                                                      |
| **deleteById(id)**       | `findById` 후 있으면 `delete()` → **SELECT + DELETE** | Spring Data 3.0부터 **조용히 무시** (2.x는 `EmptyResultDataAccessException`) |
| **deleteAll()**          | 전체 조회 후 하나씩 `remove()` (Cascade 동작)     | -                                                      |
| **deleteAllInBatch()**   | JPQL `delete` 1회 (Cascade·컨텍스트 동기화 없음)   | -                                                      |
| **@Modifying delete**    | JPQL 벌크 삭제, `clearAutomatically` 권장          | 삭제 건수 반환                                          |

<br>

### 7. 면접·실무 체크포인트

- `save()`는 **isNew 여부**로 persist/merge를 고르며, ID를 직접 할당하면 merge → SELECT가 추가된다 (`Persistable`로 해결)
- 벌크 연산은 **영속성 컨텍스트를 우회**하므로 `@Modifying(clearAutomatically = true)`로 1차 캐시를 비우고, 필요 시 `flushAutomatically`로 미반영 변경을 먼저 내보낸다
- **IDENTITY는 persist 즉시 INSERT** → 쓰기 지연·배치 INSERT 불가. **SEQUENCE는 allocationSize**로 왕복을 줄이고 배치가 가능하다
- 배치 INSERT는 `jdbc.batch_size` + `order_inserts` + (MySQL) `rewriteBatchedStatements`가 함께 필요하다
- Cascade는 **부모가 자식의 생명주기를 온전히 소유할 때만** `ALL + orphanRemoval`, 공유되는 엔티티에는 쓰지 않는다
- `deleteAllInBatch()`·벌크 delete는 Cascade가 동작하지 않으므로 자식 정리를 직접 책임진다
- 영속성 컨텍스트의 기본 동작은 unit06, 조회 성능은 unit07 참고
