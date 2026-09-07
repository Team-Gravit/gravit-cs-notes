## 영속성 컨텍스트

**영속성 컨텍스트(Persistence Context)**는 JPA가 엔티티를 보관·추적하는 **메모리 상의 작업 공간**으로, 1차 캐시·변경 감지·쓰기 지연이 모두 여기서 일어난다. `save()`를 안 불렀는데 UPDATE가 나가고, 조회했는데 SELECT가 안 나가는 "마법 같은" 동작의 원인이 전부 영속성 컨텍스트이므로, JPA를 쓴다면 이 구조를 정확히 이해해야 한다.

<br>

### 1. EntityManager와 영속성 컨텍스트

- `EntityManagerFactory`는 애플리케이션 전체에서 하나(생성 비용이 큼), `EntityManager`는 **요청·트랜잭션 단위로** 생성되는 가벼운 객체이며 스레드 간 공유하면 안 됨
- 영속성 컨텍스트는 EntityManager를 통해 접근하는 논리적 개념으로, 스프링에서는 **트랜잭션 범위와 생명주기가 같음**(트랜잭션 시작 시 생성, 종료 시 닫힘)
- 스프링이 주입하는 `EntityManager`는 실제 객체가 아니라 **공유 프록시**로, 호출 시점의 현재 트랜잭션에 묶인 진짜 EntityManager를 찾아 위임함. 덕분에 싱글톤 리포지토리에 주입해도 안전함

```
[EntityManagerFactory] ──생성──▶ [EntityManager]  ⇄  [영속성 컨텍스트]
                                                       ├─ 1차 캐시  { id → 엔티티, 스냅샷 }
                                                       └─ 쓰기 지연 SQL 저장소 [ INSERT, UPDATE, DELETE ... ]
                                                                  │ flush
                                                                  ▼
                                                              [ Database ]
```

<br>

### 2. 엔티티 생명주기

| **상태**              | **의미**                                             | **진입 방법**                          | **변경 감지** |
| --------------------- | ---------------------------------------------------- | -------------------------------------- | ------------- |
| **비영속 (new)**      | 순수 자바 객체, 컨텍스트와 무관                       | `new Member()`                         | X             |
| **영속 (managed)**    | 컨텍스트가 **관리 중**                                | `persist()`, `find()`, JPQL 조회        | **O**         |
| **준영속 (detached)** | 관리되다가 **분리됨**, 식별자는 있음                  | `detach()`, `clear()`, `close()`, 트랜잭션 종료 | X    |
| **삭제 (removed)**    | 삭제 예약됨, flush 시 DELETE                           | `remove()`                             | -             |

```java
Member member = new Member("kim");     // 비영속
em.persist(member);                    // 영속 (아직 INSERT 안 나감, IDENTITY 전략이면 즉시 나감 — unit08 참고)
member.setName("lee");                 // 영속 상태의 변경 → flush 시 UPDATE 자동 생성
em.detach(member);                     // 준영속 → 이후 변경은 반영되지 않음
Member merged = em.merge(member);      // 준영속 → 영속 (반환된 merged가 영속 객체, member는 여전히 준영속)
```

> ⚠️ `merge()`는 넘긴 객체를 영속 상태로 바꾸는 것이 아니라, **DB(또는 1차 캐시)에서 조회한 영속 엔티티에 값을 복사해 그 엔티티를 반환**한다. 인자로 넘긴 객체를 계속 수정해도 반영되지 않는다. 또한 넘긴 객체의 `null` 필드까지 그대로 덮어쓰므로 "일부 필드만 수정" 용도로 쓰면 데이터가 지워진다.

<br>

### 3. 1차 캐시와 동일성 보장

- 영속성 컨텍스트는 `Map<식별자, 엔티티>` 형태의 **1차 캐시**를 가지며, `find()`는 먼저 여기를 조회하고 없을 때만 SELECT를 실행함
- 같은 트랜잭션 안에서 같은 식별자로 두 번 조회하면 **같은 인스턴스**(`==` 참조 동일성)가 반환됨 → REPEATABLE READ 수준의 일관성을 애플리케이션 레벨에서 제공함
- 범위는 트랜잭션 하나이므로 성능 향상 효과는 제한적이며, 여러 요청에 걸친 캐시는 2차 캐시나 별도 캐시(Redis 등)의 영역임

```java
@Transactional
public void sameInstance(Long id) {
    Member a = memberRepository.findById(id).orElseThrow();   // SELECT 1회
    Member b = memberRepository.findById(id).orElseThrow();   // 1차 캐시 → SELECT 없음
    System.out.println(a == b);                               // true
}
```

- **JPQL은 1차 캐시를 거치지 않고 항상 DB에 질의**한다. 다만 결과를 영속성 컨텍스트에 넣을 때 이미 같은 식별자의 엔티티가 있으면 **DB에서 가져온 값을 버리고 기존 인스턴스를 유지**함 → 동일성은 보장되지만, 다른 트랜잭션의 커밋 내용이 반영되지 않는 것처럼 보일 수 있음

<br>

### 4. 변경 감지(Dirty Checking)와 쓰기 지연

**변경 감지**는 영속 상태 엔티티의 값이 바뀌면 flush 시점에 JPA가 알아서 UPDATE를 만드는 기능이다.

```
① 엔티티를 영속화(조회)할 때 그 시점의 값을 스냅샷으로 복사해 1차 캐시에 함께 보관
② 비즈니스 로직에서 엔티티 필드 변경 (setter, 도메인 메서드)
③ flush 시점: 1차 캐시의 모든 엔티티를 순회하며 현재 값 ↔ 스냅샷 비교
④ 달라진 엔티티마다 UPDATE 문 생성 → 쓰기 지연 저장소 → DB 전송
```

```java
// 안티패턴: 불필요한 save() 호출 — 영속 엔티티는 save() 없이도 갱신됨 (merge 호출로 SELECT만 추가될 수 있음)
@Transactional
public void changeName(Long id, String name) {
    Member member = memberRepository.findById(id).orElseThrow();
    member.changeName(name);
    memberRepository.save(member);        // 불필요
}

// 개선: 변경 감지에 맡김
@Transactional
public void changeName(Long id, String name) {
    Member member = memberRepository.findById(id).orElseThrow();
    member.changeName(name);              // 커밋 시 flush → UPDATE 자동 실행
}
```

- 기본적으로 **모든 컬럼을 포함한 UPDATE**가 나감 (SQL 재사용·캐싱을 위한 Hibernate의 전략). 컬럼이 매우 많다면 `@DynamicUpdate`로 변경된 컬럼만 포함할 수 있으나 매번 SQL을 생성해야 하므로 기본은 권장하지 않음
- 스냅샷을 유지하는 비용이 있으므로 조회 전용 트랜잭션은 `readOnly = true`로 스냅샷·비교를 생략함 (unit05 참고)

**쓰기 지연(Write-Behind)**: `persist()`·변경 감지로 만들어진 SQL은 즉시 DB로 가지 않고 **쓰기 지연 SQL 저장소**에 쌓였다가 flush 시 한 번에 전송된다. 덕분에 `hibernate.jdbc.batch_size` 설정으로 여러 INSERT를 JDBC 배치로 묶을 수 있다 (식별자 전략과의 관계는 unit08 참고).

> 💡 "변경 감지는 어떻게 구현되어 있는가"를 물으면 **스냅샷 비교**라고 답하고, "그래서 트랜잭션 안에서만 동작하며, 준영속 엔티티나 `readOnly` 트랜잭션에서는 UPDATE가 나가지 않는다"까지 연결하면 완성된 답변이 된다.

<br>

### 5. flush와 clear

**flush**는 영속성 컨텍스트의 변경 내용을 DB에 **동기화**(SQL 전송)하는 것이지 **커밋이 아니다**. flush 후에도 트랜잭션은 열려 있고 롤백이 가능하다.

| **flush 발생 시점**            | **설명**                                                                          |
| ------------------------------ | --------------------------------------------------------------------------------- |
| **트랜잭션 커밋 직전**         | `JpaTransactionManager`가 커밋 전에 자동 호출                                      |
| **JPQL / Criteria 쿼리 실행 전** | `FlushModeType.AUTO`(기본) — 쿼리 결과에 미반영 변경이 빠지지 않도록 먼저 flush     |
| **명시적 호출**                | `em.flush()`, `repository.flush()`, `saveAndFlush()`                               |

- `FlushModeType.COMMIT`으로 바꾸면 쿼리 실행 전 flush를 생략하지만, 쿼리 결과가 메모리의 변경을 반영하지 못하므로 특별한 이유가 없으면 AUTO를 유지함
- 네이티브 쿼리 실행 전에도 Hibernate는 기본적으로 flush를 수행함 (버전·설정에 따라 다를 수 있음)

**clear**는 영속성 컨텍스트를 **완전히 비워** 모든 엔티티를 준영속으로 만든다. 1차 캐시가 초기화되므로 이후 조회는 DB에서 새로 읽는다.

```java
@Transactional
public void bulkInsert(List<Member> members) {
    for (int i = 0; i < members.size(); i++) {
        em.persist(members.get(i));
        if (i % 1000 == 0) {
            em.flush();      // 쌓인 INSERT를 DB로 전송
            em.clear();      // 1차 캐시를 비워 메모리 증가 방지
        }
    }
}
```

> ⚠️ `clear()` 이후 이전에 조회해 둔 엔티티 변수를 계속 수정하면 준영속이라 **변경이 반영되지 않는다**. 벌크 연산(`@Modifying`) 뒤에 `clearAutomatically`로 컨텍스트를 비우는 이유와 그 함정은 unit08 참고.

<br>

### 6. 영속성 컨텍스트의 범위와 OSIV

스프링 부트의 `spring.jpa.open-in-view`(OSIV)는 **기본값이 `true`**로, 영속성 컨텍스트를 트랜잭션이 아니라 **HTTP 요청 전체**(인터셉터 → 뷰 렌더링 → 응답 완료)로 넓힌다. 기동 시 경고 로그가 출력된다.

| **항목**                  | **OSIV = true (기본)**                              | **OSIV = false**                                  |
| ------------------------- | --------------------------------------------------- | ------------------------------------------------- |
| **컨텍스트 범위**         | 요청 시작 ~ 응답 완료                                | 트랜잭션 시작 ~ 종료                               |
| **컨트롤러에서 지연 로딩**| **가능** (컨텍스트가 살아 있음)                      | `LazyInitializationException`                      |
| **DB 커넥션 점유**        | 트랜잭션 종료 후에도 지연 로딩 시 재획득, 요청 끝까지 유지 가능 | 트랜잭션 종료 시 즉시 반환                          |
| **적합한 환경**           | 트래픽 적은 관리자 화면, 빠른 개발                   | **트래픽 많은 API 서버** (커넥션 효율)             |

- OSIV를 끄면 지연 로딩·DTO 변환을 모두 트랜잭션 안(서비스 계층)에서 끝내야 함. 조회 전용 서비스(`@Transactional(readOnly = true)`)를 별도로 두는 구조가 흔함
- 커넥션 점유 관점의 상세 내용은 unit11 참고

<br>

### 7. 정리

- 영속성 컨텍스트는 **1차 캐시 + 스냅샷 + 쓰기 지연 저장소**로 구성되며, 스프링에서는 기본적으로 **트랜잭션과 생명주기가 같다**
- **1차 캐시**는 같은 트랜잭션 내 동일 식별자 조회를 SELECT 없이 반환하고 참조 동일성을 보장한다. JPQL은 캐시를 거치지 않고 DB에 질의한다
- **변경 감지**는 flush 시점에 스냅샷과 비교해 UPDATE를 만들며, 영속 상태 + 트랜잭션 안에서만 동작한다
- **flush**는 동기화이지 커밋이 아니며, 커밋 직전·JPQL 실행 전·명시 호출 시 발생한다. **clear**는 컨텍스트를 비워 모든 엔티티를 준영속으로 만든다
- `merge()`는 새 영속 객체를 반환하며 인자 객체는 준영속으로 남는다
- OSIV는 편의와 커넥션 효율의 트레이드오프이며, API 서버에서는 끄는 쪽을 검토한다
- 연관관계·지연 로딩·N+1은 **unit07**, 벌크 연산·식별자 전략은 **unit08** 참고
