## SQL 개요와 명령어 분류

**SQL(Structured Query Language)**은 관계형 데이터베이스에서 데이터를 정의·조작·제어하기 위해 사용하는 표준 언어이다. 이 문서에서는 SQL의 특징과 명령어 4분류(DDL·DML·DCL·TCL), 그리고 SELECT 문의 논리적 실행 순서를 정리한다.

<br>

### 1. SQL의 정의와 특징

**선언형 언어(Declarative Language)**

- "어떻게(How)" 처리할지가 아니라 **"무엇을(What)" 원하는지**를 기술하는 언어임
- 데이터를 가져오는 구체적인 절차는 DBMS의 **옵티마이저(Optimizer)**가 결정함
- 절차형 언어(C, Java 등)와 달리 반복문·분기 없이 원하는 결과 집합만 선언함

<br>

**표준화된 언어**

- ANSI/ISO 표준으로 제정되어 대부분의 RDBMS(MySQL, PostgreSQL, Oracle 등)에서 공통 문법을 사용함
- 각 DBMS는 표준에 자체 확장 기능을 더해 제공함 (예: MySQL의 `LIMIT`, Oracle의 `ROWNUM`)

<br>

**집합 단위 처리**

- SQL은 행(Row) 하나가 아닌 **집합(Set) 단위**로 데이터를 처리함
- `WHERE` 조건을 만족하는 모든 행이 한 번의 명령으로 처리됨

<br>

### 2. 명령어 4분류

SQL 명령어는 기능에 따라 **DDL, DML, DCL, TCL** 네 가지로 분류한다.

| **분류** | **의미**                            | **대표 명령어**                        | **대상**           | **자동 커밋** |
| -------- | ----------------------------------- | -------------------------------------- | ------------------ | ------------- |
| **DDL**  | 데이터 정의어 (Data Definition)     | CREATE, ALTER, DROP, TRUNCATE          | 테이블 등 객체 구조 | **O**         |
| **DML**  | 데이터 조작어 (Data Manipulation)   | SELECT, INSERT, UPDATE, DELETE         | 테이블의 행(데이터) | **X**         |
| **DCL**  | 데이터 제어어 (Data Control)        | GRANT, REVOKE                          | 사용자 권한        | **O**         |
| **TCL**  | 트랜잭션 제어어 (Transaction Control) | COMMIT, ROLLBACK, SAVEPOINT          | 트랜잭션           | -             |

> 💡 SELECT만 따로 **DQL(Data Query Language)**로 분류하기도 한다. SQLD·정보처리기사에서는 SELECT를 DML에 포함하는 4분류가 기본이다.

<br>

### 2-1. DDL — 데이터 정의어

- 테이블, 인덱스, 뷰 등 **데이터베이스 객체의 구조**를 생성·변경·삭제함
- `CREATE`(생성), `ALTER`(변경), `DROP`(삭제), `TRUNCATE`(전체 데이터 삭제)
- 실행 즉시 **자동 커밋(Auto Commit)**되어 ROLLBACK으로 되돌릴 수 없음

❗️**DDL은 롤백 불가**: DROP이나 TRUNCATE를 실행하면 트랜잭션과 무관하게 즉시 확정된다. 운영 환경에서는 실행 전 반드시 백업 여부를 확인해야 한다.

<br>

### 2-2. DML — 데이터 조작어

- 테이블에 **저장된 데이터(행)**를 조회·삽입·수정·삭제함
- `SELECT`(조회), `INSERT`(삽입), `UPDATE`(수정), `DELETE`(삭제)
- 자동 커밋되지 않으며, COMMIT 전까지 ROLLBACK으로 되돌릴 수 있음

<br>

### 2-3. DCL — 데이터 제어어

- 사용자에게 데이터베이스 객체에 대한 **권한을 부여하거나 회수**함

```sql
-- 사용자에게 students 테이블 조회 권한 부여
GRANT SELECT ON students TO user1;

-- 부여한 권한 회수
REVOKE SELECT ON students FROM user1;
```

<br>

### 2-4. TCL — 트랜잭션 제어어

- DML 작업의 결과를 **트랜잭션 단위로 확정하거나 취소**함
- `COMMIT`(확정), `ROLLBACK`(취소), `SAVEPOINT`(부분 롤백 지점 지정)

```sql
UPDATE accounts SET balance = balance - 10000 WHERE id = 1;
SAVEPOINT sp1;
UPDATE accounts SET balance = balance + 10000 WHERE id = 2;
ROLLBACK TO SAVEPOINT sp1;  -- sp1 이후 작업만 취소
COMMIT;                      -- sp1까지의 작업 확정
```

<br>

### 3. DELETE vs TRUNCATE vs DROP

세 명령어 모두 "삭제"와 관련되지만 분류·동작·복구 가능성이 전혀 다르다.

| **항목**        | **DELETE**             | **TRUNCATE**           | **DROP**               |
| --------------- | ---------------------- | ---------------------- | ---------------------- |
| **분류**        | DML                    | DDL                    | DDL                    |
| **삭제 대상**   | 행 (조건 지정 가능)    | 모든 행                | **테이블 구조 자체**   |
| **WHERE 절**    | 사용 가능              | 불가능                 | 불가능                 |
| **ROLLBACK**    | **가능** (커밋 전)     | 불가능 (자동 커밋)     | 불가능 (자동 커밋)     |
| **속도**        | 느림 (행 단위 기록)    | 빠름                   | 빠름                   |
| **저장 공간**   | 유지                   | 초기화                 | 반환                   |

> ⚠️ DELETE는 행마다 로그를 남기며 삭제하므로 느리지만 롤백이 가능하고, TRUNCATE는 테이블을 초기 상태로 재생성하는 방식이라 빠르지만 롤백이 불가능하다. 이 차이는 정보처리기사·SQLD·면접에서 모두 빈출이다.

<br>

### 4. SELECT 문의 논리적 실행 순서

SQL은 **작성 순서와 실행 순서가 다르다**. DBMS는 아래 순서로 쿼리를 논리적으로 처리한다.

```
작성 순서                       논리적 실행 순서
─────────                      ─────────────────
SELECT   컬럼        ①  FROM      대상 테이블 확정
FROM     테이블  →   ②  WHERE     행 필터링
WHERE    조건        ③  GROUP BY  그룹화
GROUP BY 컬럼        ④  HAVING    그룹 필터링
HAVING   조건        ⑤  SELECT    컬럼 선택·별칭 부여
ORDER BY 컬럼        ⑥  ORDER BY  정렬
LIMIT    개수        ⑦  LIMIT     행 개수 제한
```

<br>

```sql
SELECT dept_id, AVG(salary) AS avg_sal   -- ⑤ 컬럼 선택
FROM   employees                         -- ① 테이블 확정
WHERE  hire_date >= '2024-01-01'         -- ② 행 필터링
GROUP  BY dept_id                        -- ③ 그룹화
HAVING AVG(salary) >= 4000               -- ④ 그룹 필터링
ORDER  BY avg_sal DESC                   -- ⑥ 정렬
LIMIT  5;                                -- ⑦ 상위 5행
```

> 💡 WHERE 절에서 SELECT 절의 별칭(`avg_sal`)을 쓸 수 없는 이유가 바로 이 실행 순서 때문이다. WHERE(②)가 SELECT(⑤)보다 먼저 실행되므로 그 시점에는 별칭이 존재하지 않는다. 반면 ORDER BY(⑥)는 SELECT 이후에 실행되므로 별칭 사용이 가능하다.

<br>

### 5. 정리

- SQL은 **무엇을 원하는지 선언**하면 옵티마이저가 처리 방법을 결정하는 선언형 언어임
- 명령어는 **DDL(구조) · DML(데이터) · DCL(권한) · TCL(트랜잭션)** 4가지로 분류함
- DDL·DCL은 자동 커밋되지만 DML은 TCL(COMMIT/ROLLBACK)로 확정·취소함
- SELECT 문은 **FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT** 순으로 실행됨
- DDL의 상세 문법은 **unit06(DDL)**, DML의 상세 문법은 **unit07(DML)**을 참고할 것
