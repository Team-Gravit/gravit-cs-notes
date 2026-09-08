# CLAUDE.md

This file provides guidance to Claude Code when working in this repository.
Read this file completely before creating, editing, or reviewing any unit file.

---

## 1. Repository Overview

Team-Gravit CS educational notes repository. All content is written in **Korean**
(variable names and language keywords in code blocks are the only exception).
Pure documentation repo — no build tools, linters, or test frameworks.

**Subjects and their current unit status:**

CS 기초 6과목:

| Subject             | Current Last Unit | Next Unit to Create |
| ------------------- | ----------------- | ------------------- |
| `algorithm/`        | unit23            | unit24              |
| `data-structure/`   | unit12            | unit13              |
| `database/`         | unit21            | unit22              |
| `network/`          | unit22            | unit23              |
| `operating-system/` | unit14            | unit15              |
| `web-security/`     | unit13            | unit14              |

개편안 신규 챕터 17과목 (2026-09-08 추가, 챕터명 `카테고리 · 챕터` → 디렉터리 `<category>-<chapter>`):

| Subject             | Chapter               | Current Last Unit | Next Unit to Create |
| ------------------- | --------------------- | ----------------- | ------------------- |
| `common-server/`    | Common · Server       | unit12            | unit13              |
| `common-web/`       | Common · Web          | unit10            | unit11              |
| `common-aos/`       | Common · AOS          | unit10            | unit11              |
| `common-ios/`       | Common · iOS          | unit10            | unit11              |
| `be-spring/`        | BE · Spring           | unit11            | unit12              |
| `be-nodejs/`        | BE · Node.js          | unit09            | unit10              |
| `be-django/`        | BE · Django           | unit09            | unit10              |
| `fe-react/`         | FE · React            | unit10            | unit11              |
| `fe-vue/`           | FE · Vue.js           | unit07            | unit08              |
| `fe-nextjs/`        | FE · Next.js          | unit09            | unit10              |
| `mobile-android/`   | Mobile · Android      | unit10            | unit11              |
| `mobile-ios/`       | Mobile · iOS          | unit09            | unit10              |
| `lang-java/`        | Language · Java       | unit10            | unit11              |
| `lang-kotlin/`      | Language · Kotlin     | unit09            | unit10              |
| `lang-typescript/`  | Language · TypeScript | unit07            | unit08              |
| `lang-python/`      | Language · Python     | unit08            | unit09              |
| `lang-swift/`       | Language · Swift      | unit08            | unit09              |

> ⚠️ 기존 유닛의 번호 변경·통합·이동은 금지한다. 문제 플랫폼이 챕터-유닛-레슨 구조로
> 유닛 번호에 매핑되어 있으므로, 새로운 개념은 항상 각 과목의 마지막 유닛 번호
> 다음부터 **추가**하는 방식으로만 확장한다. (`docs/curriculum.md` 적용 방침 참고)

---

## 2. File Naming Rules

- Pattern: `unit{NN}.md` — always two-digit zero-padded number
- Location: directly inside the subject directory — `<subject>/unit{NN}.md`
- Examples: `algorithm/unit18.md`, `database/unit15.md`
- Never create subdirectories for units

---

## 3. Mandatory Markdown Structure

Every unit file MUST follow this exact heading hierarchy.
Do NOT use H1 (`#`) or H4+ (`####`) anywhere in unit files.

```markdown
## UNIT 주제명

(개요 1–2문장)

<br>

### 1. 소주제명

### 2. 소주제명

### 2-1. 세부 소주제명

### 2-2. 세부 소주제명
```

- File starts at `##` (H2) — this is the top-level title
- Sub-sections use `###` (H3) with sequential numbers: `### 1.`, `### 2.`
- Sub-sub-sections: `### 2-1.`, `### 2-2.` format
- H4 and below are **forbidden** — use `**bold text**` as sub-labels instead
- Insert `<br>` between every section for visual spacing
- Use `---` horizontal rules sparingly — only for major structural divisions

---

## 4. Callout Formats

Three callout types — use exactly these formats, no variations:

```markdown
> 💡 팁이나 보충 설명 내용

> ⚠️ 주의해야 할 내용

❗️**인라인 경고 제목**: 본문 중 예외나 경고 사항
```

- `> 💡` — tips, supplementary explanations, language-specific notes
- `> ⚠️` — warnings, common pitfalls, performance limitations
- `❗️**볼드**` — inline warnings within body text (not a block quote)

---

## 5. Table Format

Use tables for comparisons, classifications, and complexity summaries.

```markdown
| **항목**     | **설명**      | **예시**         |
| ------------ | ------------- | ---------------- |
| **O(1)**     | 상수 시간     | 배열 인덱스 접근 |
```

- Header cells use `**bold**`
- Key values in data cells use `**bold**`
- Align separator dashes with at least 3 dashes per cell

---

## 6. Code Block Format

Always specify the language identifier. Use plain blocks (no language) only for
ASCII diagrams and step-by-step process illustrations.

````markdown
```python
arr = [1, 2, 3]
for x in arr:
    print(x)
```

```java
int[] arr = new int[5];
```

```
[A] → [B] → [C]   # ASCII diagram — no language tag
```
````

---

## 7. Image Format

Images are stored in the companion repo `Team-Gravit/gravit-images`.
Always use absolute raw GitHub URLs. Relative paths are forbidden.

```markdown
<img src="https://raw.githubusercontent.com/Team-Gravit/gravit-images/main/<subject>/unit{NN}/<filename>.png" width="100%">
```

- `width="100%"` is mandatory on every image tag
- Path structure: `<subject>/unit{NN}/<filename>`
- Examples:
  - `data-structure/unit01/image.png`
  - `algorithm/unit11/image1.png`
  - `database/unit01/image2.png`
- When writing a new unit and image filenames are not yet confirmed, insert a placeholder:
  ```markdown
  <img src="https://raw.githubusercontent.com/Team-Gravit/gravit-images/main/<subject>/unit{NN}/image1.png" width="100%">
  ```

---

## 8. Quality Standards

### Minimum Requirements (a file failing any of these needs improvement)

| Check                    | Minimum                               |
| ------------------------ | ------------------------------------- |
| Line count               | ≥ 80 lines                            |
| Sections (`###`)         | ≥ 4 distinct `###` sections           |
| Callouts                 | ≥ 2 callout blocks (`> 💡` or `> ⚠️`) |
| Code example             | ≥ 1 fenced code block                 |
| `<br>` spacing           | Present between every section         |
| Language                 | 100% Korean (code keywords excepted)  |

### Target Quality (reference: `data-structure/unit01.md`, `algorithm/unit11.md`)

- 120–200+ lines
- 6–10 sections
- At least one comparison table
- At least one code block showing implementation or usage
- At least one diagram (image placeholder or ASCII art)
- 2–4 callout blocks distributed throughout
- Complexity analysis table where applicable

### Known Underperforming Files (priority targets for `/improve-unit`)

- `algorithm/unit01.md` — 43 lines, no code example, only 3 sections → needs full improvement

---

## 9. Content Inclusion Criteria

Include a topic only if it meets at least one criterion:

- Frequently tested in Korean coding interviews (e.g., backtracking, Dijkstra, DP)
- Fundamental CS concept required by Korean CS curricula
- Essential prerequisite for understanding other units in this repo

Exclude topics outside CS fundamentals scope:
- Advanced ML/AI topics (e.g., reinforcement learning algorithms)
- Framework-specific APIs
- Topics not relevant to the core five subjects

---

## 10. Subject Unit Topic Map

### algorithm (unit01–23)

| Unit   | Topic                                   |
| ------ | --------------------------------------- |
| unit01 | 시간 복잡도와 Big-O 표기법              |
| unit02 | 공간 복잡도                             |
| unit03 | 브루트 포스                             |
| unit04 | 백트래킹                                |
| unit05 | 버블 정렬                               |
| unit06 | 선택 정렬                               |
| unit07 | 삽입 정렬                               |
| unit08 | 합병 정렬                               |
| unit09 | 퀵 정렬                                 |
| unit10 | 힙 정렬                                 |
| unit11 | 기수 정렬                               |
| unit12 | 위상 정렬                               |
| unit13 | DFS·BFS                                 |
| unit14 | 그리디 알고리즘                         |
| unit15 | 다이내믹 프로그래밍                     |
| unit16 | 최소 신장 트리(MST)                     |
| unit17 | 최단 경로 알고리즘                      |
| unit18 | 재귀와 분할 정복                        |
| unit19 | 이진 탐색                               |
| unit20 | 파라메트릭 서치와 lower/upper bound     |
| unit21 | 투 포인터와 슬라이딩 윈도우             |
| unit22 | 동적 프로그래밍 활용                    |
| unit23 | 문자열 탐색 — KMP                       |

### data-structure (unit01–12)

| Unit   | Topic                             |
| ------ | --------------------------------- |
| unit01 | 배열(Array)                       |
| unit02 | 연결리스트(Linked List)           |
| unit03 | 스택과 큐(Stack & Queue)          |
| unit04 | 트리(Tree)                        |
| unit05 | 이진 트리와 이진 탐색 트리(BST)   |
| unit06 | 힙(Heap)                          |
| unit07 | 트라이(Trie)                      |
| unit08 | 균형 이진 탐색 트리               |
| unit09 | 해시테이블(Hash Table)            |
| unit10 | 그래프(Graph)                     |
| unit11 | 덱(Deque)과 우선순위 큐           |
| unit12 | 유니온-파인드(Union-Find)         |

### database (unit01–21)

| Unit   | Topic                                     |
| ------ | ----------------------------------------- |
| unit01 | 데이터 모델링 기본                        |
| unit02 | 식별 관계와 비식별 관계                   |
| unit03 | 관계형 모델 개념                          |
| unit04 | 키(Key)                                   |
| unit05 | 외래키와 제약 조건                        |
| unit06 | DDL                                       |
| unit07 | DML                                       |
| unit08 | 서브쿼리 기초                             |
| unit09 | 조인(JOIN)                                |
| unit10 | 페이징(Pagination)                        |
| unit11 | 뷰(View)                                  |
| unit12 | 정규화(Normalization)                     |
| unit13 | 트랜잭션(Transaction)                     |
| unit14 | 인덱스(Index)                             |
| unit15 | SQL 개요와 명령어 분류                    |
| unit16 | 집계 함수와 GROUP BY·HAVING               |
| unit17 | 동시성 제어 — 격리 수준·락·MVCC           |
| unit18 | SQL 튜닝과 실행 계획                      |
| unit19 | 커넥션과 커넥션 풀                        |
| unit20 | 확장 전략 — 파티셔닝·샤딩·레플리케이션    |
| unit21 | NoSQL과 CAP 이론                          |

### network (unit01–22)

| Unit   | Topic                                |
| ------ | ------------------------------------ |
| unit01 | 네트워크 기초                        |
| unit02 | 네트워크 토폴로지                    |
| unit03 | 프로토콜과 계층 구조                 |
| unit04 | 데이터 단위와 캡슐화                 |
| unit05 | 물리 계층                            |
| unit06 | 데이터 링크 계층                     |
| unit07 | 네트워크 계층                        |
| unit08 | 서브넷과 라우팅                      |
| unit09 | TCP                                  |
| unit10 | 포트(Port)                           |
| unit11 | 응용 계층                            |
| unit12 | 웹 접속 과정과 데이터 흐름           |
| unit13 | 무선 네트워크                        |
| unit14 | 네트워크 보안                        |
| unit15 | ARP와 DHCP                           |
| unit16 | TCP 신뢰성 — 흐름·혼잡·오류 제어     |
| unit17 | DNS                                  |
| unit18 | HTTP 심화 — 버전과 캐시              |
| unit19 | HTTPS와 TLS                          |
| unit20 | 쿠키·세션·토큰                       |
| unit21 | REST API                             |
| unit22 | 로드밸런싱·프록시·CDN                |

### operating-system (unit01–14)

| Unit   | Topic                     |
| ------ | ------------------------- |
| unit01 | 운영체제 개요             |
| unit02 | 프로세스 기초             |
| unit03 | 시스템 콜                 |
| unit04 | 인터럽트                  |
| unit05 | 프로세스 관리             |
| unit06 | 스레드와 멀티스레딩       |
| unit07 | CPU 스케줄링              |
| unit08 | 동기화와 병행성           |
| unit09 | 데드락                    |
| unit10 | 메모리 관리 기초          |
| unit11 | 가상 메모리               |
| unit12 | 페이지 관리               |
| unit13 | 캐시 메모리               |
| unit14 | 파일 시스템과 디스크 관리 |

### web-security (unit01–13)

| Unit   | Topic                             |
| ------ | --------------------------------- |
| unit01 | 웹 보안 기초와 OWASP Top 10       |
| unit02 | SQL 인젝션                        |
| unit03 | XSS                               |
| unit04 | CSRF                              |
| unit05 | SOP와 CORS                        |
| unit06 | 인증과 인가 — 세션 vs JWT         |
| unit07 | 세션 공격 — 하이재킹·고정         |
| unit08 | 암호학 기초                       |
| unit09 | 비밀번호의 안전한 저장            |
| unit10 | HTTPS 인증서와 CA 체인            |
| unit11 | 파일 업로드 취약점                |
| unit12 | SSRF·오픈 리다이렉트·클릭재킹     |
| unit13 | 접근 통제와 시큐어 코딩           |

---

## 11. New Chapter Unit Topic Map (2026-09-08)

개편안 문서(`docs/curriculum.md` 및 신규 챕터-유닛 표) 기반으로 추가된 챕터. 이미지 대신 ASCII 다이어그램을 사용한다.

### common-server (unit01–12) — Common · Server

| Unit   | Topic |
| ------ | ----- |
| unit01 | 트랜잭션과 격리 수준 |
| unit02 | 인덱스와 실행 계획 |
| unit03 | 동시성 제어와 락 |
| unit04 | 분산 환경의 정합성 |
| unit05 | 캐시 전략 |
| unit06 | 비동기 메시징 |
| unit07 | 데이터베이스 확장 |
| unit08 | HTTP와 REST API 설계 |
| unit09 | 인증과 인가 |
| unit10 | 트래픽 처리와 자원 관리 |
| unit11 | 배포와 무중단 전환 |
| unit12 | 관측성 |

### common-web (unit01–10) — Common · Web

| Unit   | Topic |
| ------ | ----- |
| unit01 | 브라우저 렌더링 파이프라인 |
| unit02 | Reflow와 Repaint |
| unit03 | 이벤트 루프와 실행 순서 |
| unit04 | 리소스 로딩과 파서 차단 |
| unit05 | 렌더링 전략 |
| unit06 | 브라우저 저장소와 세션 유지 |
| unit07 | 동일 출처 정책과 CORS |
| unit08 | 웹 보안 |
| unit09 | HTTP 캐싱 |
| unit10 | 웹 성능 지표와 측정 |

### common-aos (unit01–10) — Common · AOS

| Unit   | Topic |
| ------ | ----- |
| unit01 | 4대 컴포넌트와 인텐트 |
| unit02 | 액티비티와 프래그먼트 생명주기 |
| unit03 | 구성 변경과 상태 보존 |
| unit04 | 프로세스 수명과 메모리 압력 |
| unit05 | 메모리 누수 패턴 |
| unit06 | 백그라운드 작업 |
| unit07 | 데이터 저장소 선택 |
| unit08 | 네트워크 계층 |
| unit09 | 앱 아키텍처 |
| unit10 | 권한과 보안 |

### common-ios (unit01–10) — Common · iOS

| Unit   | Topic |
| ------ | ----- |
| unit01 | 앱 생명주기와 상태 전이 |
| unit02 | 뷰 컨트롤러 생명주기 |
| unit03 | 메모리 압력과 백그라운드 종료 |
| unit04 | 셀 재사용과 리스트 성능 |
| unit05 | 데이터 영속화 선택 |
| unit06 | 네트워크 계층 |
| unit07 | 샌드박스와 앱 간 공유 |
| unit08 | 앱 아키텍처 |
| unit09 | 푸시 알림 |
| unit10 | 권한과 프라이버시 |

### be-spring (unit01–11) — BE · Spring

| Unit   | Topic |
| ------ | ----- |
| unit01 | IoC/DI와 빈 생명주기 |
| unit02 | AOP와 프록시 동작 |
| unit03 | 요청 처리 흐름 |
| unit04 | 요청 전후 처리 계층 |
| unit05 | 트랜잭션 추상화 |
| unit06 | 영속성 컨텍스트 |
| unit07 | 연관관계와 N+1 |
| unit08 | JPA 쓰기와 식별자 전략 |
| unit09 | 예외 처리와 응답 규약 |
| unit10 | 인증·인가 필터 체인 |
| unit11 | 스레드와 커넥션 자원 관리 |

### be-nodejs (unit01–09) — BE · Node.js

| Unit   | Topic |
| ------ | ----- |
| unit01 | 이벤트 루프와 논블로킹 I/O |
| unit02 | 마이크로태스크와 매크로태스크 |
| unit03 | CPU 바운드 작업 처리 |
| unit04 | 모듈 시스템 |
| unit05 | 스트림과 백프레셔 |
| unit06 | 모듈과 프로바이더 스코프 (NestJS) |
| unit07 | 요청 처리 파이프라인 (NestJS) |
| unit08 | 에러 핸들링과 프로세스 안정성 |
| unit09 | 메모리 누수 진단 |

### be-django (unit01–09) — BE · Django

| Unit   | Topic |
| ------ | ----- |
| unit01 | 요청·응답 사이클과 미들웨어 |
| unit02 | QuerySet 지연 평가 |
| unit03 | 조회 최적화와 N+1 |
| unit04 | 트랜잭션 관리 |
| unit05 | 마이그레이션 |
| unit06 | 인증과 권한 |
| unit07 | DRF 직렬화와 검증 |
| unit08 | 시그널과 암묵적 결합 |
| unit09 | 동기와 비동기 혼용 |

### fe-react (unit01–10) — FE · React

| Unit   | Topic |
| ------ | ----- |
| unit01 | 렌더링과 재조정 |
| unit02 | 훅의 동작 원리 |
| unit03 | 상태 설계 |
| unit04 | 상태 관리 전략 |
| unit05 | useEffect의 정의와 오용 |
| unit06 | 렌더링 최적화 판단 |
| unit07 | ref와 DOM 접근 |
| unit08 | 동시성 렌더링 |
| unit09 | 에러 경계와 예외 처리 |
| unit10 | 컴포넌트 설계 |

### fe-vue (unit01–07) — FE · Vue.js

| Unit   | Topic |
| ------ | ----- |
| unit01 | 반응성 시스템 원리 |
| unit02 | ref와 reactive |
| unit03 | computed와 watch |
| unit04 | Composition API와 Options API |
| unit05 | 컴포넌트 통신 |
| unit06 | 렌더링 제어 |
| unit07 | 생명주기와 DOM 접근 시점 |

### fe-nextjs (unit01–09) — FE · Next.js

| Unit   | Topic |
| ------ | ----- |
| unit01 | 렌더링 전략 선택 |
| unit02 | 서버·클라이언트 컴포넌트 경계 |
| unit03 | 데이터 페칭과 폭포수 |
| unit04 | 캐싱 레이어 |
| unit05 | 재검증 전략 |
| unit06 | 서버 액션 |
| unit07 | 하이드레이션 |
| unit08 | 라우팅·레이아웃과 미들웨어 |
| unit09 | 자산과 번들 최적화 |

### mobile-android (unit01–10) — Mobile · Android

| Unit   | Topic |
| ------ | ----- |
| unit01 | ViewModel과 상태 보존 |
| unit02 | 선언형 UI와 Recomposition |
| unit03 | Compose 상태 관리 |
| unit04 | Compose 안정성과 최적화 |
| unit05 | Compose 부수효과 API |
| unit06 | 생명주기 인식 데이터 수집 |
| unit07 | 의존성 주입 |
| unit08 | Navigation과 화면 전환 |
| unit09 | 리스트 성능 |
| unit10 | 테스트 가능한 구조 |

### mobile-ios (unit01–09) — Mobile · iOS

| Unit   | Topic |
| ------ | ----- |
| unit01 | SwiftUI 뷰 정체성과 업데이트 |
| unit02 | 상태 프로퍼티 래퍼 |
| unit03 | Observation 프레임워크 |
| unit04 | 뷰 모디파이어와 재사용 설계 |
| unit05 | UIKit과 SwiftUI 상호운용 |
| unit06 | 레이아웃 시스템 |
| unit07 | 리스트 성능 |
| unit08 | 화면 전환과 흐름 제어 |
| unit09 | Combine과 데이터 바인딩 |

### lang-java (unit01–10) — Language · Java

| Unit   | Topic |
| ------ | ----- |
| unit01 | JVM 메모리 구조 |
| unit02 | 가비지 컬렉션 |
| unit03 | equals와 hashCode 계약 |
| unit04 | 컬렉션 내부 구조 |
| unit05 | 예외 체계와 설계 |
| unit06 | 제네릭과 타입 소거 |
| unit07 | 스트림과 함수형 인터페이스 |
| unit08 | 동시성 기본 |
| unit09 | 고수준 동시성 API |
| unit10 | 불변 객체와 값 설계 |

### lang-kotlin (unit01–09) — Language · Kotlin

| Unit   | Topic |
| ------ | ----- |
| unit01 | Null Safety 설계 |
| unit02 | 코루틴 구조화된 동시성 |
| unit03 | 코루틴 예외 처리 |
| unit04 | Flow |
| unit05 | 확장 함수와 스코프 함수 |
| unit06 | data·sealed·object |
| unit07 | 위임과 프로퍼티 |
| unit08 | 가시성과 불변성 |
| unit09 | 자바 상호운용 |

### lang-typescript (unit01–07) — Language · TypeScript

| Unit   | Topic |
| ------ | ----- |
| unit01 | 구조적 타이핑 |
| unit02 | 제네릭과 제약 |
| unit03 | 타입 좁히기 |
| unit04 | any·unknown·never |
| unit05 | 인터페이스와 타입 별칭 |
| unit06 | 유틸리티 타입과 매핑 타입 |
| unit07 | 컴파일과 런타임 경계 |

### lang-python (unit01–08) — Language · Python

| Unit   | Topic |
| ------ | ----- |
| unit01 | GIL과 동시성 모델 |
| unit02 | 메모리 관리 |
| unit03 | 가변·불변 객체와 함정 |
| unit04 | 이터레이터와 제너레이터 |
| unit05 | 데코레이터 동작 원리 |
| unit06 | 클래스와 MRO |
| unit07 | 타입 힌트 |
| unit08 | 예외와 컨텍스트 매니저 |

### lang-swift (unit01–08) — Language · Swift

| Unit   | Topic |
| ------ | ----- |
| unit01 | 값 타입 vs 참조 타입 |
| unit02 | ARC와 순환 참조 |
| unit03 | 옵셔널 처리 전략 |
| unit04 | 프로토콜 지향 프로그래밍 |
| unit05 | 제네릭과 associatedtype |
| unit06 | Swift Concurrency |
| unit07 | actor와 데이터 격리 |
| unit08 | 에러 처리 |
