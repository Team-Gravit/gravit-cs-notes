## 식별 관계와 비식별 관계

### 1. 식별 관계(Identifying Relationship)

부모 테이블의 기본 키 또는 유니크 키를 자신의 기본 키로 사용하는 관계

<img src="https://raw.githubusercontent.com/Team-Gravit/gravit-images/main/database/unit02/image1.png" width="75%">

<br>
<br>

**특징**

- 부모 테이블의 PK가 **자식 테이블의 PK이자 FK 역할을 동시에 수행**
- 부모 데이터가 존재해야만 자식 데이터를 생성할 수 있음
- 부모와 자식의 **생명주기가 일치**
- ERD에서 **실선**으로 표현

<br>

### 2. 비식별 관계(Non-Identifying Relationship)

부모 테이블의 기본 키 또는 유니크 키를 자식 테이블의 외래 키로만 사용하고, 자식은 자체적인 키본 키를 갖는 관계

<img src="https://raw.githubusercontent.com/Team-Gravit/gravit-images/main/database/unit02/image2.png" width="75%">

<br>
<br>


**특징**

- 부모의 PK가 자식의 일반 속성(FK)으로만 사용
- 자식 데이터는 부모 데이터 없이도 **독립적으로 생성 가능**
- 부모와 자식의 **생명주기 분리**
- ERD에서 **점선**으로 표현

<br>

### 3. 선택 기준

**식별 관계를 선택하는 경우**

- 부모와 자식 엔터티가 강하게 결합되어 있을 때
- 부모 엔터티가 없이는 자식 엔터티가 의미가 없을 때

<br>

**비식별 관계를 선택하는 경우**

- 자식 엔터티가 독립적으로 존재할 수 있을 때
- 부모-자식 관계가 선택적일 때