# web-security — 레퍼런스 모음

`web-security/`는 현재 `unit01`이 비어 있는 상태다. 이 문서는 **unit01부터 새로 작성**하기 위한
외부 자료 **66건** (1차 36 + 2차 30)을 정리한 것이다.

> ⚠️ 현재 `web-security/unit01/unit01.md`(중첩 디렉터리) 구조는 잘못되었다.
> 다른 과목과 동일하게 `web-security/unit01.md`로 작성해야 한다. (CLAUDE.md §1 참조)

<br>

## 0. UNIT 구성 제안

수집 자료를 근거로 한 초기 unit 로드맵이다. 모두 OWASP Top 10 및 정보보안기사 출제 범위와 겹친다.

| UNIT   | 주제 (제안)                        | 근거                                   |
| ------ | ---------------------------------- | -------------------------------------- |
| unit01 | 웹 보안 기초와 OWASP Top 10        | 업계 표준 취약점 분류                  |
| unit02 | SQL 인젝션                         | OWASP A03, 정보보안기사·정보처리기사   |
| unit03 | XSS (Cross-Site Scripting)         | OWASP A03, 실무 빈출                   |
| unit04 | CSRF                               | 국내 기술면접 빈출                     |
| unit05 | 동일 출처 정책(SOP)과 CORS         | 프론트-백엔드 협업 필수 개념           |
| unit06 | 인증과 인가 — 세션 vs JWT          | OWASP A07, 백엔드 면접 빈출            |
| unit07 | 암호학 기초 — 대칭키·공개키·해시   | 정보보안기사 필수, HTTPS 선행 지식     |
| unit08 | 접근 통제와 시큐어 코딩 가이드     | OWASP A01, KISA 개발보안 가이드        |

<br>

## 1. 기술 블로그 (21)

### 1-1. 주요 취약점 개관

| #   | 제목                                            | 링크                                                                    | 관련 UNIT      |
| --- | ----------------------------------------------- | ------------------------------------------------------------------------- | -------------- |
| 1   | 웹 보안 취약점 정리 (Injection, XSS, CSRF)      | https://velog.io/@jackjack/웹-보안-취약점-정리Injection-XSS-CSRF          | unit01~04      |
| 2   | 웹 해킹 공격 (XSS, CSRF, SQL Injection 등)      | https://velog.io/@dudgus1670/XSS-CSRF-SQL-Injection                       | unit02~04      |
| 3   | 보안 — XSS, CSRF, SQL INJECTION                 | https://velog.io/@yshjft/보안-XSS-CSRF-SQL-INJECTION                      | unit02~04      |
| 4   | XSS, CSRF, SQL Injection이란                    | https://velog.io/@sinclebear/XSS-CSRF-SQL-Injection이란                   | unit02~04      |
| 5   | XSS, SQL Injection 방어하기 — 웹취약점 분석     | https://webhack.dynu.net/?idx=20161109.002                               | unit02, 03     |
| 6   | 웹 보안 취약점 사전 — 15가지 완전 해설 (SecuFi) | https://secufi.kr/vulnerabilities                                        | unit01         |
| 7   | XSS 공격 vs SQL 인젝션, 차이점과 예방 (SK쉴더스) | https://www.skshieldus.com/blog-security/security-trend-idx-45          | unit02, 03     |
| 8   | 웹 보안의 핵심: CORS, SOP, XSS, CSRF, SQL Injection | https://f-lab.kr/insight/understanding-web-security-20260516         | unit01, 05     |

### 1-2. OWASP Top 10 해설

| #   | 제목                                        | 링크                                              | 관련 UNIT |
| --- | ------------------------------------------- | --------------------------------------------------- | --------- |
| 9   | **OWASP Top 10 - 2021 톺아보기 (넷마블 기술블로그)** | https://netmarble.engineering/owasp-top-10-2021-1/ | unit01 |
| 10  | 2021 OWASP TOP10 (이스트시큐리티 알약 블로그) | https://blog.alyac.co.kr/4135                      | unit01    |
| 11  | OWASP Top 10 2021 해설 (IDCHOWTO)           | https://idchowto.com/owasp-top-10-2021/            | unit01    |

### 1-3. SOP · CORS

| #   | 제목                                     | 링크                                                                       | 관련 UNIT |
| --- | ---------------------------------------- | ---------------------------------------------------------------------------- | --------- |
| 12  | CORS가 대체 무엇일까? (feat. SOP) — hudi | https://hudi.blog/sop-and-cors/                                             | unit05    |
| 13  | 동일 출처 정책(SOP)과 CORS               | https://jaehyeon48.github.io/web/sop-and-cors/                              | unit05    |
| 14  | 이해하기 쉬운 웹 보안 모델 이야기 (SOP, CORS) | https://blog.zairo.kr/entry/이해하기-쉬운-웹-보안-모델-이야기-1-SOP-CORS  | unit05    |

### 1-4. 인증 · 인가 · 암호

| #   | 제목                                              | 링크                                                                              | 관련 UNIT |
| --- | ------------------------------------------------- | ----------------------------------------------------------------------------------- | --------- |
| 15  | 세션 기반 인증과 토큰 기반 인증 (feat. 인증과 인가) | https://hudi.blog/session-based-auth-vs-token-based-auth/                          | unit06    |
| 16  | 로그인 인증방식 Session VS JWT (팔만코딩경)       | https://80000coding.oopy.io/1f213f10-185c-4b4e-8372-119402fecdd0                   | unit06    |
| 17  | 세션 기반 인증 방식과 토큰 기반 인증(JWT)         | https://yonghyunlee.gitlab.io/node/jwt/                                            | unit06    |
| 18  | Spring Security Part2 — OAuth 2.0 아키텍처와 보안 취약점 사례 (이글루코퍼레이션) | https://www.igloo.co.kr/security-information/spring-security-part2-oauth-2-0-아키텍처-이해와-보안-취약점-사례/ | unit06 |
| 19  | 대칭키, 비대칭키, 공개키, 개인키, 전자서명        | https://ansohxxn.github.io/bitcoin/cryptography/                                   | unit07    |
| 20  | 정보보호론 — 대칭키·공개키 암호화, HASH, PKI      | https://velog.io/@sjoh0704/정보보호론-대칭키-암호화-공개키-암호화-HASH-PKI-정리     | unit07    |
| 21  | 공개 키 암호 방식 (위키백과)                      | https://ko.wikipedia.org/wiki/공개_키_암호_방식                                    | unit07    |

<br>

## 2. 공식 표준 · 정부 가이드 (6)

| #   | 자료                                                | 링크                                                                                          | 활용 포인트                        |
| --- | --------------------------------------------------- | ----------------------------------------------------------------------------------------------- | ---------------------------------- |
| 22  | **OWASP Top 10:2021 (공식)**                        | https://owasp.org/Top10/2021/                                                                  | unit01 분류 체계의 1차 근거        |
| 23  | OWASP Top 10 2021 한글본 (라스컴 시큐리티)          | https://lassecu.com/references/information-protection-guide/68                                 | 한국어 용어 통일                   |
| 24  | **KISA 한국인터넷진흥원 — 소프트웨어 개발보안**     | https://www.kisa.or.kr/2060204                                                                 | 공식 가이드 원문                   |
| 25  | 행정안전부 — 소프트웨어 개발보안(시큐어 코딩) 가이드 | https://www.mois.go.kr/frt/bbs/type001/commonSelectBoardArticle.do?bbsId=BBSMSTR_000000000045&nttId=34430 | unit08 근거 |
| 26  | 공공데이터포털 — 소프트웨어 개발보안 가이드 (2021)  | https://www.data.go.kr/data/15049187/fileData.do                                               | PDF 다운로드                       |
| 27  | KISA 암호이용활성화 — 암호기술의 정의               | https://seed.kisa.or.kr/kisa/intro/EgovDefinition.do                                           | unit07 용어 정의                   |

> 💡 진단 항목 분류(**입력값 검증 / 인증·접근제어 / 암호화 / 에러처리 / 세션관리 / 시스템 설정**)는
> KISA 개발보안 가이드의 공식 체계다. unit 구성의 뼈대로 삼기에 적합하다.

<br>

## 3. 자격증 — 정보보안기사 (5)

| #   | 자료                                        | 링크                                                                                       | 활용 포인트                |
| --- | ------------------------------------------- | -------------------------------------------------------------------------------------------- | -------------------------- |
| 28  | 정보보안기사 시험 가이드 (인프런)           | https://www.inflearn.com/pages/information-security-engineer-exam-guide                     | 5과목 출제 범위 확인       |
| 29  | 정보보안기사 (나무위키)                     | https://namu.wiki/w/정보보안기사                                                            | 시험 체계·과목 구성        |
| 30  | 정보보안기사 요약 1. 시스템 보안 (JMoon)    | https://jmoon.co.kr/23                                                                      | 과목별 요약                |
| 31  | 정보보안기사 필기 요약 (합본)               | https://www.scribd.com/document/703882877/정보보안기사-필기-요약-합본                       | 전 과목 요약               |
| 32  | 정보보안기사 동회차 합격 후기 (필기·실기)   | https://blog.system32.kr/506                                                                | 출제 경향·공부 순서        |

<br>

## 4. 실습 · 워게임 (4)

| #   | 자료                                     | 링크                                                        | 활용 포인트                        |
| --- | ---------------------------------------- | ------------------------------------------------------------- | ---------------------------------- |
| 33  | **Dreamhack — Web Hacking 기초 코스**    | https://dreamhack.io/lecture/paths/web-hacking-fundamental   | unit별 실습 문제 매핑              |
| 34  | Dreamhack.io (나무위키) — 플랫폼 소개    | https://namu.wiki/w/Dreamhack.io                             | 학습 경로 파악                     |
| 35  | webhacking.kr — 웹 해킹 워게임           | https://webhacking.kr/                                       | 심화 실습                          |
| 36  | 해킹 공부하기 좋은 Wargame 총집합        | https://blog.sechack.kr/54                                   | 추가 실습 사이트 목록              |

> ⚠️ 실습 자료는 **본인 소유 또는 명시적으로 허용된 환경**에서만 사용해야 한다.
> 개념노트에는 공격 절차의 재현 방법이 아니라 **취약점의 원리와 방어 코드**를 중심으로 서술한다.

<br>

---

<br>

# 2차 수집분 (30건)

1차가 XSS·CSRF·SQLi에 집중되어 있어, **파일 업로드 · SSRF · 세션 공격 · 비밀번호 저장 ·
인증서 검증 · 시큐어 코딩**을 보강했다. 아래 자료를 반영해 UNIT 로드맵을 다음과 같이 확장할 수 있다.

| UNIT (확장 제안) | 주제                          | 근거                          |
| ---------------- | ----------------------------- | ----------------------------- |
| unit09           | 파일 업로드 취약점과 웹쉘     | 국내 침해사고 최다 초기 침투 경로 |
| unit10           | SSRF · 오픈 리다이렉트 · 클릭재킹 | OWASP A10 / 피싱 연계        |
| unit11           | 세션 공격 (하이재킹 · 고정)   | OWASP A07, 정보보안기사       |
| unit12           | 비밀번호 안전한 저장 (해싱·솔트) | 개인정보보호법 실무 필수     |
| unit13           | HTTPS 인증서와 CA 체인 검증   | network TLS unit과 연계       |

<br>

## 5. 파일 업로드 취약점 · 웹쉘 (6)

| #   | 제목                                          | 링크                                                                          | 관련 UNIT   |
| --- | --------------------------------------------- | ------------------------------------------------------------------------------- | ----------- |
| 37  | 파일 업로드 취약점 — 웹 쉘(Web Shell)         | https://velog.io/@woo2083/파일-업로드-취약점-웹-쉘Web-Shell-kjnlvf1n            | unit09 후보 |
| 38  | [보안] 파일 업로드 취약점 (낮코밤코)          | https://owin2828.github.io/devlog/2020/01/09/etc-2.html                        | unit09 후보 |
| 39  | 파일업로드 공격 실습 (File Upload Attack)     | https://c0msherl0ck.github.io/web/post-file_upload_attack/                      | unit09 후보 |
| 40  | 파일 업로드 취약점 진단 가이드                | https://velog.io/@mandu0707/취약점-파일-업로드-취약점-File-Upload-Vulnerability | unit09 후보 |
| 41  | [웹 취약점] 파일 업로드 취약점 (정보보안 기록 저장소) | http://coashanee5.blogspot.com/2018/06/blog-post_25.html                 | unit09 후보 |
| 42  | **웹쉘을 이용한 공격 패러다임 변화 및 대응전략** (이글루코퍼레이션) | https://www.igloo.co.kr/security-information/웹쉘을-이용한-공격-패러다임-변화-및-대응전략/ | unit09 후보 |

> ⚠️ 방어 대책은 4개 축으로 정리한다 — **확장자 화이트리스트 / 파일 내용(시그니처) 검증 /
> 파일명 랜덤화 / 업로드 경로를 웹 루트 밖에 배치**. 확장자 블랙리스트만으로는 우회된다.

<br>

## 6. SSRF · 클릭재킹 · 오픈 리다이렉트 (4)

| #   | 제목                                        | 링크                                                                                             | 관련 UNIT   |
| --- | ------------------------------------------- | -------------------------------------------------------------------------------------------------- | ----------- |
| 43  | **SSRF 취약점을 이용한 공격사례 분석 및 대응방안** (이글루) | https://www.igloo.co.kr/security-information/ssrf-취약점을-이용한-공격사례-분석-및-대응방안/ | unit10 후보 |
| 44  | SSRF 공격의 동작 방식과 대처법 (ITWorld)    | https://www.itworld.co.kr/article/3553704/                                                        | unit10 후보 |
| 45  | 클릭재킹과 오픈 리다이렉트 공격 기법        | https://velog.io/@jerrychu/클릭재킹과-오픈-리다이렉트-공격-기법                                    | unit10 후보 |
| 46  | Open Redirect (버그바운티클럽 Pentest Gym)  | https://www.bugbountyclub.com/pentestgym/view/49                                                  | unit10 후보 |

> 💡 SSRF 방어의 핵심은 **허용 목록(Allowlist) 기반 검증**이다. 사용자가 URL 전체를 입력하게 하지
> 말고 미리 정해진 도메인/경로 집합만 선택하게 한다. 2019년 Capital One 침해(약 1억 600만 명)의
> 시작이 SSRF였다는 사례를 함께 넣으면 설득력이 커진다.

<br>

## 7. 세션 공격 (6)

| #   | 제목                                          | 링크                                                                                   | 관련 UNIT   |
| --- | --------------------------------------------- | ---------------------------------------------------------------------------------------- | ----------- |
| 47  | 세션 하이재킹이란 무엇이며, 어떻게 방어하나요? | https://velog.io/@ouk/세션-하이재킹Session-Hijacking이란-무엇이며-어떻게-방어하나요       | unit11 후보 |
| 48  | 세션 하이재킹 (MDN 용어 사전 한국어)          | https://developer.mozilla.org/ko/docs/Glossary/Session_Hijacking                         | unit11 후보 |
| 49  | 세션 고정 취약점                              | https://velog.io/@inmo/세션-고정-취약점                                                  | unit11 후보 |
| 50  | [WEB] 세션 고정 취약점 (정보보안 기록 저장소) | http://coashanee5.blogspot.com/2017/05/web.html                                          | unit11 후보 |
| 51  | 세션 하이재킹 (IT 위키)                       | https://itwiki.kr/w/세션_하이재킹                                                        | unit11 후보 |
| 52  | 브루트 포스 (나무위키)                        | https://namu.wiki/w/브루트%20포스                                                        | unit11 후보 |

> 💡 **세션 고정 방어의 핵심 한 줄**: 로그인 성공 시점에 반드시 세션 ID를 재발급한다.
> 이 원칙 하나가 unit11의 결론이 되어야 한다.

<br>

## 8. 비밀번호 안전한 저장 (4)

| #   | 제목                                    | 링크                                                                              | 관련 UNIT   |
| --- | --------------------------------------- | ----------------------------------------------------------------------------------- | ----------- |
| 53  | 단방향 암호화 아는척하기 (2) — BCrypt   | https://velog.io/@jh9/단방향-알고리즘-아는척하기2-BCrypt                            | unit12 후보 |
| 54  | 비밀번호 단방향 암호화에 대하여 (Medium) | https://pakss328.medium.com/비밀번호-단방향-암호화에-대하여-f2739a1485e             | unit12 후보 |
| 55  | bcrypt 비밀번호 해싱 (junyeokk)         | https://www.junyeokk.me/notes/security/bcrypt                                      | unit12 후보 |
| 56  | 패스워드 단방향 암호화(bcrypt)          | https://velog.io/@anjaekk/패스워드-단방향-암호화bcrypt                              | unit12 후보 |

> ⚠️ **"빠른 해시 함수는 비밀번호 저장에 부적합하다"** 가 unit12의 핵심 메시지다.
> SHA-256은 초당 수십억 회 계산이 가능해 무차별 대입에 취약하다. bcrypt는 **의도적으로 느리게**
> 설계되었고 **Salt + Key Stretching**을 내장한다. algorithm unit01(복잡도)과 반대 방향의
> 사고를 요구하는 지점이라 대비 서술이 효과적이다.

<br>

## 9. HTTPS 인증서 · CA 체인 (4)

| #   | 제목                                          | 링크                                                                        | 관련 UNIT   |
| --- | --------------------------------------------- | ----------------------------------------------------------------------------- | ----------- |
| 57  | HTTPS와 SSL 인증서, SSL 동작방법 (초보몽키)   | https://wayhome25.github.io/cs/2018/03/11/ssl-https/                         | unit13 후보 |
| 58  | ROOT CA 인증서는 무엇인가? (brunch)           | https://brunch.co.kr/@sangjinkang/47                                         | unit13 후보 |
| 59  | 브라우저 및 인증서 유효성 검사 (SSL.com 한국어) | https://www.ssl.com/ko/article/browsers-and-certificate-validation/         | unit13 후보 |
| 60  | CA(인증 기관)란 무엇입니까? (SSL.com 한국어)  | https://www.ssl.com/ko/기사/인증-기관-CA-란-무엇입니까%3F/                     | unit13 후보 |

> 💡 인증서 체인은 **ROOT → Intermediate → Leaf(서버)** 3단계다. network 과목의 TLS 핸드셰이크
> unit과 상호 참조하면 "HTTPS는 어떻게 안전한가"가 두 과목에 걸쳐 완성된다.

<br>

## 10. 시큐어 코딩 · 자격증 추가 (3)

| #   | 자료                                                | 링크                                                                                       | 관련 UNIT   |
| --- | --------------------------------------------------- | -------------------------------------------------------------------------------------------- | ----------- |
| 61  | **소프트웨어 개발 보안 — 시큐어 코딩 (LG CNS)**     | https://blog.lgcns.com/1152                                                                 | unit08      |
| 62  | [정보처리기사 실기] 9. 소프트웨어 개발 보안 구축    | https://velog.io/@ybseo/정보처리기사-실기-9.-소프트웨어-개발-보안-구축-시큐어-코딩-가이드보안-솔루션비즈니스-연속성-계획 | unit08 |
| 63  | [보.알.남] 시큐어 코딩, 안전한 소프트웨어 개발 위한 나침반 | https://m.boannews.com/html/detail.html?idx=112603                                     | unit08      |

> 💡 국내는 **2012년 12월부터 SW 개발 보안 의무제**가 시행되었고, 행정안전부 고시 기준
> **보안약점 47개 항목**이 공공 SI 사업의 준수 대상이다. unit08에서 이 제도적 배경을 짚으면
> "왜 시큐어 코딩이 필요한가"에 실무 근거가 생긴다.

<br>

## 11. 서적 · 로드맵 (3)

| #   | 자료                                          | 링크                                                              | 활용 포인트               |
| --- | --------------------------------------------- | ------------------------------------------------------------------- | ------------------------- |
| 64  | **면접을 위한 CS 전공지식 노트** (길벗/주홍철) | https://www.gilbut.co.kr/book/view?bookcode=BN003386               | 노트 서술 스타일 참고     |
| 65  | 면접을 위한 CS 전공지식 노트 — 예상질문       | https://thebook.io/080326/0144/                                    | unit별 확인 문제 설계     |
| 66  | TeachYourselfCS-KR (한국어 번역)              | https://github.com/minnsane/TeachYourselfCS-KR/blob/main/README.md | 보안·시스템 교재 로드맵   |
