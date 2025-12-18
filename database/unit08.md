## 서브쿼리 기초 (단일행/다중행, IN, EXISTS)

**서브쿼리**(Subquery)는 다른 쿼리 내부에 포함되어 있는 **SELECT 문**을 의미한다. 서브쿼리를 포함하는 쿼리를 **외부쿼리**(메인쿼리)라 하고, 서브쿼리를 **내부쿼리**라고 한다.

<br>

### 1. 서브쿼리의 특징

**실행 순서**

- 서브쿼리 실행 → 메인쿼리 실행

<br>

**표현 방식**

- 반드시 **괄호()** 로 감싸서 표현

<br>

**사용 가능 범위**

- 서브쿼리는 메인쿼리의 컬럼 사용 가능
- 메인쿼리는 서브쿼리의 컬럼 사용 불가

<br>

### 2. 서브쿼리의 장점

- 쿼리를 **구조화**하여 각 부분을 명확히 구분 가능
- 복잡한 **JOIN**이나 **UNION**을 대체하는 또 다른 방법 제공
- **가독성**이 좋음

<br>

### 3. 서브쿼리 실행 조건

- 서브쿼리는 **SELECT 문**으로만 작성 가능
- 반드시 **괄호()** 안에 존재해야 함
- 괄호 끝에 **세미콜론(;)**을 쓰지 않음
- **ORDER BY**를 사용할 수 없음

<br>

### 4. 서브쿼리 위치에 따른 분류

```sql
SELECT col1, (SELECT ...)    *-- 스칼라 서브쿼리*
FROM (SELECT ...)            *-- 인라인 뷰*
WHERE col = (SELECT ...)     *-- 중첩 서브쿼리*
```

**중첩 서브쿼리 (Nested Subquery)**

- **WHERE** 절에 사용되는 서브쿼리
- 조건 값을 서브쿼리로 지정

<br>

**인라인 뷰 (Inline View)**

- **FROM** 절에 사용되는 서브쿼리
- 하나의 테이블처럼 사용
- **반드시 별칭(alias)을 지정**해야 함

<br>

**스칼라 서브쿼리 (Scalar Subquery)**

- **SELECT** 절에 사용되는 서브쿼리
- 하나의 컬럼처럼 사용
- **하나의 레코드만 반환** 가능

<br>

### 5-1. 단일행 서브쿼리

서브쿼리의 결과가 **단 하나의 행**만 반환되는 경우이다. 비교 연산자(**=, >, <, >=, <=, !=**)를 사용한다.

```sql
*- 정대리의 직급을 구하시오*
SELECT office_worker
FROM employee
WHERE name = (SELECT name FROM employee WHERE name = '정대리');
```

<br>

### 5-2. 다중행 서브쿼리

서브쿼리의 결과가 **여러 개의 행**을 반환하는 경우이다. **IN, ANY, ALL, EXISTS** 등의 연산자를 사용한다.

```sql
*- 정대리보다 급여가 높은 사람들을 구하시오*
SELECT *
FROM employee
WHERE salary > ( SELECT salary FROM employee WHERE name = '정대리'
);
```

<br>

### 6. 다중행 연산자

**IN 연산자**

- 서브쿼리 결과 중 **하나라도 일치**하면 참

```sql
SELECT *
FROM employee
WHERE office_worker IN (
    SELECT office_worker
    FROM employee
    WHERE office_worker = '사원'
);
```

<br>

**ANY 연산자**

- 서브쿼리 결과 중 **하나라도 조건을 만족**하면 참
- **OR**의 의미

```sql
SELECT name, height
FROM userTbl
WHERE height = ANY(
    SELECT height
    FROM userTbl
    WHERE addr IN ('경남')
);
```

<br>

**ALL 연산자**

- 서브쿼리 결과의 **모든 값이 조건을 만족**해야 참
- **AND**의 의미

```sql
SELECT *
FROM city
WHERE population > ALL(
    SELECT population
    FROM city
    WHERE district = 'New York'
);
```

<br>

**EXISTS 연산자**

- 서브쿼리 결과가 **존재하는지** 여부를 확인
- 결과가 존재하면 참, 없으면 거짓

```sql
SELECT *
FROM employee e
WHERE EXISTS (
    SELECT 1
    FROM department d
    WHERE e.dept_id = d.id
);
```

<br>

### 7. 인라인 뷰 (Inline View)

FROM 절에 서브쿼리를 사용하는 경우로, **반드시 별칭(alias)을 지정**해야 한다.

```sql
*- 직급이 사원인 사람들의 이름과 급여를 구하시오*
SELECT EX1.name, EX1.salary
FROM ( SELECT * FROM employee WHERE office_worker = '사원'
) EX1;
```

<br>

### 8. 스칼라 서브쿼리 (Scalar Subquery)

SELECT 절에 사용되는 서브쿼리로, **하나의 값만 반환**해야 한다.

```sql
*- 정대리 급여와 전체 평균 급여를 구하시오*
SELECT name, salary, ( SELECT ROUND(AVG(salary), -1) FROM employee
) AS '평균급여'
FROM employee
WHERE name = '정대리';
```

<br>

**특징**

- 하나의 레코드만 반환 가능
- 두 개 이상의 레코드는 반환 불가
- 일치하는 데이터가 없으면 **NULL** 반환

<br>

### 9. DML과 서브쿼리

**INSERT 문**

다른 테이블의 데이터를 조회하여 삽입한다.

```sql
-- table2의 모든 데이터를 table1에 삽입
INSERT INTO table1 (SELECT * FROM table2);
```

<br>

**UPDATE 문**

서브쿼리로 조건을 지정하여 데이터를 수정한다.

```sql
-- 인턴의 급여를 10만원 인상
UPDATE employee
SET salary = (salary + 100000)
WHERE id = (
    SELECT id
    FROM employee
    WHERE office_worker = '인턴'
);
```

<br>

**DELETE 문**

서브쿼리로 조건을 지정하여 데이터를 삭제한다.

```sql
-- 인턴 직급 데이터 삭제
DELETE FROM employee
WHERE id = (
    SELECT id
    FROM employee
    WHERE office_worker = '인턴'
);
```
