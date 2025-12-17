## DML(Data Manipulation Language)

데이터베이스에 **저장된 데이터를 조작**하는 언어이다. 테이블의 행을 조회, 삽입, 수정, 삭제하는 역할

| **명령어** | **기능** |
| --- | --- |
| SELECT | 데이터 조회 |
| INSERT | 데이터 삽입 |
| UPDATE | 데이터 수정 |
| DELETE | 데이터 삭제 |

<br>

### 1. INSERT

테이블에 새로운 행을 **삽입**하는 명령어

<br>

**단일행 삽입**

```sql
INSERT INTO students (student_id, name, email)
VALUES (1, '홍길동', 'hong@example.com');
```

<br>

**다중행 삽입**

```sql
INSERT INTO students (student_id, name, email)
VALUES 
    (2, '김철수', 'kim@example.com'),
    (3, '이영희', 'lee@example.com');
```

<br>

### 2. UPDATE

테이블의 기존 데이터를 **수정**하는 명령어

<br>

**특정 행 수정**

```sql
UPDATE students
SET email = 'newhong@example.com'
WHERE student_id = 1;
```

<br>

**다중 컬럼 수정**

```sql
UPDATE students
SET name = '홍길동수정', email = 'updated@example.com'
WHERE student_id = 1;
```

<br>

### 3. DELETE

테이블의 행을 **삭제**하는 명령어

<br>

**전체 행 삭제**

```sql
DELETE FROM students;
```

<br>

**특정 행 삭제**

```sql
DELETE FROM students WHERE student_id = 1;
```

<br>

### 4. SELECT

테이블의 데이터를 **조회**하는 명령어

<br>

**전체 컬럼 조회**

```sql
SELECT * FROM students;
```

<br>

**특정 컬럼 조회**

```sql
SELECT student_id FROM students;
```

<br>

### 5. WHERE 절

조건을 지정하여 특정 행만 조회, 수정, 삭제하는 데 사용

<br>

**비교 연산**

```sql
SELECT * FROM students WHERE student_id = 1;
SELECT * FROM students WHERE student_id > 5;
SELECT * FROM students WHERE student_id != 3;
```

<br>

**논리 연산**

```sql
SELECT * FROM students WHERE department_id = 1 AND name = '홍길동';
SELECT * FROM students WHERE department_id = 1 OR department_id = 2;
SELECT * FROM students WHERE NOT department_id = 1;
```

<br>

**범위 지정**

```sql
SELECT * FROM students WHERE student_id BETWEEN 1 AND 10;
```

<br>

**패턴 매칭**

```sql
-- '홍'으로 시작하는 이름
SELECT * FROM students WHERE name LIKE '홍%';

-- '동'으로 끝나는 이름
SELECT * FROM students WHERE name LIKE '%동';

-- '길'을 포함하는 이름
SELECT * FROM students WHERE name LIKE '%길%';
```

<br>

**목록 조건**

```sql
SELECT * FROM students WHERE department_id IN (1, 2, 3);
```

<br>

**NULL 조건**

```sql
SELECT * FROM students WHERE email IS NULL;
SELECT * FROM students WHERE email IS NOT NULL;
```