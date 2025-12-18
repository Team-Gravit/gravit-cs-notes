## 뷰(View)

**뷰**는 하나 이상의 테이블에서 유도된 **가상 테이블**이다. 실제 데이터를 저장하지 않고 **SELECT 문만 저장**하여 필요할 때마다 실행한다.

<br>

### 1. 뷰의 특징

**가상 테이블**

- 물리적으로 데이터를 저장하지 않음
- SELECT 쿼리만 저장

<br>

**동적 실행**

- 뷰 조회 시마다 기반 쿼리 실행
- 항상 최신 데이터 반영

<br>

**테이블처럼 사용**

- SELECT, INSERT, UPDATE, DELETE 가능 (제약 있음)
- 다른 뷰나 쿼리에서 참조 가능

<br>

### 2. 뷰 생성

**기본 구조**

```sql
CREATE VIEW view_name AS
SELECT column1, column2
FROM table
WHERE 조건;
```

<br>

**예시**

```sql
-- 활성 고객 뷰
CREATE VIEW active_customers AS
SELECT id, name, email
FROM customers
WHERE status = 'active';

-- 뷰 사용
SELECT * FROM active_customers;
```

<br>

**복잡한 뷰**

```sql
-- 주문 통계 뷰
CREATE VIEW order_statistics AS
SELECT
    c.id AS customer_id,
    c.name,
    COUNT(o.id) AS order_count,
    SUM(o.total_price) AS total_spent
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name;
```

<br>

### 3. 뷰 수정 및 삭제

**뷰 수정**

```sql
-- 기존 뷰 대체
CREATE OR REPLACE VIEW active_customers AS
SELECT id, name, email, phone
FROM customers
WHERE status = 'active';
```

<br>

**뷰 삭제**

```sql
DROP VIEW active_customers;

-- 존재하는 경우만 삭제
DROP VIEW IF EXISTS active_customers;
```

<br>

### 4. 뷰의 장점

**보안성**

- 특정 컬럼만 노출하여 민감 정보 보호
- 사용자별로 다른 뷰 제공 가능

```sql
-- 급여 정보 제외한 직원 뷰
CREATE VIEW employee_public AS
SELECT id, name, department, position
FROM employees;
-- salary 컬럼은 제외됨
```

<br>

**단순화**

- 복잡한 조인/집계 쿼리를 단순하게 사용
- 코드 재사용성 향상

```sql
-- 복잡한 쿼리를 뷰로 저장
CREATE VIEW monthly_sales AS
SELECT
    DATE_FORMAT(order_date, '%Y-%m') AS month,
    SUM(total_price) AS total_sales
FROM orders
GROUP BY DATE_FORMAT(order_date, '%Y-%m');

-- 간단하게 사용
SELECT * FROM monthly_sales WHERE month = '2024-01';
```

<br>

**독립성**

- 테이블 구조 변경 시 뷰만 수정
- 애플리케이션 코드는 변경 불필요

<br>

**논리적 데이터 구조**

- 물리적 테이블 구조와 독립적인 논리적 구조 제공
- 비즈니스 로직에 맞는 데이터 표현

<br>

### 5. 뷰의 단점

**성능 문제**

- 뷰 조회마다 기반 쿼리 실행
- 복잡한 뷰는 성능 저하 가능
- 인덱스 직접 생성 불가

<br>

**수정 제한**

- 복잡한 뷰는 INSERT/UPDATE/DELETE 불가
- 집계 함수, GROUP BY, DISTINCT 사용 시 수정 불가

```sql
-- 수정 불가능한 뷰
CREATE VIEW customer_stats AS
SELECT
    customer_id,
    COUNT(*) AS order_count
FROM orders
GROUP BY customer_id;

-- 오류 발생
INSERT INTO customer_stats VALUES (101, 5);
```

<br>

**종속성**

- 기반 테이블 구조 변경 시 뷰도 영향받음
- 테이블 삭제 시 뷰도 사용 불가

<br>

### 6. 수정 가능한 뷰 조건

**단순 뷰만 수정 가능**

- 단일 테이블 기반
- 집계 함수 없음
- GROUP BY, DISTINCT 없음
- UNION 없음

```sql
-- 수정 가능한 뷰
CREATE VIEW simple_customers AS
SELECT id, name, email
FROM customers
WHERE status = 'active';

-- 수정 가능
UPDATE simple_customers
SET email = 'new@email.com'
WHERE id = 1;
```

<br>

### 7. WITH CHECK OPTION

뷰를 통한 데이터 수정 시 **뷰의 조건을 벗어나는 수정을 방지**한다.

<br>

**기본 동작**

```sql
CREATE VIEW active_customers AS
SELECT * FROM customers
WHERE status = 'active';

-- 문제: 조건 벗어나는 수정 가능
UPDATE active_customers
SET status = 'inactive'
WHERE id = 1;
-- 뷰 조건에 맞지 않지만 수정됨
```

<br>

**WITH CHECK OPTION 사용**

```sql
CREATE VIEW active_customers AS
SELECT * FROM customers
WHERE status = 'active'
WITH CHECK OPTION;

-- 오류 발생: 조건 위반
UPDATE active_customers
SET status = 'inactive'
WHERE id = 1;
-- CHECK OPTION failed
```
