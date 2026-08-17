새로운 CS 개념노트 unit 파일을 작성한다.

## 사용법
`/write-unit <subject> <topic>`

예시:
- `/write-unit web-security XSS공격`
- `/write-unit algorithm 위상정렬`
- `/write-unit network HTTP헤더`

## Arguments
$ARGUMENTS 형식: `<subject> <topic>` — 첫 번째 토큰은 subject 디렉토리명, 나머지는 주제명(한국어).

---

## 실행 지침

### Step 1 — 출력 경로 결정

1. `$ARGUMENTS`를 파싱해 `SUBJECT`(첫 번째 공백 구분 토큰)와 `TOPIC`(나머지 전체)을 추출한다.
2. `CLAUDE.md` 섹션 1의 "Next Unit to Create" 컬럼에서 해당 subject의 다음 번호를 확인한다.
3. 출력 경로: `<SUBJECT>/unit{NN}.md`

   **web-security 예외:** `web-security/unit01/unit01.md`(빈 파일)가 존재하고
   `web-security/unit01.md`가 없는 경우, unit02가 아닌 `web-security/unit01.md`에 작성한다.

4. 경로를 확정한 뒤 Step 2로 진행한다.

### Step 2 — 자료 수집 (병렬 WebSearch)

다음 4개의 검색을 동시에 실행한다:

- 검색 1: "`TOPIC` CS 개념 원리 설명"
- 검색 2: "`TOPIC` 동작 방식 예시 단계별"
- 검색 3: "`TOPIC` 시간복잡도 공간복잡도 장단점" (복잡도가 없는 주제면 생략)
- 검색 4: "`TOPIC` 코딩테스트 한국 자주 출제"

수집한 정보를 종합해 정확하고 충분한 내용을 확보한다. 기술적 사실은 절대 임의로 만들지 않는다.

### Step 3 — 개념노트 작성

`CLAUDE.md`의 모든 규칙을 따라 완성된 unit 파일을 작성한다.

**작성 전 체크리스트:**
- [ ] 파일 최상단이 `## TOPIC명` (H2) 인가
- [ ] `###` 섹션이 4개 이상인가
- [ ] `<br>` 태그가 모든 섹션 사이에 있는가
- [ ] callout 블록(`> 💡` 또는 `> ⚠️`)이 2개 이상인가
- [ ] 언어 명시 fenced code block이 1개 이상인가
- [ ] 비교 또는 요약 표가 1개 이상인가
- [ ] 전체 한국어 (코드 키워드 제외)
- [ ] 복잡도가 있는 주제면 복잡도 표 포함
- [ ] 다이어그램이 필요한 경우 이미지 placeholder 또는 ASCII 다이어그램 삽입

**권장 섹션 구성** (주제에 맞게 조정):
```
## TOPIC명

(개요 1–2문장: 이 개념이 무엇이고 왜 중요한가)

<img src="https://raw.githubusercontent.com/Team-Gravit/gravit-images/main/SUBJECT/unit{NN}/image1.png" width="100%">

<br>

### 1. 핵심 개념 / 특징

### 2. 동작 원리 / 구조

### 3. 동작 과정 예시  ← ASCII 다이어그램 또는 이미지 placeholder

### 4. 구현 예시      ← 코드블록 필수

### 5. 시간·공간 복잡도  ← 표로 정리

### 6. 장점 / 단점

### 7. 관련 개념과 비교 / 사용 사례  ← 비교 표
```

목표 분량: 최소 120줄, 이상적으로 150줄 이상.

### Step 4 — 파일 저장 및 결과 출력

결정된 경로에 파일을 Write로 저장한 뒤 아래 형식으로 요약을 출력한다:

```
✅ 파일 생성 완료
경로: <SUBJECT>/unit{NN}.md
주제: TOPIC
섹션 수: N개
줄 수: N줄
```

작성 후 `CLAUDE.md` 섹션 8의 "Minimum Requirements"를 기준으로 self-check를 수행한다.
미달 항목이 있으면 즉시 수정한 뒤 파일을 저장한다.
