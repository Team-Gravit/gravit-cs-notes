# CLAUDE.md

This file provides guidance to Claude Code when working in this repository.
Read this file completely before creating, editing, or reviewing any unit file.

---

## 1. Repository Overview

Team-Gravit CS educational notes repository. All content is written in **Korean**
(variable names and language keywords in code blocks are the only exception).
Pure documentation repo — no build tools, linters, or test frameworks.

**Five subjects and their current unit status:**

| Subject           | Current Last Unit | Next Unit to Create |
| ----------------- | ----------------- | ------------------- |
| `algorithm/`      | unit23            | unit24              |
| `data-structure/` | unit12            | unit13              |
| `database/`       | unit21            | unit22              |
| `network/`        | unit22            | unit23              |
| `web-security/`   | unit13            | unit14              |

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
