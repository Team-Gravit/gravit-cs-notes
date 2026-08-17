## 집계 함수와 GROUP BY·HAVING

여러 행을 하나의 값으로 요약하는 **집계 함수(Aggregate Function)**와, 행을 그룹으로 묶어 집계하는 **GROUP BY**, 그룹에 조건을 거는 **HAVING**을 정리한다. SELECT 문의 기본 문법은 unit07(DML)을 참고할 것.

<br>

### 1. 집계 함수 5종

집계 함수는 **여러 행의 값을 입력받아 단 하나의 값을 반환**하는 함수이다.

| **함수**    | **기능**       | **NULL 처리**        | **적용 가능 타입**   |
| ----------- | -------------- | -------------------- | -------------------- |
| **COUNT**   | 행의 개수      | COUNT(\*)만 NULL 포함 | 모든 타입            |
| **SUM**     | 합계           | **NULL 제외**        | 숫자                 |
| **AVG**     | 평균           | **NULL 제외**        | 숫자                 |
| **MAX**     | 최댓값         | **NULL 제외**        | 숫자, 문자, 날짜     |
| **MIN**     | 최솟값         | **NULL 제외**        | 숫자, 문자, 날짜     |

<br>

### 2. COUNT(*) vs COUNT(컬럼)

- **COUNT(\*)**: NULL 여부와 무관하게 **행 자체의 개수**를 셈
- **COUNT(컬럼)**: 해당 컬럼 값이 **NULL이 아닌 행**만 셈
- **COUNT(DISTINCT 컬럼)**: NULL을 제외하고 **중복을 제거한 값의 개수**를 셈

```sql
-- employees: 전체 5행, bonus 컬럼에 NULL 2개, 중복 값 1쌍 존재
SELECT COUNT(*)              FROM employees;  -- 5
SELECT COUNT(bonus)          FROM employees;  -- 3 (NULL 제외)
SELECT COUNT(DISTINCT bonus) FROM employees;  -- 2 (NULL·중복 제외)
```

> ⚠️ **AVG의 NULL 제외 함정**: `AVG(bonus)`는 "합계 ÷ 전체 행 수"가 아니라 "합계 ÷ NULL이 아닌 행 수"로 계산된다. NULL을 0으로 간주해 전체 평균을 구하려면 `AVG(COALESCE(bonus, 0))` 또는 `SUM(bonus) / COUNT(*)`를 사용해야 한다. SQLD와 면접에서 자주 출제되는 포인트다.

<br>

### 3. GROUP BY의 동작 원리

GROUP BY는 지정한 컬럼의 **값이 같은 행끼리 그룹으로 분할**한 뒤, 각 그룹마다 집계 함수를 한 번씩 적용해 **그룹당 한 행**을 반환한다.

```
employees                     dept_id로 그룹 분할            그룹별 집계
─────────────────            ──────────────────            ─────────────────
dept_id | salary             [ 그룹 10 ]                   dept_id | AVG
--------+-------             10 | 3000                     --------+------
  10    | 3000        →      10 | 5000              →        10    | 4000
  20    | 4000               [ 그룹 20 ]                     20    | 4500
  10    | 5000               20 | 4000
  20    | 5000               20 | 5000
```

<br>

```sql
SELECT dept_id, AVG(salary) AS avg_sal, COUNT(*) AS cnt
FROM   employees
GROUP  BY dept_id;
```

| **dept_id** | **avg_sal** | **cnt** |
| ----------- | ----------- | ------- |
| **10**      | 4000        | 2       |
| **20**      | 4500        | 2       |

<br>

### 3-1. GROUP BY 사용 시 SELECT 절 제약

- GROUP BY를 사용하면 SELECT 절에는 **GROUP BY에 명시한 컬럼**과 **집계 함수**만 올 수 있음
- 그룹당 한 행만 출력되는데, 그룹화되지 않은 컬럼은 그룹 안에 여러 값이 존재해 어떤 값을 보여줄지 정할 수 없기 때문

❗️**비집계 컬럼 주의**: `GROUP BY dept_id` 상태에서 `SELECT name`을 쓰면 표준 SQL에서는 오류가 발생한다. MySQL은 설정(`ONLY_FULL_GROUP_BY` 해제 시)에 따라 임의의 한 값을 반환해 버그의 원인이 되므로 주의해야 한다.

<br>

### 4. HAVING vs WHERE

둘 다 "조건 필터"지만 **적용 대상과 실행 시점**이 다르다. 실행 순서는 `FROM → WHERE → GROUP BY → HAVING → SELECT`이다(unit15 참고).

| **항목**          | **WHERE**                | **HAVING**                 |
| ----------------- | ------------------------ | -------------------------- |
| **필터링 대상**   | 개별 **행**              | **그룹**                   |
| **실행 시점**     | **GROUP BY 이전**        | **GROUP BY 이후**          |
| **집계 함수 사용** | 불가능                   | **가능**                   |
| **역할**          | 그룹화 전 행을 걸러냄    | 집계 결과로 그룹을 걸러냄  |

<br>

```sql
SELECT dept_id, AVG(salary) AS avg_sal
FROM   employees
WHERE  hire_date >= '2024-01-01'   -- 행 필터: 그룹화 전에 적용
GROUP  BY dept_id
HAVING AVG(salary) >= 4000;        -- 그룹 필터: 집계 후에 적용
```

> 💡 HAVING으로도 걸 수 있는 일반 조건(예: `dept_id = 10`)은 WHERE에 두는 것이 좋다. WHERE가 먼저 실행되어 그룹화 대상 행 수를 줄이므로 성능상 유리하다.

<br>

### 5. ROLLUP과 CUBE

GROUP BY의 확장 기능으로, 그룹별 집계에 **소계·총계 행을 추가**로 생성한다. SQLD 2과목에서 출제된다.

- **ROLLUP(A, B)**: (A, B) 조합별 집계 + **A별 소계** + **전체 총계**를 계층적으로 생성함
- **CUBE(A, B)**: ROLLUP의 결과에 더해 **B별 소계**까지, 가능한 **모든 조합**의 집계를 생성함
- **GROUPING SETS**: 원하는 그룹 조합만 직접 지정함

```sql
-- 부서별·직급별 급여 합계 + 부서별 소계 + 총계
SELECT dept_id, job, SUM(salary)
FROM   employees
GROUP  BY ROLLUP(dept_id, job);
```

- 소계·총계 행에서 그룹화되지 않은 컬럼은 **NULL**로 표시됨

<br>

### 6. 정리

- 집계 함수는 여러 행을 하나의 값으로 요약하며, **COUNT(\*)를 제외하면 모두 NULL을 제외**하고 계산함
- GROUP BY는 같은 값의 행을 그룹으로 분할해 **그룹당 한 행**을 반환함
- SELECT 절에는 그룹화 컬럼과 집계 함수만 사용할 수 있음
- **WHERE는 그룹화 전의 행**을, **HAVING은 그룹화 후의 그룹**을 필터링함
- ROLLUP·CUBE는 소계·총계가 필요한 보고서성 집계에 사용함
