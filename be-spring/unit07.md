## 연관관계와 N+1

JPA는 객체의 참조(`order.getMember()`)를 SQL 조인으로 번역해 주지만, **언제 연관 데이터를 가져올지**(로딩 전략)를 잘못 설정하면 목록 하나를 조회하는 데 수백 개의 쿼리가 나가는 **N+1 문제**가 생긴다. 지연·즉시 로딩의 동작과 fetch join·EntityGraph·batch size의 차이, 그리고 컬렉션 페이징 함정까지 알아야 JPA 성능 문제를 진단하고 고칠 수 있다.

<br>

### 1. 연관관계 매핑과 기본 로딩 전략

| **애노테이션** | **관계**  | **기본 FetchType** | **비고**                                        |
| -------------- | --------- | ------------------ | ----------------------------------------------- |
| **@ManyToOne** | N : 1     | **EAGER**          | 가장 흔한 매핑, FK를 가진 쪽이 연관관계 주인    |
| **@OneToOne**  | 1 : 1     | **EAGER**          | FK가 없는 쪽(mappedBy)은 프록시 불가 → 항상 즉시 로딩됨 |
| **@OneToMany** | 1 : N     | LAZY               | 컬렉션, mappedBy로 주인 지정                    |
| **@ManyToMany**| N : M     | LAZY               | 실무에서는 중간 엔티티로 풀어 쓰는 것을 권장    |

- **연관관계 주인**: 외래 키를 관리하는 쪽. 주인이 아닌 쪽(`mappedBy`)의 변경은 DB에 반영되지 않으므로, 양방향이면 **양쪽 다 세팅하는 편의 메서드**를 둠
- **지연 로딩(LAZY)**: 연관 엔티티 자리에 **프록시 객체**를 넣어 두고, 실제 필드에 접근하는 순간 SELECT를 실행함
- **즉시 로딩(EAGER)**: 엔티티를 조회할 때 연관 엔티티까지 **함께** 조회함. `find()`는 조인으로 한 번에 가져오지만, **JPQL은 먼저 본 엔티티를 조회한 뒤 연관 엔티티를 개별 SELECT**하므로 N+1이 발생함

```java
@Entity
public class Order {
    @Id @GeneratedValue
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)          // 기본값 EAGER를 반드시 LAZY로 변경
    @JoinColumn(name = "member_id")
    private Member member;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> orderItems = new ArrayList<>();

    public void addItem(OrderItem item) {         // 양방향 편의 메서드
        orderItems.add(item);
        item.setOrder(this);
    }
}
```

> 💡 "모든 연관관계를 LAZY로 두고, 필요한 곳에서만 fetch join으로 가져온다"가 JPA의 기본 원칙이다. EAGER는 "어디서 어떤 쿼리가 나갈지 예측할 수 없다"는 점이 가장 큰 문제이며, 면접에서 "왜 LAZY를 기본으로 하는가"를 물으면 이 예측 가능성을 답한다.

<br>

### 2. N+1 문제의 발생 과정

```
List<Order> orders = em.createQuery("select o from Order o", Order.class).getResultList();
      → SELECT * FROM orders                         ... 1회  (N건 반환)

for (Order o : orders) {
    o.getMember().getName();                         // LAZY 프록시 초기화
}
      → SELECT * FROM member WHERE id = 1           ┐
      → SELECT * FROM member WHERE id = 2           ├ N회
      → ...                                         ┘
```

- 이름의 의미: 목록 조회 **1회** + 연관 엔티티 조회 **N회**
- LAZY는 접근 시점에, EAGER는 JPQL 실행 직후에 N번 나가므로 **로딩 전략을 바꾸는 것만으로는 해결되지 않음**
- 지연 로딩 시점에 영속성 컨텍스트가 닫혀 있으면 N+1 대신 `LazyInitializationException`이 발생함 (OSIV·컨텍스트 범위는 unit06 참고)

<br>

### 3. 해결법 ① fetch join

JPQL의 `join fetch`는 연관 엔티티를 **한 번의 조인 쿼리로 함께 조회해 영속 상태로 만든다**.

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    @Query("select o from Order o join fetch o.member")               // ToOne fetch join
    List<Order> findAllWithMember();

    @Query("select o from Order o join fetch o.orderItems oi join fetch oi.item")   // 컬렉션 fetch join
    List<Order> findAllWithItems();
}
```

- 일반 `join`은 조인 조건으로 필터링만 하고 연관 엔티티를 **로딩하지 않는다**. `join fetch`만 로딩함
- 컬렉션 fetch join은 1:N 조인 결과로 **부모 행이 자식 수만큼 중복**됨. Hibernate 5까지는 `select distinct o`로 중복을 제거해야 했지만, **Hibernate 6부터는 자동으로 중복이 제거**되어 `distinct`가 필요 없음
- fetch join 대상에는 별칭(alias)을 붙여 `where`에 쓰지 않는 것이 원칙 — 컬렉션 일부만 로딩된 엔티티가 영속 상태가 되어 데이터 정합성이 깨짐

> ⚠️ `List` 타입 컬렉션 두 개 이상을 동시에 fetch join하면 `MultipleBagFetchException`이 발생한다. 카테시안 곱으로 데이터가 폭증하기 때문이다. 컬렉션 fetch join은 **하나만** 하고 나머지는 batch size로 해결하거나, 컬렉션 타입을 `Set`으로 바꾼다.

<br>

### 4. 해결법 ② @EntityGraph

JPQL을 직접 쓰지 않고 **애노테이션으로 함께 로딩할 속성을 선언**하는 방식이다. 메서드 이름 기반 쿼리에도 붙일 수 있다.

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    @EntityGraph(attributePaths = {"member", "delivery"})
    List<Order> findByStatus(OrderStatus status);          // LEFT OUTER JOIN으로 member, delivery 함께 로딩

    @EntityGraph(attributePaths = "member")
    @Query("select o from Order o where o.createdAt >= :since")
    List<Order> findRecent(@Param("since") LocalDateTime since);
}
```

| **항목**            | **fetch join (JPQL)**                 | **@EntityGraph**                              |
| ------------------- | ------------------------------------- | --------------------------------------------- |
| **조인 종류**       | 기본 **INNER JOIN** (`left join fetch` 가능) | **LEFT OUTER JOIN**                         |
| **선언 위치**       | JPQL 문자열 안                        | 애노테이션 (메서드 이름 쿼리와 조합 가능)      |
| **복잡한 조건**     | 자유로움                              | 단순 로딩 선언에 한정                          |
| **컬렉션 다중 로딩**| `MultipleBagFetchException` 동일       | 동일하게 발생                                  |

<br>

### 5. 해결법 ③ batch size — 컬렉션에 적합

`hibernate.default_batch_fetch_size`(전역) 또는 `@BatchSize`(개별)를 설정하면, 지연 로딩이 일어날 때 **같은 타입의 프록시·컬렉션을 식별자 IN 절로 묶어서** 한 번에 가져온다.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 100     # 보통 100 ~ 1000 사이, DB의 IN 절 한계와 메모리를 고려
```

```
설정 전:  SELECT * FROM order_item WHERE order_id = 1
          SELECT * FROM order_item WHERE order_id = 2
          ... (N회)

설정 후:  SELECT * FROM order_item WHERE order_id IN (1, 2, 3, ..., 100)   ← 1회 (100건 단위)
```

- 쿼리 수가 N에서 **N / batch_size**로 줄고, 조인이 아니므로 **부모 데이터 중복·페이징 문제가 없음**
- fetch join처럼 즉시 가져오는 것이 아니라 **지연 로딩이 발생하는 시점에** 묶어서 가져오는 방식이므로, 영속성 컨텍스트가 살아 있어야 함
- 실무 권장 조합: **ToOne 관계는 fetch join, ToMany(컬렉션) 관계는 batch size**

<br>

### 6. 컬렉션 페이징 함정

컬렉션을 fetch join한 JPQL에 `Pageable`이나 `setFirstResult/setMaxResults`를 적용하면, Hibernate는 **모든 데이터를 메모리로 읽은 뒤 애플리케이션에서 페이징**한다. 조인된 행 수와 부모 수가 다르기 때문에 SQL `LIMIT`으로는 정확한 부모 개수를 자를 수 없기 때문이다.

```
WARN HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory
```

```java
// 안티패턴: 컬렉션 fetch join + 페이징 → 전체 데이터를 메모리에 올림 (OOM 위험)
@Query("select o from Order o join fetch o.orderItems")
Page<Order> findAllWithItems(Pageable pageable);

// 개선: ToOne만 fetch join하고 페이징, 컬렉션은 batch size로 지연 로딩
@Query(value = "select o from Order o join fetch o.member",
       countQuery = "select count(o) from Order o")
Page<Order> findAllWithMember(Pageable pageable);
// + hibernate.default_batch_fetch_size: 100 → o.getOrderItems() 접근 시 IN 쿼리로 묶어 로딩
```

- 설정 `hibernate.query.fail_on_pagination_over_collection_fetch=true`를 켜면 경고 대신 예외를 던져 배포 전에 발견할 수 있음
- ToOne 관계는 행 수가 늘지 않으므로 fetch join + 페이징이 안전함

> 💡 "컬렉션 페이징은 어떻게 처리하는가"는 JPA 면접의 대표 심화 질문이다. **"ToOne은 fetch join, 컬렉션은 batch size, 그리고 페이징은 부모 기준으로"** 라는 공식을 이유(행 중복 → 메모리 페이징)와 함께 설명한다.

<br>

### 7. 선택 기준 정리

| **상황**                                    | **권장 방법**                                   | **이유**                                      |
| ------------------------------------------- | ----------------------------------------------- | --------------------------------------------- |
| **ToOne 연관 엔티티가 항상 필요**            | `join fetch` 또는 `@EntityGraph`                | 행 수 변화 없음, 쿼리 1회                      |
| **컬렉션 연관 + 페이징 없음**                | 컬렉션 하나만 `join fetch`, 나머지는 batch size | `MultipleBagFetchException`·중복 회피          |
| **컬렉션 연관 + 페이징 필요**                | ToOne만 fetch join + **batch size**             | 메모리 페이징 방지                             |
| **화면 전용 데이터, 엔티티 불필요**          | DTO 프로젝션 (`select new ...`, QueryDSL)        | 필요한 컬럼만 조회, 영속성 컨텍스트 부담 없음 |
| **조건에 따라 로딩 여부가 달라짐**           | LAZY + 필요한 메서드에서만 fetch join            | 예측 가능한 쿼리                               |

- 연관관계는 **모두 LAZY**로 두고, 조회 메서드 단위로 로딩 전략을 결정한다
- N+1은 로딩 전략이 아니라 **조회 방식(fetch join·EntityGraph·batch size)** 으로 해결한다
- 쿼리 로그(`spring.jpa.show-sql`, `p6spy`)와 테스트에서 쿼리 횟수를 검증하는 습관이 N+1을 조기에 잡는 가장 확실한 방법이다
- 영속성 컨텍스트·지연 로딩 예외는 unit06, 쓰기 측 성능(벌크·배치 INSERT)은 unit08 참고
