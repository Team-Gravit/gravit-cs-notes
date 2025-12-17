## 외래키와 제약 조건

**외래키**(FK, Foreign Key)는 다른 테이블의 **기본키 또는 고유키를 참조**하여 테이블 간의 관계를 정의하는 키이다. 이를 통해 **참조 무결성**을 보장한다.

<br>

### 1. 외래키(Foreign Key)

다른 테이블의 **기본키**(PK) 또는 **고유키**(Unique Key)**를 참조하는 속성** 또는 속성들의 집합. 테이블 간의 관계를 표현할 때 사용.

<img src="https://raw.githubusercontent.com/Team-Gravit/gravit-images/main/database/unit05/image1.png" width="100%">

<br>
<br>

**외래키의 특성**

- 참조되는 부모 테이블의 기본키 또는 고유키 값만 가질 수 있음
- NULL 값을 가질 수 있음
- 하나의 테이블에 여러 개의 외래키가 존재할 수 있음

<br>

### 2. 외래키 제약 조건

**데이터 삽입/수정 제한**

자식 테이블에 데이터를 삽입하거나 수정할 때, **외래키 값이 부모 테이블에 존재하지 않으면** 오류가 발생함.

<br>

**데이터 삭제 제한**

부모 테이블의 행을 삭제하려면, **해당 행을 참조하는 자식 테이블의 행이 없어야** 함

<br>

### 3. 참조 동작 옵션

부모 테이블의 데이터가 **삭제**, **수정**될 때, 자식 테이블의 데이터 처리 방식을 지정

<br>

**동작 옵션**

| **옵션** | **설명** |
| --- | --- |
| **CASCADE** | 부모 삭제, 수정 시 자식도 함께 삭제, 수정 |
| **SET NULL** | 부모 삭제, 수정 시 자식의 외래키를 NULL로 설정 |
| **SET DEFAULT** | 부모 삭제, 수정 시 자식의 외래키를 기본값으로 설정 |
| **RESTRICT** | 자식이 참조 중이면 부모 삭제, 수정 불가 |
| **NO ACTION** | **RESTRICT**과 동일 |

<br>

**SQL 예시**

```sql
CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    name VARCHAR(50),
    department_id INT,
    FOREIGN KEY (department_id) 
        REFERENCES departments(department_id)
        ON DELETE CASCADE
        ON UPDATE SET NULL
);
```

<br>

### 4. 일반 제약 조건

**NOT NULL 제약**

해당 속성에 **NULL 값을 허용하지 않음**. 반드시 값이 존재해야 함

```sql
CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    name VARCHAR(50) NOT NULL,      -- NULL 불가
    email VARCHAR(100)              -- NULL 허용
);
```

<br>

**UNIQUE 제약**

해당 속성의 값이 **고유(중복 불가능)해야 함을 보장**하는 제약 조건이다.

```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    email VARCHAR(100) UNIQUE,       -- 중복 불가, NULL 가능
    phone VARCHAR(20) UNIQUE         -- 중복 불가, NULL 가능
);
```

<br>

**CHECK 제약**

해당 속성의 값이 특정 조건을 만족해야 함.

```sql
CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price INT CHECK (price > 0),
);
```

<br>

**DEFAULT 제약**

해당 속성의 값이 지정되지 않았을 때, 기본값을 설정

```sql
CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    stock INT DEFAULT 0
);
```