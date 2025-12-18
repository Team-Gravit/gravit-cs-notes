## 인덱스(Index)

데이터베이스에서 **테이블의 특정 컬럼에 대해 빠른 검색을 가능하게 하는 추가 자료구조**. 인덱스를 통해 전체 테이블 스캔(Full Table Scan) 대신 일부 블록만 읽어 결과를 얻는다.

<br>

### 1. 인덱스의 역할

**검색 속도 향상**

- 전체 테이블 스캔 없이 **특정 데이터에 빠르게 접근**함
- 평균적으로 **O(log n)** 시간 복잡도로 데이터 탐색

<br>

**정렬과 범위 검색 지원**

- **ORDER BY**, **BETWEEN** 등의 연산에서 효율적으로 동작함
- 인덱스가 정렬된 상태로 유지되어 별도 정렬 불필요

<br>

**유니크 제약 조건 지원**

- **PRIMARY KEY**, **UNIQUE** 제약 조건 구현의 기반
- 중복 값 삽입 방지

<br>

### 2. 인덱스의 비용

**쓰기 성능 저하**

- **INSERT**, **UPDATE**, **DELETE** 시 인덱스도 함께 갱신해야 함
- 쓰기 작업마다 추가적인 디스크 I/O와 CPU 비용 발생

<br>

**스토리지 증가**

- 인덱스는 추가 디스크 공간을 차지함
- 넓은 커버링 인덱스는 용량을 급격히 증가시킴

<br>

**유지 관리 비용**

- 인덱스가 분산되거나 통계가 오래되면 쿼리 플래너가 비효율적 선택을 함
- 정기적인 **REINDEX**, **ANALYZE** 작업 필요

<br>

---

## B-Tree 인덱스

**균형 이진 탐색 트리** 구조를 기반으로 하는 인덱스로, 대부분의 RDBMS에서 **기본 인덱스 구조**로 사용한다.

<br>

### 1. B-Tree의 특징

**균형 트리 구조**

- 모든 리프 노드가 같은 깊이에 위치하는 균형 트리 구조 유지
- 데이터가 삽입, 삭제되더라도 항상 일정한 높이를 유지
- **검색**, **삽입**, **삭제** 연산이 일정한 시간 복잡도를 가짐

<br>

**디스크 접근 최적화**

- 노드가 여러 키와 자식을 한 블록에 저장하여 디스크에서 한 번에 많은 데이터를 읽어옴
- 디스크 I/O를 최소화하여 대용량 데이터 처리에 효과적

<br>

**효율적인 범위 검색**

- 정렬된 키 기반으로 검색이 이루어져 특정 범위 내의 값을 효율적으로 검색 가능
- **BETWEEN**, **>**, **<** 등의 연산에 적합

<br>

**높은 팬 아웃(Fan-out)**

- 각 노드가 많은 자식을 가질 수 있어 트리의 높이를 낮춤
- 트리의 높이가 낮을수록 검색 속도가 빨라짐

<br>

### 2. B+Tree의 특징

B-Tree의 변형으로, 데이터베이스 인덱스에서 가장 많이 사용되는 구조이다.

<br>

**모든 데이터는 리프 노드에만 저장**

- 내부 노드는 **탐색을 위한 키**만 포함
- 리프 노드에만 실제 데이터(또는 데이터 포인터)가 저장됨

<br>

**리프 노드 간의 연결**

- 리프 노드가 **링크드 리스트 형태**로 연결됨
- **순차적 접근**이 매우 빠르게 이루어짐
- 범위 쿼리나 정렬된 데이터 검색에 효율적

<br>

**더 큰 팬 아웃**

- 내부 노드에 데이터가 저장되지 않아 같은 크기의 노드에서 더 많은 자식 포인터를 가질 수 있음
- 트리의 **높이를 더 낮추는 효과**로 디스크 접근 횟수 감소

<br>

**균일한 데이터 접근 경로**

- 모든 데이터가 리프 노드에 있어 **항상 리프 노드까지 도달**해야 함
- 어떤 데이터를 찾더라도 **비슷한 경로와 디스크 접근**으로 **접근 시간이 일정**하게 유지

<br>

### 3. MySQL의 인덱스 구조

**클러스터드 인덱스(Clustered Index)**

- InnoDB 테이블에서 **PRIMARY KEY** 인덱스는 클러스터드 인덱스로 구현됨
- B+Tree의 리프 노드에 **실제 테이블의 데이터 행**이 저장됨
- 기본 키를 사용한 검색은 곧바로 리프 노드에서 데이터 행에 접근하여 매우 효율적
- 테이블에 기본 키가 없다면 InnoDB는 자체적으로 고유한 **Row ID**를 생성하여 클러스터드 인덱스로 사용

<br>

**보조 인덱스(Secondary Index)**

- 클러스터드 인덱스 외에 테이블의 다른 열에 대해 생성하는 인덱스
- B+Tree로 구현되며, 리프 노드에는 **기본 키 값**을 참조하는 방식으로 저장됨
- 보조 인덱스 사용 후 실제 데이터 행을 찾기 위해 기본 키를 이용해 클러스터드 인덱스에 다시 접근하는 과정(**백 트래킹, Back Tracking**)이 발생할 수 있음

<br>

---

## 복합 인덱스(Composite Index)

**여러 컬럼을 묶어 하나의 인덱스를 생성**하는 방식이다. 단일 컬럼 인덱스보다 더 정교한 검색 조건을 처리할 수 있으며, 올바른 컬럼 순서 설정이 성능에 큰 영향을 미친다.

<br>

### 1. 복합 인덱스 생성

```sql
-- 단일 인덱스
CREATE INDEX idx_country ON users(country);
CREATE INDEX idx_status ON users(status);

-- 복합 인덱스
CREATE INDEX idx_country_status ON users(country, status);
CREATE INDEX idx_country_status_created ON users(country, status, created_at);
```

<br>

### 2. 복합 인덱스의 구조

복합 인덱스는 **첫 번째 컬럼으로 정렬**하고, 같은 값 내에서 **두 번째 컬럼으로 정렬**하는 방식으로 구성된다.

```sql
-- 인덱스: (country, status, created_at)

인덱스 내부 구조:
+--------+--------+------------+
| country| status | created_at |
+--------+--------+------------+
| KR     | active | 2024-01-01 |
| KR     | active | 2024-01-15 |
| KR     | inactive| 2024-02-01 |
| US     | active | 2024-01-10 |
| US     | inactive| 2024-01-20 |
+--------+--------+------------+

1차 정렬: country
2차 정렬: status (country가 같을 때)
3차 정렬: created_at (country, status가 같을 때)
```

<br>

### 3. 왼쪽 접두사 규칙(Leftmost Prefix Rule)

복합 인덱스는 **왼쪽부터 순서대로** 인덱스를 활용한다. 첫 번째 컬럼을 사용하지 않으면 인덱스가 제대로 활용되지 않는다.

```sql
CREATE INDEX idx_abc ON users(country, status, created_at);

-- ✅ 인덱스 완전 활용
WHERE country = 'KR' AND status = 'active' AND created_at > '2024-01-01'

-- ✅ 인덱스 부분 활용 (country, status까지)
WHERE country = 'KR' AND status = 'active'

-- ✅ 인덱스 부분 활용 (country만)
WHERE country = 'KR'

-- ⚠️ 인덱스 부분 활용 (country만, status는 스킵)
WHERE country = 'KR' AND created_at > '2024-01-01'

-- ❌ 인덱스 미활용 (첫 컬럼 누락)
WHERE status = 'active'

-- ❌ 인덱스 미활용 (첫 컬럼 누락)
WHERE status = 'active' AND created_at > '2024-01-01'

-- ❌ 인덱스 미활용 (첫 컬럼 누락)
WHERE created_at > '2024-01-01'
```

<br>

### 4. 인덱스 매칭률

```
**인덱스 매칭률** = 질의문에서 equal(=) 조건으로 사용된 인덱스 컬럼 수 / 인덱스를 구성하는 전체 컬럼 수
```

**첫 컬럼부터 순서대로 연속되는 경우만** 매칭으로 인정된다.

```sql
CREATE INDEX idx_abc ON users(country, status, created_at);

-- 매칭률 3/3 (100%)
WHERE country = 'KR' AND status = 'active' AND created_at = '2024-01-01'

-- 매칭률 2/3 (67%)
WHERE country = 'KR' AND status = 'active'

-- 매칭률 1/3 (33%)
WHERE country = 'KR'

-- 매칭률 1/3 (33%) - status 스킵, created_at은 매칭 안됨
WHERE country = 'KR' AND created_at = '2024-01-01'

-- 매칭률 0/3 (0%) - 첫 컬럼 누락
WHERE status = 'active' AND created_at = '2024-01-01'
```

<br>

### 5. 범위 조건과 인덱스 활용

범위 조건(>, <, >=, <=, BETWEEN, LIKE)이 사용되면 **그 이후 컬럼은 인덱스로 활용되지 않는다**.

```sql
CREATE INDEX idx_abc ON users(country, status, created_at);

-- country: = (활용 O)
-- status: > (활용 O, 하지만 범위 조건)
-- created_at: 인덱스 활용 불가 (이전 컬럼이 범위 조건)
WHERE country = 'KR' AND status > 'active' AND created_at > '2024-01-01'

-- country: = (활용 O)
-- status: = (활용 O)
-- created_at: > (활용 O)
WHERE country = 'KR' AND status = 'active' AND created_at > '2024-01-01'
```

<br>

**예시 비교**

```sql
-- ✅ 좋은 예: 범위 조건을 마지막에
CREATE INDEX idx_status_created ON orders(status, created_at);
WHERE status = 'pending' AND created_at > '2024-01-01'
-- 인덱스 활용: status(=), created_at(>)

-- ❌ 나쁜 예: 범위 조건이 중간에
CREATE INDEX idx_created_status ON orders(created_at, status);
WHERE created_at > '2024-01-01' AND status = 'pending'
-- 인덱스 활용: created_at(>)만, status는 활용 불가
```

<br>

### 6. 선택도(Selectivity)

선택도는 **전체 데이터 중 특정 값이 차지하는 비율**을 의미한다. 선택도가 높을수록(중복 값이 적을수록) 필터링 효과가 좋다.

```sql
-- 선택도 계산
SELECT
    COUNT(DISTINCT country) / COUNT(*) AS selectivity_country,
    COUNT(DISTINCT status) / COUNT(*) AS selectivity_status,
    COUNT(DISTINCT user_id) / COUNT(*) AS selectivity_user_id
FROM users;

-- 결과 예시:
-- country: 0.001 (200개 국가 / 200,000 레코드) - 낮은 선택도
-- status: 0.0001 (2개 상태 / 200,000 레코드) - 매우 낮은 선택도
-- user_id: 1.0 (200,000 고유값 / 200,000 레코드) - 매우 높은 선택도
```

<br>

**선택도 기준**

- **1.0에 가까울수록**: 중복이 거의 없음 (높은 선택도) → 앞쪽 배치
- **0에 가까울수록**: 중복이 많음 (낮은 선택도) → 뒤쪽 배치

<br>

### 7. 복합 인덱스 설계 원칙

**원칙 1: 선택도가 높은 컬럼을 앞에 배치**

```sql
-- ❌ 나쁜 예: 선택도가 낮은 컬럼이 앞에
CREATE INDEX idx_status_userid ON users(status, user_id);
-- status는 'active', 'inactive' 두 값만 존재 (선택도 낮음)

-- ✅ 좋은 예: 선택도가 높은 컬럼이 앞에
CREATE INDEX idx_userid_status ON users(user_id, status);
-- user_id는 고유값 (선택도 높음)
```

<br>

**원칙 2: = 조건을 범위 조건보다 앞에 배치**

```sql
-- ❌ 나쁜 예
CREATE INDEX idx_created_status ON orders(created_at, status);
WHERE created_at > '2024-01-01' AND status = 'pending'
-- created_at이 범위 조건이라 status는 인덱스 활용 불가

-- ✅ 좋은 예
CREATE INDEX idx_status_created ON orders(status, created_at);
WHERE status = 'pending' AND created_at > '2024-01-01'
-- 둘 다 인덱스 활용 가능
```

<br>

**원칙 3: 자주 사용되는 쿼리 패턴을 우선 고려**

```sql
-- 쿼리 패턴 분석
-- 패턴 1 (80%): WHERE country = ? AND status = ?
-- 패턴 2 (15%): WHERE country = ?
-- 패턴 3 (5%): WHERE status = ?

-- ✅ 패턴 1, 2를 모두 커버
CREATE INDEX idx_country_status ON users(country, status);

-- ❌ 패턴 3만 커버 (빈도가 낮음)
CREATE INDEX idx_status ON users(status);
```

<br>

### 8. 복합 인덱스 vs 단일 인덱스

```sql
-- 테이블
CREATE TABLE orders (
    id INT,
    user_id INT,
    status VARCHAR(20),
    created_at DATETIME
);

-- 방법 1: 단일 인덱스 여러 개
CREATE INDEX idx_user ON orders(user_id);
CREATE INDEX idx_status ON orders(status);
-- 쿼리: WHERE user_id = 1 AND status = 'pending'
-- 문제: 옵티마이저가 둘 중 하나만 선택하거나 Index Merge 사용 (비효율)

-- 방법 2: 복합 인덱스
CREATE INDEX idx_user_status ON orders(user_id, status);
-- 쿼리: WHERE user_id = 1 AND status = 'pending'
-- 효율: 두 조건을 하나의 인덱스로 처리
```

<br>

---

## 커버링 인덱스(Covering Index)

쿼리에 필요한 **모든 컬럼을 인덱스에 포함**시켜 테이블 접근 없이 **인덱스만으로 응답 가능**하게 하는 기법이다. 실제 테이블 데이터를 읽지 않고 인덱스만 스캔하여 결과를 반환하므로 성능이 크게 향상된다.

<br>

### 1. 커버링 인덱스의 개념

**일반 인덱스 조회 과정**

```
1. 인덱스 스캔 → 조건에 맞는 레코드의 주소 찾기
2. 테이블 접근 → 실제 레코드 읽기 (Random I/O)
3. 필요한 컬럼 추출
```

<br>

**커버링 인덱스 조회 과정**

```
1. 인덱스 스캔 → 조건에 맞는 레코드 찾기
2. 인덱스에서 직접 데이터 추출 (테이블 접근 불필요)
```

<br>

### 2. 커버링 인덱스 생성

```sql
-- 일반 인덱스
CREATE INDEX idx_name ON users(name);

SELECT name, email, age FROM users WHERE name = 'Kim';
-- 동작: 인덱스에서 name 찾기 → 테이블에서 email, age 읽기

-- 커버링 인덱스
CREATE INDEX idx_name_email_age ON users(name, email, age);

SELECT name, email, age FROM users WHERE name = 'Kim';
-- 동작: 인덱스에서 name, email, age 모두 읽기 (테이블 접근 불필요)
```

<br>

### 3. 커버링 인덱스 적용 예시

**예시 1: 사용자 검색**

```sql
-- 자주 실행되는 쿼리
SELECT user_id, name, email
FROM users
WHERE status = 'active';

-- 일반 인덱스
CREATE INDEX idx_status ON users(status);
-- 동작: 인덱스에서 status 확인 → 테이블에서 user_id, name, email 읽기

-- 커버링 인덱스
CREATE INDEX idx_status_cover ON users(status, user_id, name, email);
-- 동작: 인덱스에서 모든 컬럼 읽기 (테이블 접근 불필요)
```

<br>

**예시 2: 주문 조회**

```sql
-- 자주 실행되는 쿼리
SELECT order_id, order_date, total_amount
FROM orders
WHERE user_id = 123 AND status = 'completed';

-- 커버링 인덱스
CREATE INDEX idx_orders_cover ON orders(
    user_id, status, order_id, order_date, total_amount
);
-- WHERE 조건: user_id, status
-- SELECT 컬럼: order_id, order_date, total_amount
-- → 모두 인덱스에 포함
```

<br>

**예시 3: 게시글 목록**

```sql
-- 자주 실행되는 쿼리
SELECT post_id, title, created_at
FROM posts
WHERE category_id = 5 AND is_published = 1
ORDER BY created_at DESC
LIMIT 10;

-- 커버링 인덱스
CREATE INDEX idx_posts_cover ON posts(
    category_id, is_published, created_at, post_id, title
);
-- WHERE: category_id, is_published
-- ORDER BY: created_at
-- SELECT: post_id, title, created_at
-- → 모두 인덱스에 포함
```

<br>

### 4-1. 커버링 인덱스의 장점

**디스크 I/O 감소**

테이블 레코드를 읽지 않고 인덱스만으로 데이터를 조회한다.

```sql
-- 일반 인덱스
CREATE INDEX idx_status ON orders(status);
SELECT order_id, amount FROM orders WHERE status = 'pending';
-- I/O: 인덱스 읽기 + 테이블 읽기 (Random I/O)

-- 커버링 인덱스
CREATE INDEX idx_status_cover ON orders(status, order_id, amount);
SELECT order_id, amount FROM orders WHERE status = 'pending';
-- I/O: 인덱스 읽기만 (Sequential I/O)
```

<br>

**인덱스 온리 스캔(Index Only Scan)**

```sql
-- 100만 건의 데이터에서 1,000건 조회

-- 일반 인덱스
-- 인덱스 스캔: 1,000번
-- 테이블 접근: 1,000번 (Random I/O)
-- 총 I/O: 2,000번

-- 커버링 인덱스
-- 인덱스 스캔: 1,000번
-- 테이블 접근: 0번
-- 총 I/O: 1,000번 (50% 감소)
```

<br>

**쿼리 성능 향상**

```sql
-- 성능 비교 예시
CREATE TABLE products (
    id INT PRIMARY KEY,
    category_id INT,
    name VARCHAR(100),
    price DECIMAL(10,2),
    stock INT
);

-- 1,000,000건 데이터

-- 일반 인덱스
CREATE INDEX idx_category ON products(category_id);
SELECT id, name, price FROM products WHERE category_id = 10;
-- 실행 시간: 250ms

-- 커버링 인덱스
CREATE INDEX idx_category_cover ON products(category_id, id, name, price);
SELECT id, name, price FROM products WHERE category_id = 10;
-- 실행 시간: 80ms (70% 개선)
```

<br>

### 4-2. 커버링 인덱스의 단점

**인덱스 크기 증가**

많은 컬럼을 포함하면 인덱스 크기가 커져 스토리지 비용이 증가한다.

```sql
-- 일반 인덱스
CREATE INDEX idx_status ON orders(status);
-- 인덱스 크기: 약 10MB

-- 커버링 인덱스
CREATE INDEX idx_status_cover ON orders(
    status, order_id, user_id, order_date, total_amount, shipping_address
);
-- 인덱스 크기: 약 80MB (8배 증가)
```

<br>

**메모리 부담**

인덱스가 메모리(버퍼 풀)에 올라가는데, 크기가 크면 메모리 효율이 떨어진다.

```sql
-- 서버 메모리: 16GB
-- 버퍼 풀: 10GB

-- 일반 인덱스 10개: 총 100MB → 버퍼 풀에 여유 있음
-- 커버링 인덱스 10개: 총 800MB → 버퍼 풀 압박
```

<br>

**쓰기 성능 저하**

인덱스에 포함된 컬럼이 많을수록 INSERT, UPDATE 시 부담이 증가한다.

```sql
-- 일반 인덱스
CREATE INDEX idx_status ON orders(status);

INSERT INTO orders (...) VALUES (...);
-- 인덱스 갱신: status 컬럼만

-- 커버링 인덱스
CREATE INDEX idx_status_cover ON orders(
    status, order_id, user_id, order_date, total_amount
);

INSERT INTO orders (...) VALUES (...);
-- 인덱스 갱신: 5개 컬럼 모두
-- 쓰기 성능 약 20~30% 저하 가능
```

<br>

**UPDATE 성능 영향**

```sql
-- 커버링 인덱스
CREATE INDEX idx_status_cover ON orders(status, order_id, total_amount);

-- status만 변경
UPDATE orders SET status = 'completed' WHERE order_id = 123;
-- 인덱스 갱신 필요: status 컬럼 포함

-- total_amount만 변경
UPDATE orders SET total_amount = 50000 WHERE order_id = 123;
-- 인덱스 갱신 필요: total_amount 컬럼 포함
-- → 인덱스에 포함된 컬럼을 변경하면 인덱스도 갱신됨
```

<br>

### 5. 커버링 인덱스 vs 일반 인덱스

| 구분            | 일반 인덱스 | 커버링 인덱스 |
| --------------- | ----------- | ------------- |
| **테이블 접근** | 필요        | 불필요        |
| **I/O**         | 많음        | 적음          |
| **조회 성능**   | 보통        | 빠름          |
| **인덱스 크기** | 작음        | 큼            |
| **쓰기 성능**   | 빠름        | 느림          |
| **메모리 사용** | 적음        | 많음          |

<br>

---

## 프리픽스 인덱스(Prefix Index)

문자열 컬럼의 **일부분(접두사)만 인덱싱**하는 기법이다. 긴 문자열 컬럼에서 전체를 인덱싱하는 대신 **앞부분 N글자만** 인덱스로 생성하여 공간과 성능을 최적화한다.

<br>

### 1. 프리픽스 인덱스 생성

```sql
-- 일반 인덱스
CREATE INDEX idx_email ON users(email);

-- 프리픽스 인덱스 (앞 10글자만 인덱싱)
CREATE INDEX idx_email_prefix ON users(email(10));
```

<br>

### 2. 프리픽스 인덱스의 동작 원리

**일반 인덱스**

```
email 컬럼 전체 값: 'hong.gildong@example.com'
인덱스 저장 값: 'hong.gildong@example.com' (전체)
```

<br>

**프리픽스 인덱스**

```
email 컬럼 전체 값: 'hong.gildong@example.com'
인덱스 저장 값: 'hong.gildo' (앞 10글자만)
```

<br>

### 3. 적절한 프리픽스 길이 결정

```sql
-- 유니크한 값의 개수 확인
SELECT COUNT(DISTINCT email) FROM users;
-- 결과: 10000

-- 프리픽스 길이별 유니크 값 개수 확인
SELECT
    COUNT(DISTINCT LEFT(email, 5)) AS prefix_5,
    COUNT(DISTINCT LEFT(email, 10)) AS prefix_10,
    COUNT(DISTINCT LEFT(email, 15)) AS prefix_15,
    COUNT(DISTINCT LEFT(email, 20)) AS prefix_20
FROM users;

-- 결과 예시:
-- prefix_5: 500   (5%)
-- prefix_10: 8000 (80%)
-- prefix_15: 9500 (95%)
-- prefix_20: 9900 (99%)
```

<br>

**선택도(Selectivity) 계산**

```sql
-- 선택도가 높을수록 좋은 인덱스
SELECT
    COUNT(DISTINCT LEFT(email, 10)) / COUNT(*) AS selectivity_10,
    COUNT(DISTINCT LEFT(email, 15)) / COUNT(*) AS selectivity_15
FROM users;

-- selectivity_10: 0.80 (80%)
-- selectivity_15: 0.95 (95%)
```

일반적으로 **80% 이상**의 선택도면 충분하다.

<br>

### 4-1. 프리픽스 인덱스의 장점

**인덱스 크기 감소**

전체 문자열이 아닌 앞부분만 인덱싱하여 공간을 절약한다.

```sql
-- 인덱스 크기 비교
-- 일반 인덱스: email VARCHAR(100) → 100바이트
-- 프리픽스 인덱스: email(10) → 10바이트 (90% 절약)
```

<br>

**쓰기 성능 향상**

인덱스 크기가 작아 INSERT, UPDATE 시 부담이 감소한다.

```sql
-- INSERT 시 인덱스 갱신 부담 감소
INSERT INTO users (email, name) VALUES ('user@example.com', '홍길동');
-- 일반 인덱스: 전체 18글자 처리
-- 프리픽스 인덱스: 앞 10글자만 처리
```

<br>

**메모리 효율성**

인덱스가 메모리에 올라갈 때 더 많은 엔트리를 캐시할 수 있다.

<br>

### 4-2. 프리픽스 인덱스의 단점

**정확도 저하**

접두사만으로는 완전히 구분되지 않을 수 있어 추가 필터링이 필요하다.

```sql
-- 프리픽스 인덱스 사용 시
SELECT * FROM users WHERE email = 'hong.gildong@example.com';

-- 내부 동작:
-- 1단계: 인덱스에서 'hong.gildo'로 시작하는 모든 행 찾기
--        'hong.gildong@example.com'
--        'hong.gildong@naver.com'
--        'hong.gildong123@gmail.com'
-- 2단계: 실제 테이블에서 정확한 값 필터링 (추가 작업)
```

<br>

**ORDER BY, GROUP BY 제한**

프리픽스 인덱스는 정렬 작업에 활용되지 못한다.

```sql
-- 프리픽스 인덱스가 있어도 filesort 발생
SELECT * FROM users
WHERE email LIKE 'hong%'
ORDER BY email;

-- EXPLAIN 결과: Using filesort
```

<br>

**LIKE 패턴 제한**

뒤쪽 패턴 매칭은 인덱스를 사용할 수 없다.

```sql
-- 인덱스 사용 가능
SELECT * FROM users WHERE email LIKE 'hong%';

-- 인덱스 사용 불가
SELECT * FROM users WHERE email LIKE '%@example.com';
SELECT * FROM users WHERE email LIKE '%hong%';
```

<br>

### 5. 사용 예시

**URL 컬럼 인덱싱**

```sql
-- URL은 프로토콜 부분이 거의 동일하므로 프리픽스 길이 조정 필요
CREATE TABLE pages (
    id INT PRIMARY KEY,
    url VARCHAR(2048)
);

-- 적절한 프리픽스 길이 찾기
SELECT
    COUNT(DISTINCT LEFT(url, 20)) AS prefix_20,
    COUNT(DISTINCT LEFT(url, 50)) AS prefix_50,
    COUNT(DISTINCT url) AS full_url
FROM pages;

-- 결과에 따라 인덱스 생성
CREATE INDEX idx_url_prefix ON pages(url(50));
```

<br>

**이메일 도메인 검색**

```sql
-- 이메일 앞부분 검색 (프리픽스 인덱스 효과적)
SELECT * FROM users
WHERE email LIKE 'hong%';

-- 이메일 도메인 검색 (프리픽스 인덱스 비효과적)
SELECT * FROM users
WHERE email LIKE '%@gmail.com';

-- 도메인 검색을 위한 별도 컬럼 권장
ALTER TABLE users ADD COLUMN email_domain VARCHAR(100);
UPDATE users SET email_domain = SUBSTRING_INDEX(email, '@', -1);
CREATE INDEX idx_email_domain ON users(email_domain);
```

<br>

### 6. 전체 인덱스 vs 프리픽스 인덱스

| 구분            | 전체 인덱스 | 프리픽스 인덱스 |
| --------------- | ----------- | --------------- |
| **저장 공간**   | 큼          | 작음            |
| **쓰기 성능**   | 느림        | 빠름            |
| **검색 정확도** | 높음        | 낮음            |
| **ORDER BY**    | 가능        | 불가능          |
| **정렬 활용**   | 가능        | 제한적          |
| **적용 대상**   | 짧은 문자열 | 긴 문자열       |
