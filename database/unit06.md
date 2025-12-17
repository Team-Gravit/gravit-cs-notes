## DDL(Data Definition Language)

데이터베이스의 **구조를 정의하는 언어**이다. **테이블**, **인덱스**, **스키마** 등 데이터베이스 객체를 생성, 수정, 삭제하는 역할을 한다.

| **명령어** | **기능** |
| --- | --- |
| **CREATE** | 데이터베이스, 테이블 등 객체 생성 |
| **ALTER** | 테이블 구조 수정 |
| **DROP** | 데이터베이스, 테이블 등 객체 삭제 |
| **TRUNCATE** | 테이블 데이터 전체 초기화 |

<br>

### 1. CREATE

데이터베이스 **객체를 생성**하는 명령어

<br>

**데이터베이스 생성**

```sql
CREATE DATABASE school;
```

<br>

**테이블 생성**

```sql
CREATE TABLE students (
    student_id INT PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE,
    department_id INT,
    created_at DATE DEFAULT CURRENT_DATE,
    FOREIGN KEY (department_id) REFERENCES departments(department_id)
);
```

<br>

### 2. ALTER

테이블의 **구조를 수정**하는 명령어

<br>

**컬럼 관련**

```sql
ALTER TABLE students ADD phone VARCHAR(20); -- 컬럼 추가

ALTER TABLE students MODIFY name VARCHAR(100); -- 컬럼 수정

ALTER TABLE students DROP COLUMN phone; -- 컬럼 삭제

ALTER TABLE students RENAME COLUMN name TO n_name; -- 컬럼명 수정
```

<br>

**제약 조건 관련**

```sql
ALTER TABLE students ADD CONSTRAINT uk_email UNIQUE (email); -- 제약 조건 추가

ALTER TABLE students DROP CONSTRAINT uk_email; -- 제약 조건 삭제
```

<br>

### 3. DROP

데이터베이스 **객체를 삭제**하는 명령어

<br>

**테이블 삭제**

```sql
DROP TABLE students
```

<br>

**데이터베이스 삭제**

```sql
DROP DATABASE school;
```

<br>

**존재 여부 확인 후 삭제**

```sql
DROP TABLE IF EXISTS students;
```

<br>

### 4. TRUNCATE

테이블의 **모든 데이터를 삭제**하고 초기 상태로 되돌리는 명령어

```sql
TRUNCATE TABLE students;
```