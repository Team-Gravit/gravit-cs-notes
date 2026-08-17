## SQL 인젝션(SQL Injection)

**SQL 인젝션**은 사용자 입력값이 SQL 쿼리의 일부로 그대로 해석되어, 공격자가 데이터베이스를 임의로 조작할 수 있게 되는 취약점이다. OWASP A03(인젝션)에 속하며, 데이터 탈취·인증 우회·데이터 삭제로 이어지는 대표적이고 치명적인 웹 공격이다.

<br>

### 1. 원리

애플리케이션이 사용자 입력을 **문자열 연결**로 SQL 쿼리에 끼워 넣을 때 발생한다. 입력값이 데이터가 아니라 **SQL 구문**으로 해석되면, 공격자가 쿼리의 논리 자체를 바꿔버린다.

- 정상 의도: 입력값은 검색할 **데이터**로만 쓰이길 기대
- 공격: 입력값에 SQL 문법 기호(`'`, `--`, `OR` 등)를 섞어 **쿼리 구조를 변경**

<br>

### 2. 취약한 코드 예시

아래처럼 입력값을 문자열로 직접 이어 붙이면 위험하다.

```java
// 취약한 코드 — 입력값을 쿼리에 직접 연결
String id = request.getParameter("id");
String pw = request.getParameter("pw");
String sql = "SELECT * FROM users WHERE id = '" + id + "' AND pw = '" + pw + "'";
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery(sql);
```

```python
# 파이썬에서도 동일한 문제 — f-string으로 직접 연결
id = request.args.get("id")
sql = f"SELECT * FROM users WHERE id = '{id}'"
cursor.execute(sql)
```

<br>

### 3. 인증 우회 과정 (`' OR '1'='1`)

로그인 폼의 비밀번호 칸에 `' OR '1'='1` 을 입력하면 쿼리가 아래처럼 변형된다.

```
입력: pw = ' OR '1'='1

원래 쿼리:
  SELECT * FROM users WHERE id = 'admin' AND pw = '입력값'

변형된 쿼리:
  SELECT * FROM users WHERE id = 'admin' AND pw = '' OR '1'='1'
                                                        └──────┘
                                            '1'='1' 은 항상 참(TRUE)

결과: WHERE 조건 전체가 항상 참 → 비밀번호 없이 인증 통과
```

`'1'='1'`은 언제나 참이므로 `OR`로 연결된 조건 전체가 참이 되어, 비밀번호를 몰라도 로그인에 성공한다.

<br>

### 4. 공격 유형

| **유형**        | **설명**                                                     |
| --------------- | ------------------------------------------------------------ |
| **Error-based** | DB **에러 메시지**에 노출되는 정보를 이용해 구조를 파악        |
| **Union-based** | `UNION` 구문으로 **다른 테이블의 데이터**를 결과에 합쳐 추출   |
| **Blind**       | 응답에 데이터가 직접 안 보일 때 **참/거짓·시간 지연**으로 추론 |

<br>

### 5. 피해 사례와 영향

- 회원 전체의 **개인정보·비밀번호 탈취**
- 관리자 계정 **인증 우회** 후 시스템 장악
- `DROP TABLE` 등으로 **데이터 전체 삭제**

> ⚠️ SQL 인젝션은 데이터베이스에 직접 도달하는 공격이므로 피해 규모가 매우 크다. 국내외에서 대형 개인정보 유출 사고의 상당수가 이 취약점에서 비롯되었다.

<br>

### 6. 방어 — Prepared Statement (핵심)

가장 근본적인 방어는 **Prepared Statement(파라미터 바인딩)**다. 쿼리의 **구조를 먼저 확정**한 뒤 입력값을 **데이터로만** 채워 넣으므로, 입력값이 아무리 SQL 문법을 담고 있어도 구문으로 해석되지 않는다.

```java
// 안전한 코드 — 파라미터 바인딩 (? 자리표시자)
String sql = "SELECT * FROM users WHERE id = ? AND pw = ?";
PreparedStatement pstmt = conn.prepareStatement(sql);
pstmt.setString(1, id);   // 값은 데이터로만 취급됨
pstmt.setString(2, pw);
ResultSet rs = pstmt.executeQuery();
```

```python
# 파이썬 — 플레이스홀더 사용
sql = "SELECT * FROM users WHERE id = %s AND pw = %s"
cursor.execute(sql, (id, pw))   # 값은 데이터로만 전달됨
```

> 💡 Prepared Statement에서는 `' OR '1'='1` 을 넣어도 그 문자열 전체가 하나의 **아이디 값**으로 취급되어 조회에 실패할 뿐, 쿼리 구조는 바뀌지 않는다. 이것이 방어의 핵심 원리다.

<br>

### 7. 추가 방어 대책

- **입력값 검증**: 허용된 형식(길이·문자 집합)만 통과시킨다
- **최소 권한 원칙**: DB 계정에 꼭 필요한 권한만 부여해 피해 범위를 제한한다
- **에러 메시지 은닉**: 상세 DB 오류를 사용자에게 노출하지 않는다

> 💡 대부분의 **ORM**(예: JPA, SQLAlchemy)은 내부적으로 파라미터 바인딩을 사용하므로 기본적으로 SQL 인젝션에 안전하다. 단, ORM에서 직접 SQL 문자열을 조립하는 경우에는 여전히 취약할 수 있다.

❗️**시험 포인트**: SQL 인젝션과 그 방어책인 Prepared Statement는 정보처리기사·정보보안기사에서 매우 자주 출제된다.
