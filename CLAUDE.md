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
| `algorithm/`      | unit17            | unit18              |
| `data-structure/` | unit10            | unit11              |
| `database/`       | unit14            | unit15              |
| `network/`        | unit14            | unit15              |
| `web-security/`   | unit01 (empty)    | unit01 (write this) |

> ⚠️ `web-security/unit01` currently exists as a nested directory
> (`web-security/unit01/unit01.md`) with empty content — this is an incorrect structure.
> When writing web-security unit01, create `web-security/unit01.md` directly
> (matching the pattern of all other subjects), not inside a subdirectory.

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
- `web-security/unit01/unit01.md` — empty file, wrong directory structure → write `web-security/unit01.md` instead

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

### algorithm (unit01–17)

| Unit   | Topic                          |
| ------ | ------------------------------ |
| unit01 | 시간복잡도와 Big-O             |
| unit02 | 공간복잡도 / 점근적 표기법     |
| unit03 | 배열·문자열                    |
| unit04 | 스택                           |
| unit05 | 큐                             |
| unit06 | 힙                             |
| unit07 | 재귀                           |
| unit08 | 정렬 (버블·선택·삽입)          |
| unit09 | 정렬 (합병·퀵·기수)            |
| unit10 | 이진 탐색                      |
| unit11 | DFS·BFS                        |
| unit12 | 백트래킹                       |
| unit13 | 다익스트라                     |
| unit14 | 벨만-포드 / 플로이드-워셜      |
| unit15 | 동적 프로그래밍 기초           |
| unit16 | 동적 프로그래밍 심화           |
| unit17 | 그리디 알고리즘                |

### data-structure (unit01–10)

| Unit   | Topic                             |
| ------ | --------------------------------- |
| unit01 | 배열(Array)                       |
| unit02 | 연결리스트(Linked List)           |
| unit03 | 스택(Stack) & 큐(Queue)           |
| unit04 | 트리(Tree) 기초                   |
| unit05 | 이진 트리 & 이진 탐색 트리(BST)   |
| unit06 | 힙(Heap)                          |
| unit07 | 우선순위 큐(Priority Queue)       |
| unit08 | 그래프(Graph)                     |
| unit09 | 해시 테이블(Hash Table)           |
| unit10 | 트라이(Trie) & 유니온-파인드      |

### database (unit01–14)

| Unit      | Topic                    |
| --------- | ------------------------ |
| unit01    | 데이터 모델링 기본       |
| unit02    | E-R 다이어그램           |
| unit03    | 정규화                   |
| unit04–14 | SQL, 트랜잭션, 인덱스 등 |

### network (unit01–14)

| Unit      | Topic                |
| --------- | -------------------- |
| unit01    | 네트워크 기초        |
| unit02–14 | TCP/IP, HTTP, DNS 등 |

### web-security

| Unit   | Status                          |
| ------ | ------------------------------- |
| unit01 | 비어있음 — `/write-unit`으로 작성 필요 |
