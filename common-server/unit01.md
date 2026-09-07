## 트랜잭션과 격리 수준

**트랜잭션(Transaction)**은 "전부 성공하거나 전부 실패해야 하는" 작업의 논리적 단위이며, **격리 수준(Isolation Level)**은 동시에 실행되는 트랜잭션끼리 서로의 중간 상태를 얼마나 볼 수 있게 할지를 정하는 규칙이다. 백엔드 서버는 결국 DB 위에서 돈·재고·좌석 같은 상태를 바꾸는 프로그램이므로, 이 두 개념을 정확히 알아야 "가끔 값이 이상하게 꼬이는" 장애를 설계 단계에서 막을 수 있다.

<br>

### 1. 트랜잭션과 ACID

트랜잭션이 보장해야 하는 네 가지 성질을 **ACID**라고 부른다.

| **성질**                  | **의미**                                                     | **누가 보장하는가**                       |
| ------------------------- | ------------------------------------------------------------ | ----------------------------------------- |
| **원자성 (Atomicity)**    | 전부 반영되거나 전부 취소됨. 중간 상태로 끝나지 않음         | Undo 로그 기반 **롤백**                   |
| **일관성 (Consistency)**  | 트랜잭션 전후로 제약 조건(PK·FK·CHECK 등)이 항상 만족됨      | 제약 조건 + **애플리케이션 로직**         |
| **격리성 (Isolation)**    | 동시에 실행돼도 순차 실행한 것과 같은 결과가 나와야 함       | **격리 수준**·락·MVCC                     |
| **지속성 (Durability)**   | 커밋된 결과는 장애가 나도 사라지지 않음                      | Redo 로그(WAL)의 **디스크 동기화**        |

- 원자성과 지속성은 DBMS가 로그로 거의 완전하게 책임지지만, **격리성은 성능과 맞바꾸는 "조절 가능한" 성질**임
- 일관성은 DB 제약 조건만으로 충분하지 않으며, "잔액은 음수가 될 수 없다" 같은 업무 규칙은 결국 개발자가 지켜야 함

```sql
-- 계좌 이체: 두 UPDATE가 하나의 단위여야 한다
START TRANSACTION;
UPDATE account SET balance = balance - 10000 WHERE id = 1;
UPDATE account SET balance = balance + 10000 WHERE id = 2;
COMMIT;   -- 둘 중 하나라도 실패하면 ROLLBACK
```

> 💡 면접에서 "ACID를 설명해 보세요"는 정의 암기보다 **각 성질이 어떤 메커니즘으로 보장되는지**(롤백, 로그, 락)를 함께 말할 수 있는지를 본다.

<br>

### 2. 격리 수준과 이상 현상

격리성을 완벽히 지키려면 트랜잭션을 한 번에 하나씩 실행하면 되지만, 그러면 처리량이 극단적으로 떨어진다. 그래서 SQL 표준은 **"어느 정도의 이상 현상을 감수할지"**를 4단계로 나누어 선택하게 했다.

```
격리 수준 ↑ (엄격)                      격리 수준 ↓ (느슨)
SERIALIZABLE > REPEATABLE READ > READ COMMITTED > READ UNCOMMITTED
   정합성 ↑ / 동시성·성능 ↓                정합성 ↓ / 동시성·성능 ↑
```

<br>

### 2-1. 세 가지 대표 이상 현상

| **이상 현상**               | **설명**                                                     | **핵심 키워드**       |
| --------------------------- | ------------------------------------------------------------ | --------------------- |
| **Dirty Read**              | 다른 트랜잭션이 **커밋하지 않은 값**을 읽음. 롤백되면 없던 값이 됨 | 커밋 전 읽기          |
| **Non-Repeatable Read**     | 같은 행을 두 번 읽었는데 사이에 **UPDATE·커밋**되어 값이 달라짐 | 같은 행, 다른 값      |
| **Phantom Read**            | 같은 조건으로 두 번 조회했는데 사이에 **INSERT·커밋**되어 행이 늘어남 | 같은 조건, 다른 행 수 |

```
[Non-Repeatable Read 타임라인]
시간 →
TX A: ── SELECT balance → 100 ─────────────────── SELECT balance → 50  (같은 행인데 값이 다름)
TX B: ──────────────── UPDATE balance=50; COMMIT ──
```

<br>

### 2-2. 격리 수준 × 이상 현상 매트릭스

| **격리 수준**        | **Dirty Read** | **Non-Repeatable Read** | **Phantom Read** | **구현 방식(일반적)**                |
| -------------------- | -------------- | ----------------------- | ---------------- | ------------------------------------ |
| **READ UNCOMMITTED** | 발생           | 발생                    | 발생             | 사실상 읽기에 아무 제어 없음         |
| **READ COMMITTED**   | 차단           | 발생                    | 발생             | **문장(Statement) 단위** 스냅샷      |
| **REPEATABLE READ**  | 차단           | 차단                    | 발생 가능        | **트랜잭션 단위** 스냅샷             |
| **SERIALIZABLE**     | 차단           | 차단                    | 차단             | 범위 락 또는 직렬화 충돌 감지        |

- READ COMMITTED와 REPEATABLE READ의 차이는 결국 **"스냅샷을 언제 찍는가"**임. 전자는 SELECT 문장마다, 후자는 트랜잭션 시작(첫 읽기) 시점에 찍음
- 대부분의 현대 DBMS는 이 스냅샷을 **MVCC(다중 버전 동시성 제어)**로 구현해 읽기가 쓰기를 기다리지 않게 함

> ⚠️ 표는 SQL 표준 기준이다. 실제 DBMS는 표준보다 **더 엄격하게** 동작하는 경우가 많으므로(3절 참고), "표준에서는 ~, MySQL에서는 ~"처럼 구분해서 답해야 한다.

<br>

### 2-3. 표에 없는 이상 현상 — Lost Update

두 트랜잭션이 같은 값을 읽고 각자 계산해 덮어쓰면 **한쪽의 갱신이 사라진다**. 스냅샷 격리(REPEATABLE READ)만으로는 막지 못하는 경우가 많아 실무에서 가장 자주 만나는 문제다.

```
시간 →
TX A: ── SELECT stock → 10 ─── UPDATE stock = 9 (10-1) ── COMMIT
TX B: ──── SELECT stock → 10 ──────────── UPDATE stock = 9 (10-1) ── COMMIT
결과: 두 번 팔았는데 재고는 1만 줄어듦 → TX A의 갱신이 유실됨
```

```sql
-- 안티패턴: 애플리케이션에서 읽고 계산해서 덮어쓰기
SELECT stock FROM product WHERE id = 1;          -- 10
UPDATE product SET stock = 9 WHERE id = 1;       -- 다른 TX도 똑같이 9를 씀

-- 개선: DB가 현재 값을 기준으로 원자적으로 갱신
UPDATE product SET stock = stock - 1
WHERE  id = 1 AND stock >= 1;                    -- 갱신 행 수가 0이면 재고 부족
```

Lost Update를 락·버전으로 막는 방법은 **unit03(동시성 제어와 락)**에서 자세히 다룬다.

<br>

### 3. DBMS별 기본값과 실제 동작 차이

| **DBMS**             | **기본 격리 수준**    | **주목할 특징**                                                                 |
| -------------------- | --------------------- | ------------------------------------------------------------------------------- |
| **MySQL (InnoDB)**   | **REPEATABLE READ**   | 넥스트 키 락 + MVCC로 일반적인 Phantom Read를 대부분 차단                       |
| **PostgreSQL**       | **READ COMMITTED**    | READ UNCOMMITTED를 지정해도 READ COMMITTED로 동작. RR에서 Phantom Read도 차단 |
| **Oracle**           | **READ COMMITTED**    | REPEATABLE READ 미지원(READ COMMITTED·SERIALIZABLE만 제공)                      |
| **SQL Server**       | **READ COMMITTED**    | 기본은 락 기반이며, 옵션으로 스냅샷 방식 선택 가능                              |

- **같은 코드가 DBMS를 바꾸면 다르게 동작**할 수 있음. 예를 들어 MySQL에서 잘 되던 "한 트랜잭션 안에서 두 번 읽기" 로직이 PostgreSQL 기본값에서는 값이 바뀔 수 있음
- PostgreSQL의 REPEATABLE READ는 갱신 충돌 시 `could not serialize access` 오류를 던지므로 **재시도 로직**이 필요함
- 세부 옵션과 기본값은 **버전에 따라 다를 수 있으므로** 운영 DB의 설정을 직접 확인해야 함

```sql
-- 현재 세션의 격리 수준 확인 (MySQL 8.x)
SELECT @@transaction_isolation;

-- PostgreSQL
SHOW transaction_isolation;

-- 특정 트랜잭션만 격리 수준 변경
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

> 💡 "MySQL은 왜 기본값이 REPEATABLE READ인가?"라는 질문이 나오면, 과거 **문장 기반(Statement-based) 복제**에서 소스와 레플리카의 결과를 일치시키기 위한 역사적 이유가 크다고 답하면 된다. 현재는 행 기반 복제가 일반적이라 READ COMMITTED로 낮춰 운영하는 서비스도 많다.

<br>

### 4. 애플리케이션에서 트랜잭션 다루기

프레임워크는 보통 선언적 트랜잭션을 제공하며, 내부적으로는 **커넥션을 빌려 `autocommit`을 끄고, 메서드가 끝나면 커밋/롤백 후 반납**하는 프록시 구조다.

```java
@Transactional(isolation = Isolation.READ_COMMITTED, timeout = 5)
public void transfer(Long from, Long to, long amount) {
    accountRepository.withdraw(from, amount);   // UPDATE ... balance - amount
    accountRepository.deposit(to, amount);      // UPDATE ... balance + amount
    // 예외 없이 끝나면 커밋, RuntimeException이면 롤백
}
```

**흔한 함정**

- **트랜잭션 범위가 너무 김**: 외부 API 호출·파일 업로드를 트랜잭션 안에서 하면 커넥션과 락을 오래 점유해 전체 처리량이 떨어짐 (unit10 커넥션 풀 고갈 참고)
- **롤백 규칙 오해**: 많은 프레임워크가 기본적으로 **런타임 예외만 롤백**하고 체크 예외는 커밋함. 프레임워크·버전에 따라 다를 수 있으니 확인 필요
- **자기 호출(self-invocation)**: 같은 클래스 안에서 `this.method()`로 호출하면 프록시를 거치지 않아 트랜잭션이 적용되지 않음
- **읽기 전용 표시 누락**: 조회 트랜잭션에 `readOnly` 힌트를 주면 불필요한 변경 감지(Dirty Checking)와 플러시를 생략해 성능이 좋아지고, 복제 환경에서는 레플리카로 라우팅하는 기준이 됨 (unit07 참고)

> ⚠️ 격리 수준을 높이는 것은 공짜가 아니다. SERIALIZABLE은 대기·롤백이 급증하므로, 대부분의 서비스는 **READ COMMITTED 또는 REPEATABLE READ를 기본으로 두고, 정합성이 중요한 지점만 명시적 락이나 원자적 UPDATE로 보호**하는 방식을 택한다.

<br>

### 5. 격리 수준 선택 기준

| **상황**                                       | **권장 접근**                                          |
| ---------------------------------------------- | ------------------------------------------------------ |
| **일반 CRUD·조회 위주 서비스**                 | DBMS 기본값 유지 (READ COMMITTED / REPEATABLE READ)    |
| **한 트랜잭션에서 같은 데이터를 여러 번 읽음** | REPEATABLE READ 또는 명시적 락으로 일관성 확보         |
| **재고·잔액처럼 갱신 충돌이 치명적**           | 격리 수준보다 **원자적 UPDATE·비관적/낙관적 락**(unit03) |
| **정산·회계처럼 절대 어긋나면 안 되는 배치**   | SERIALIZABLE + 재시도, 단 동시성이 낮은 시간대에 실행  |
| **통계·대시보드처럼 약간의 오차 허용**         | 낮은 격리 수준 + 레플리카 조회                         |

<br>

### 6. 면접·실무 체크포인트

- **ACID** 각 성질이 무엇으로 보장되는지(롤백·제약 조건·격리 수준·WAL) 연결해 설명할 수 있는가
- **Dirty / Non-Repeatable / Phantom Read**를 타임라인으로 그리고, 어느 격리 수준부터 차단되는지 말할 수 있는가
- READ COMMITTED와 REPEATABLE READ의 차이가 **스냅샷 시점(문장 vs 트랜잭션)**이라는 것을 아는가
- **Lost Update**는 격리 수준 표에 없지만 실무에서 가장 흔하며, 원자적 UPDATE나 락으로 막아야 한다는 것을 아는가
- MySQL은 **REPEATABLE READ**, PostgreSQL·Oracle은 **READ COMMITTED**가 기본이고, 같은 코드가 DBMS마다 다르게 동작할 수 있음을 아는가
- 트랜잭션 안에서 외부 API 호출을 하면 안 되는 이유(커넥션·락 점유)를 설명할 수 있는가
