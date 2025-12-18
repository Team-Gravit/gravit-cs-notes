## 조인(JOIN)

**조인**은 두 개 이상의 테이블을 연결하여 데이터를 조회하는 SQL 기법이다. 관계형 데이터베이스에서 데이터를 여러 테이블에 나누어 저장하고, 필요 시 조인을 통해 결합하여 원하는 정보를 추출한다.

<br>

### 1. 조인의 필요성

**데이터 중복 최소화**

- 정규화를 통해 데이터를 여러 테이블로 분산 저장
- 중복 데이터를 제거하여 저장 공간 절약 및 일관성 유지

<br>

**유연한 데이터 조회**

- 필요한 데이터만 선택적으로 결합
- 다양한 조건으로 데이터 검색 가능

<br>

### 2-1. INNER JOIN

두 테이블에서 조인 조건을 만족하는 행만 반환한다. 양쪽 테이블 모두에 매칭되는 데이터가 있어야 결과에 포함된다.

<br>

**기본 구조**

```sql
SELECT A.column, B.column
FROM table_a A
INNER JOIN table_b B ON A.key = B.key;
```

<br>

**동작 방식**

```
table_a          table_b
id | name        id | value
1  | A           1  | 100
2  | B           3  | 300
3  | C

↓ INNER JOIN ON table_a.id = table_b.id

결과:
id | name | value
1  | A    | 100
3  | C    | 300
```

<br>

### 2-2. LEFT JOIN

왼쪽 테이블의 모든 행을 반환하고, 오른쪽 테이블에서 매칭되는 행을 결합한다. 오른쪽 테이블에 매칭되는 행이 없으면 NULL로 채운다.

<br>

**기본 구조**

```sql
SELECT A.column, B.column
FROM table_a A
LEFT JOIN table_b B ON A.key = B.key;
```

<br>

**동작 방식**

```
table_a          table_b
id | name        id | value
1  | A           1  | 100
2  | B           3  | 300
3  | C

↓ LEFT JOIN ON table_a.id = table_b.id

결과:
id | name | value
1  | A    | 100
2  | B    | NULL
3  | C    | 300
```

<br>

### 2-3. RIGHT JOIN

오른쪽 테이블의 모든 행을 반환하고, 왼쪽 테이블에서 매칭되는 행을 결합한다. 왼쪽 테이블에 매칭되는 행이 없으면 NULL로 채운다.

<br>

**기본 구조**

```sql
SELECT A.column, B.column
FROM table_a A
RIGHT JOIN table_b B ON A.key = B.key;
```

<br>

**동작 방식**

```
table_a          table_b
id | name        id | value
1  | A           1  | 100
2  | B           3  | 300
                 4  | 400

↓ RIGHT JOIN ON table_a.id = table_b.id

결과:
id   | name | value
1    | A    | 100
3    | NULL | 300
4    | NULL | 400
```

<br>

### 2-4. FULL OUTER JOIN

양쪽 테이블의 모든 행을 반환한다. 매칭되지 않는 행은 NULL로 채운다.

<br>

**기본 구조**

```sql
SELECT A.column, B.column
FROM table_a A
FULL OUTER JOIN table_b B ON A.key = B.key;
```

<br>

**동작 방식**

```
table_a          table_b
id | name        id | value
1  | A           1  | 100
2  | B           3  | 300

↓ FULL OUTER JOIN ON table_a.id = table_b.id

결과:
id | name | value
1  | A    | 100
2  | B    | NULL
3  | NULL | 300
```

<br>

**MySQL 구현**

MySQL은 FULL OUTER JOIN을 직접 지원하지 않으므로 UNION을 사용한다.

```sql
SELECT A.column, B.column
FROM table_a A
LEFT JOIN table_b B ON A.key = B.key
UNION
SELECT A.column, B.column
FROM table_a A
RIGHT JOIN table_b B ON A.key = B.key;
```

<br>

### 2-5. CROSS JOIN

두 테이블의 모든 행을 조합한 카테시안 곱을 반환한다. 조인 조건 없이 모든 경우의 수를 생성한다.

<br>

**기본 구조**

```sql
SELECT A.column, B.column
FROM table_a A
CROSS JOIN table_b B;
```

<br>

**동작 방식**

```
table_a          table_b
id | name        value
1  | A           100
2  | B           200

↓ CROSS JOIN

결과:
id | name | value
1  | A    | 100
1  | A    | 200
2  | B    | 100
2  | B    | 200
```

<br>

### 3. 다중 조인

3개 이상의 테이블을 연결하여 데이터를 조회한다. 조인은 순차적으로 실행되며, 이전 조인 결과에 다음 테이블을 결합한다.

<br>

**기본 구조**

```sql
SELECT A.column, B.column, C.column
FROM table_a A
LEFT JOIN table_b B ON A.key1 = B.key1
LEFT JOIN table_c C ON A.key2 = C.key2;
```

<br>

**실행 순서**

1. `table_a`와 `table_b`를 먼저 조인
2. 조인 결과에 `table_c`를 추가로 조인
3. 최종 결과 반환

<br>

**예시**

```sql
-- 사용자, 주문, 상품 정보 조회
SELECT u.user_id, u.name, o.order_id, o.order_date, p.product_name, p.price
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id
LEFT JOIN products p ON o.product_id = p.product_id
WHERE u.user_id = 1;
```

<br>

### 4. 조인 시 주의사항

**조인 순서**

- 조인 순서에 따라 성능이 달라질 수 있음
- 일반적으로 결과 행 수가 적은 테이블을 먼저 조인

<br>

**NULL 처리**

- LEFT/RIGHT JOIN 사용 시 NULL 값 처리 필요
- `COALESCE()`, `IFNULL()` 함수로 기본값 설정 가능

<br>

**중복 데이터**

- 1:N 관계 조인 시 중복 행 발생 가능
- `DISTINCT` 또는 `GROUP BY`로 중복 제거

<br>

### 5. 조인 성능 최적화

**인덱스 활용**

- 조인 키 컬럼에 인덱스 생성
- 조인 성능이 크게 향상됨

<br>

**필요한 컬럼만 선택**

- `SELECT *` 대신 필요한 컬럼만 명시
- 데이터 전송량 감소

<br>

**조인 조건 명확화**

- 명확한 조인 조건 사용
- 카테시안 곱 방지
