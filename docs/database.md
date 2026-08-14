# database — 레퍼런스 모음

`database/unit01 ~ unit14` 개선 및 신규 unit 작성에 활용할 외부 자료 **69건** (1차 39 + 2차 30).

<br>

## 1. 기술 블로그 (24)

### 1-1. 데이터 모델링 · 정규화

| #   | 제목                                          | 링크                                                                         | 관련 UNIT      |
| --- | --------------------------------------------- | ------------------------------------------------------------------------------ | -------------- |
| 1   | 정규형 (1NF, 2NF, 3NF, BCNF) — Rebro          | https://rebro.kr/160                                                          | unit03         |
| 2   | 정규화(Normalization) 과정 (1NF~BCNF)         | https://hoyeonkim795.github.io/posts/normalization-course/                    | unit03         |
| 3   | 데이터베이스 정규화 이론 3 — 2NF·3NF·BCNF     | https://www.korecmblog.com/blog/database-normalization-3-2nf-3nf-bcnf          | unit03         |
| 4   | 정규화(1NF, 2NF, 3NF, BCNF, 4NF, 5NF)         | https://velog.io/@wisdom-one/정규화Normalization                               | unit03         |
| 5   | 정규화란? 예시로 보는 이상 현상 (코드 연구소) | https://code-lab1.com/정규화/                                                  | unit01, 03     |
| 6   | 정규화와 반정규화 (원자값·조인 성능 트레이드오프) | https://dongho.oopy.io/1ee5ca9b-265e-8052-9e0b-d2827e137c88                | unit03         |

### 1-2. 인덱스

| #   | 제목                                          | 링크                                                                       | 관련 UNIT   |
| --- | --------------------------------------------- | ---------------------------------------------------------------------------- | ----------- |
| 7   | 데이터베이스 인덱스 기초 개념 정리            | https://wkdtjsgur100.github.io/database-index/                              | 인덱스 unit |
| 8   | MySQL 인덱스 개념 및 B-Tree 인덱스            | https://velog.io/@hyj2508/MySQL-인덱스-개념-및-B-Tree-인덱스                 | 인덱스 unit |
| 9   | MySQL B-Tree 인덱스 구조와 인덱스 스캔        | https://velog.io/@semi-cloud/MySQL-B-Tree-인덱스-구조와-인덱스-스캔          | 인덱스 unit |
| 10  | MySQL B+Tree 구조를 통한 인덱스 연산 방식     | https://haon.blog/haon/index/concept/                                      | 인덱스 unit |
| 11  | MySQL·InnoDB의 인덱스 (클러스터링 인덱스)     | https://infoqoch.github.io/mysql/mysql-index.html                          | 인덱스 unit |
| 12  | DBMS의 인덱스 B-TREE (Real MySQL 정리, Medium) | https://swknight13.medium.com/dbms-의-인덱스-b-tree-4ff039dca22             | 인덱스 unit |

### 1-3. 트랜잭션 · 동시성 제어

| #   | 제목                                                  | 링크                                                                          | 관련 UNIT      |
| --- | ----------------------------------------------------- | ------------------------------------------------------------------------------- | -------------- |
| 13  | 트랜잭션의 격리 수준(Isolation Level)이란? (nesoy)    | https://nesoy.github.io/blog/Database-Transaction-isolation                    | 트랜잭션 unit  |
| 14  | 트랜잭션 격리 수준 (joont92)                          | https://joont92.github.io/db/트랜잭션-격리-수준-isolation-level/               | 트랜잭션 unit  |
| 15  | 엔진 수준의 Lock, MVCC, 트랜잭션 격리 수준            | https://www.nasa1515.com/database-cs-lock-mvcc-transaction-isolation-level/    | 트랜잭션 unit  |
| 16  | ACID란? (원자성·일관성·고립성·지속성)                  | https://velog.io/@bagt/Database-ACID란                                         | 트랜잭션 unit  |
| 17  | DB를 지탱하는 트랜잭션 (brunch)                       | https://brunch.co.kr/@skeks463/27                                              | 트랜잭션 unit  |
| 18  | **JPA Transactional 잘 알고 쓰고 계신가요?** (카카오페이) | https://tech.kakaopay.com/post/jpa-transactional-bri/                       | 트랜잭션 unit  |

### 1-4. SQL · 실행계획 · 분산

| #   | 제목                                        | 링크                                                       | 관련 UNIT        |
| --- | ------------------------------------------- | ------------------------------------------------------------ | ---------------- |
| 19  | SQL 성능분석하기 — 옵티마이저               | https://velog.io/@ehdcks3421/SQL-성능분석하기-옵티마이저     | SQL 튜닝 unit    |
| 20  | PostgreSQL 실행계획 분석하기 (서브쿼리)     | https://hyunwook.dev/230                                    | SQL 튜닝 unit    |
| 21  | 데이터베이스 파티셔닝과 샤딩 (hudi)         | https://hudi.blog/db-partitioning-and-sharding/             | unit15 후보      |
| 22  | 샤딩(Sharding)이란?                         | https://velog.io/@kyeun95/데이터베이스-샤딩Sharding이란      | unit15 후보      |
| 23  | **우아한형제들 기술블로그** (DB 카테고리)   | https://techblog.woowahan.com/                              | 실무 사례        |
| 24  | **토스 테크** (DB·인프라 사례)              | https://toss.tech/                                          | 실무 사례        |

> 💡 unit15 이후 신규 unit 후보: **파티셔닝/샤딩**, **레플리케이션과 복제 지연**, **NoSQL과 CAP 정리**.
> 세 주제 모두 국내 백엔드 면접 빈출이며 unit01~14의 선행 개념 위에서 설명 가능하다.

<br>

## 2. 서적 (5)

| #   | 도서명                          | 출판사/저자                | 링크                                                                 | 활용 포인트                          |
| --- | ------------------------------- | -------------------------- | ---------------------------------------------------------------------- | ------------------------------------ |
| 25  | **Real MySQL 8.0 (1권)**        | 위키북스 / 백은빈·이성욱   | https://www.aladin.co.kr/shop/wproduct.aspx?ItemId=278488709          | 아키텍처·트랜잭션·인덱스 정확한 근거 |
| 26  | Real MySQL 8.0 (2권)            | 위키북스 / 백은빈·이성욱   | https://ridibooks.com/books/1160000043                                 | 옵티마이저·실행계획·힌트             |
| 27  | Real MySQL 8.0 (1권) 전자책     | 교보 eBook                 | https://ebook-product.kyobobook.co.kr/dig/epd/ebook/4801158392704      | 목차 확인용                          |
| 28  | Real MySQL 8.0 독서 리뷰        | velog                      | https://velog.io/@han41562/Real-MySQL-8.0-책-회고리뷰                  | 챕터별 요점 파악                     |
| 29  | Real MySQL 8.0 (2권) 본문 미리보기 | Google Books            | https://books.google.com/books?id=OQ1sEAAAQBAJ                         | 특정 개념 원문 확인                  |

> ⚠️ Real MySQL은 입문서가 아니다. unit01~03(모델링·ERD·정규화)의 근거로는 부적합하며,
> **인덱스·트랜잭션·실행계획 unit의 정확성 검증용**으로 쓰는 것이 맞다.

<br>

## 3. 자격증 (10)

### 3-1. SQLD (SQL 개발자)

| #   | 자료                                        | 링크                                                                       | 관련 UNIT   |
| --- | ------------------------------------------- | ---------------------------------------------------------------------------- | ----------- |
| 30  | **한국데이터산업진흥원 데이터자격검정 (공식)** | https://www.dataq.or.kr/                                                  | 출제 범위   |
| 31  | SQLD 자격증 총정리 (Codeit)                 | https://www.codeit.kr/articles/everythingAboutSQLD                          | 시험 개요   |
| 32  | SQLD 요약정리                               | https://velog.io/@jiyean99/SQLD-요약정리                                    | 전 과목     |
| 33  | SQLD 개념 1과목·2과목 정리                  | https://velog.io/@sincerely/SQLD-개념-및-시험-정리                          | unit01~03   |
| 34  | SQLD 1과목 — 데이터 모델링의 이해           | https://velog.io/@juyeonma9/SQLD-1과목-데이터-모델링의-이해-개념-정리       | unit01, 02  |
| 35  | SQLD 핵심 요약 — 데이터 모델링의 이해       | https://thisiswoo.github.io/development/sqld-core-summary-part1.html        | unit01, 02  |
| 36  | SQLD 1과목 1장 데이터 모델링의 이해         | https://velog.io/@sy508011/SQLD-1과목-1장-데이터-모델링의-이해              | unit01      |

### 3-2. 정보처리기사

| #   | 자료                                             | 링크                                                    | 관련 UNIT     |
| --- | ------------------------------------------------ | --------------------------------------------------------- | ------------- |
| 37  | 정보처리기사 데이터베이스 빈출 개념 요약·기출 팁 | https://www.machuda.kr/blog/article/93                   | 전 unit       |
| 38  | 시나공 — 정보처리기사 필기 기출문제              | https://www.sinagong.co.kr/pds/001001001/past-exams      | 정규화·SQL 기출 |
| 39  | 전자문제집 CBT — 정보처리기사                    | https://www.comcbt.com/xe/iz                             | 반복 학습     |

> 💡 SQLD 1과목은 이 리포지토리의 **unit01(데이터 모델링) · unit02(E-R 다이어그램) · unit03(정규화)**와
> 범위가 거의 일치한다. 세 unit 개선 시 SQLD 요약 자료의 용어 정의를 우선 반영할 것.

<br>

---

<br>

# 2차 수집분 (30건)

1차가 인덱스·트랜잭션에 치우쳐 있어, **키/무결성 · SQL 문법 · DB 객체 · 커넥션 관리 · NoSQL**을 보강했다.

<br>

## 4. 키와 무결성 제약조건 (4)

| #   | 제목                                    | 링크                                                                                   | 관련 UNIT  |
| --- | --------------------------------------- | ---------------------------------------------------------------------------------------- | ---------- |
| 40  | 키(Key)의 개념 및 종류                  | https://velog.io/@letskuku/데이터베이스-키Key의-개념-및-종류                             | unit01, 02 |
| 41  | 키의 종류 — 최소성과 유일성             | https://velog.io/@00yubin00/DB-키의-종류-슈퍼키-후보키-기본키-대체키-외래키              | unit01, 02 |
| 42  | 키(Key) 개념과 종류 (Hoyeon)            | https://hoyeonkim795.github.io/posts/키-개념-및-종류/                                    | unit01, 02 |
| 43  | 데이터베이스 키 (IT 위키)               | https://itwiki.kr/w/데이터베이스_키                                                      | unit01, 02 |

> ⚠️ **슈퍼키 = 유일성만, 후보키 = 유일성 + 최소성**의 구분은 SQLD·정보처리기사에서 거의 매회
> 출제된다. unit01 또는 unit02에 비교 표를 반드시 넣을 것.

<br>

## 5. SQL 문법 — 명령어 분류와 JOIN (7)

| #   | 제목                                              | 링크                                                                             | 관련 UNIT  |
| --- | ------------------------------------------------- | ---------------------------------------------------------------------------------- | ---------- |
| 44  | SQL 종류 — DDL / DML / DCL (sangminlog)           | https://sangm1n.github.io/sql/                                                    | SQL unit   |
| 45  | DDL, DML, DCL, TCL, DQL 개념 및 정리              | https://devfoxstar.github.io/database/ddl-dml-dcl-tcl-dql/                        | SQL unit   |
| 46  | SQL 명령어 정리 + DDL과 DML의 트랜잭션 차이       | https://velog.io/@soo7132/DB-SQL-명령어-정리-DDL-DML-DQL-DCL-TCL                  | SQL unit   |
| 47  | SQL 기초 정리 — DML, DDL, TCL, DCL                | https://velog.io/@0nee/SQL-기초-정리-5-DML-DDL-TCL-DCL-정리                       | SQL unit   |
| 48  | **SQL 기본 문법: JOIN (한빛미디어 혼공)**         | https://hongong.hanbit.co.kr/sql-기본-문법-joininner-outer-cross-self-join/       | JOIN unit  |
| 49  | SQL의 꽃, JOIN 정복기 (INNER/OUTER/SELF/CROSS)    | https://velog.io/@mikio/SQL의-꽃-JOIN-정복하기-INNER-JOIN-OUTER-JOIN-SELF-JOIN     | JOIN unit  |
| 50  | [SQL] 조인 — Inner, Outer, Full, Cross, Natural, Self | https://iamjaeeuncho.github.io/study/SQL_Join/                                | JOIN unit  |

> 💡 SQL 실행 순서는 작성 순서와 다르다: **FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY**.
> SQLD·정보처리기사 모두에서 출제되므로 SQL unit에 도식으로 넣을 것.

<br>

## 6. DB 객체 — 뷰 · 프로시저 · 트리거 (3)

| #   | 제목                                   | 링크                                                       | 관련 UNIT   |
| --- | -------------------------------------- | ------------------------------------------------------------ | ----------- |
| 51  | 뷰(View)의 개념과 특징                 | https://velog.io/@chy0428/DB-뷰View의-개념과-특징           | DB 객체 unit |
| 52  | 뷰 & 프로시저 & 트리거의 간략한 개념   | https://blog.pages.kr/100                                   | DB 객체 unit |
| 53  | 뷰와 시스템 카탈로그 (diadia blog)     | https://jihunn-kim.github.io/database/database_system_8/    | DB 객체 unit |

<br>

## 7. 커넥션 풀 · 교착상태 (5)

| #   | 제목                                            | 링크                                                                    | 관련 UNIT       |
| --- | ----------------------------------------------- | ------------------------------------------------------------------------- | --------------- |
| 54  | **데이터베이스 커넥션 풀(DBCP)과 HikariCP** (hudi) | https://hudi.blog/dbcp-and-hikaricp/                                   | 커넥션 unit 후보 |
| 55  | 데이터베이스 — 교착상태(Deadlock)               | https://parkmuhyeun.github.io/etc/database/2022-07-03-Deadlock/          | 트랜잭션 unit   |
| 56  | HikariCP와 DBCP 최적화 고민하기 — 이론편        | https://haon.blog/database/hikaricp-theory/                             | 커넥션 unit 후보 |
| 57  | HikariCP 코드 분석하기 1편                      | https://hyunwook.dev/202                                                | 커넥션 unit 후보 |
| 58  | [DB] DB Connection Pool (아마란스 생각)         | https://amaran-th.github.io/데이터베이스/[DB]%20DB%20Connection%20Pool/  | 커넥션 unit 후보 |

> 💡 커넥션을 맺는 작업은 **TCP 3-way handshake를 수반하는 무거운 작업**이다. network 과목의
> TCP unit과 상호 참조하면 두 과목을 잇는 좋은 연결 고리가 된다.

<br>

## 8. NoSQL · 분산 (5)

| #   | 제목                                          | 링크                                                                        | 관련 UNIT   |
| --- | --------------------------------------------- | ----------------------------------------------------------------------------- | ----------- |
| 59  | **NoSQL에 대하여 (등장배경, CAP이론, 종류)**  | https://goodgid.github.io/NoSQL/                                             | unit16 후보 |
| 60  | RDBMS, NoSQL 차이를 간단하게 비교해보자       | https://renine94.github.io/util/데이터베이스-비교/                            | unit16 후보 |
| 61  | 채팅 시스템 NoSQL 특성 및 비교 분석 (CAP, PACELC) | https://velog.io/@murphytklee/채팅-시스템-NoSQL-특성-및-비교-분석-CAP-PACELC | unit16 후보 |
| 62  | 데이터베이스 시스템 이해하기: MySQL, MongoDB, PostgreSQL, Redis | https://velog.io/@sh93/데이터베이스-시스템-이해하기-MySQL-MongoDB-PostgreSQL-Redis | unit16 후보 |
| 63  | 레디스(Redis)란 무엇인가? (Medium)            | https://jyejye9201.medium.com/레디스-redis-란-무엇인가-2b7af75fa818            | unit16 후보 |

<br>

## 9. 자격증 추가 — SQLD 2과목 · 정보처리기사 실기 (6)

| #   | 자료                                       | 링크                                                                       | 관련 UNIT     |
| --- | ------------------------------------------ | ---------------------------------------------------------------------------- | ------------- |
| 64  | [SQLD 요약 정리] 2과목 1장 SQL 기본        | https://velog.io/@sy508011/SQLD-요약-정리-2과목-1장-SQL-기본                 | SQL unit      |
| 65  | [SQLD] 2과목 SQL 기본 및 활용 — 윈도우 함수 | https://velog.io/@zinu/SQLD-2과목-SQL-기본-및-활용-윈도우-함수              | SQL 활용 unit |
| 66  | [SQLD/개정] 2과목 1장, SQL 기본            | https://mwohye.github.io/sqld/SQLD-2/                                       | SQL unit      |
| 67  | SQLD 2과목 요약정리 (Yelm-blog)            | https://yelm-212.github.io/sqld/2/                                          | SQL unit      |
| 68  | [정보처리기사 실기] 7. SQL 응용 — 트랜잭션 | https://velog.io/@hammii/정보처리기사-실기-7.-SQL-응용-트랜잭션              | 트랜잭션 unit |
| 69  | [정보처리기사 실기] 데이터베이스 정리      | https://velog.io/@kya754/정보처리기사-실기-대비-데이터베이스                 | 전 unit       |

> ⚠️ SQLD 2과목(SQL 기본 및 활용)은 현재 리포지토리에서 **가장 취약한 영역**으로 보인다.
> unit04~14가 "SQL, 트랜잭션, 인덱스 등"으로 뭉뚱그려져 있어, 실제 커버 범위 점검이 먼저 필요하다.
