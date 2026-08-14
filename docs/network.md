# network — 레퍼런스 모음

`network/unit01 ~ unit14` 개선 및 신규 unit 작성에 활용할 외부 자료 **66건** (1차 36 + 2차 30).

<br>

## 1. 기술 블로그 · 위키 (24)

### 1-1. 계층 모델 · TCP/UDP

| #   | 제목                                        | 링크                                                                          | 관련 UNIT   |
| --- | ------------------------------------------- | ------------------------------------------------------------------------------- | ----------- |
| 1   | OSI 7계층 (goodgid)                         | https://goodgid.github.io/OSI-7-Layer/                                         | unit01      |
| 2   | OSI 7계층과 TCP/IP 4계층의 차이 (dev-letter) | https://dev-letter.kr/posts/osi-7-layer                                       | unit01, 02  |
| 3   | OSI 7계층의 네트워크·전송·애플리케이션 계층 | https://haon.blog/haon/network/network-transport-application/                  | unit01, 02  |
| 4   | TCP/UDP와 3-Way / 4-Way Handshake           | https://velog.io/@averycode/네트워크-TCPUDP와-3-Way-Handshake4-Way-Handshake    | TCP unit    |
| 5   | 3-way / 4-way 핸드셰이크 (네트워크 완전정복) | https://wikidocs.net/369170                                                   | TCP unit    |
| 6   | 개발자를 위한 네트워크 기초 완전 정복       | https://www.youngju.dev/blog/culture/2026-03-23-networking-fundamentals-developer-guide | 전체 |

### 1-2. IP · 라우팅

| #   | 제목                                       | 링크                                                                     | 관련 UNIT  |
| --- | ------------------------------------------ | -------------------------------------------------------------------------- | ---------- |
| 7   | IP 고갈 문제 (서브넷팅·NAT·DHCP)           | https://rokwonk.github.io/cs/네트워크-IP-고갈문제와-해결책/                | IP unit    |
| 8   | 네트워크와 IP 주소 — 홉바이홉·라우팅       | https://velog.io/@lnh03280/네트워크와-IP-주소-이해하기-쉽게-정리한-필수-개념 | IP unit   |
| 9   | Subnetting (Catsbi's DLog)                 | https://catsbi.oopy.io/3555f477-30c0-46c5-9e13-0bb3b9ba8ea2               | IP unit    |

### 1-3. HTTP · HTTPS · DNS

| #   | 제목                                      | 링크                                                                       | 관련 UNIT   |
| --- | ----------------------------------------- | ---------------------------------------------------------------------------- | ----------- |
| 10  | HTTP/1.0, 1.1, 2.0, 3.0 그리고 QUIC       | https://velog.io/@minu/HTTP1.0-HTTP1.1-HTTP2-and-QUIC                        | HTTP unit   |
| 11  | **HTTP/3 explained (한국어)** — H3 vs H2  | https://http3-explained.haxx.se/ko/h3/h3-h2                                 | HTTP unit   |
| 12  | QUIC과 HTTP/3 — 기존 성능 개선 기법과 한계 | https://www.saturnsoft.net/network/2019/03/26/quic-http3-2/                 | HTTP unit   |
| 13  | HTTP/3 (나무위키)                         | https://namu.wiki/w/HTTP/3                                                   | HTTP unit   |
| 14  | HTTP 완벽 가이드 4장 — 커넥션 관리        | https://velog.io/@bahar-j/HTTP-완벽-가이드-4장-커넥션-관리                   | HTTP unit   |
| 15  | DNS, TCP/IP 와 TLS(SSL) 핸드셰이크        | https://velog.io/@taehyung/DNS-TCPIP-와-TLSSSL-핸드셰이크                    | DNS/TLS unit |
| 16  | TLS 용어 해설 (토스페이먼츠 개발자센터)   | https://docs.tosspayments.com/resources/glossary/tls                        | TLS unit    |
| 17  | TLS (나무위키) — 버전별 변화              | https://namu.wiki/w/TLS                                                      | TLS unit    |
| 18  | REST API 관점에서 바라보는 HTTP 상태 코드 | https://library.gabia.com/contents/8346/                                    | HTTP unit   |
| 19  | REST API 모범 사례 (freeCodeCamp 한국어)  | https://www.freecodecamp.org/korean/news/rest-api-mobeom-sarye-rest-endeupointeu-seolgye-yesi/ | REST unit |

### 1-4. 세션 · 캐시 · 부하 분산

| #   | 제목                                     | 링크                                                                        | 관련 UNIT     |
| --- | ---------------------------------------- | ----------------------------------------------------------------------------- | ------------- |
| 20  | 쿠키 2부: 세션은 쿠키가 필요해 (Dale Seo) | https://daleseo.com/http-session/                                            | 쿠키/세션 unit |
| 21  | 쿠키/세션/토큰/캐시/CDN (brunch)         | https://brunch.co.kr/@danni/5                                                | 쿠키/세션 unit |
| 22  | 완벽 정리! 쿠키, 세션, 토큰, 캐시 그리고 CDN | https://hongong.hanbit.co.kr/완벽-정리-쿠키-세션-토큰-캐시-그리고-cdn/     | 쿠키/세션 unit |
| 23  | NHN Cloud 로드 밸런서 개요 (공식 문서)   | https://docs.nhncloud.com/ko/Network/Load%20Balancer/ko/overview/            | 부하분산 unit |
| 24  | 웹소켓과 REST API의 차이점 및 적용 시나리오 | https://f-lab.kr/insight/websocket-vs-rest-api                             | unit15 후보   |

> 💡 unit15 이후 신규 unit 후보: **웹소켓과 실시간 통신**, **로드밸런싱·프록시·CDN**,
> **HTTP/3와 QUIC**. 세 주제 모두 unit01~14의 TCP/HTTP 지식을 전제로 자연스럽게 이어진다.

<br>

## 2. 서적 (5)

| #   | 도서명                              | 출판사/저자        | 링크                                                              | 활용 포인트                       |
| --- | ----------------------------------- | ------------------ | ------------------------------------------------------------------- | --------------------------------- |
| 25  | **모두의 네트워크**                 | 길벗               | https://www.gilbut.co.kr/book/view?bookcode=BN002021               | unit01 기초 개념 서술 구조 참고   |
| 26  | 한 권으로 끝내는 네트워크 기초      | 길벗               | https://www.gilbut.co.kr/book/view?bookcode=BN003412               | 도해 133개 — 도식 구성 참고       |
| 27  | 그림으로 배우는 HTTP & Network Basic (요약) | 우에노 센 | https://lumiloves.github.io/2018/01/07/book-http-network-basic      | HTTP unit 목차 설계               |
| 28  | 그림으로 배우는 HTTP & Network Basic (연재 정리) | hudi.blog | https://hudi.blog/series/그림으로-배우는-http-&-network-basic/  | 장별 핵심 요약                    |
| 29  | 그림으로 배우는 HTTP & Network Basic (서지) | Goodreads   | https://www.goodreads.com/book/show/36543211-http-network-basic     | 원서 정보 확인                    |

<br>

## 3. 자격증 (5)

| #   | 자료                                              | 링크                                                                              | 활용 포인트             |
| --- | ------------------------------------------------- | ----------------------------------------------------------------------------------- | ----------------------- |
| 30  | 정보처리기사 네트워크 정리 (velog)                | https://velog.io/@kwonja/정보처리기사-네트워크-정리                                 | 프로토콜 용어 정의      |
| 31  | 정보처리기사 실기 — 서브넷 마스크 기출 문제 풀이  | https://velog.io/@sssdoooy/정보처리기사-실기-프로그래밍-기출-서브넷-서브넷-마스크-요약-정리 | 서브넷 계산 예제  |
| 32  | 시나공 — 정보처리기사 필기 기출문제               | https://www.sinagong.co.kr/pds/001001001/past-exams                                | OSI 계층 기출 패턴      |
| 33  | 전자문제집 CBT — 정보처리기사                     | https://www.comcbt.com/xe/iz                                                       | 반복 학습               |
| 34  | 정보보안기사 요약 — 네트워크 보안 파트 (JMoon)    | https://jmoon.co.kr/23                                                             | 보안 관점 프로토콜 정리 |

> ⚠️ 정보처리기사에서는 **서브넷 마스크 계산**과 **OSI 7계층별 프로토콜/장비 매칭**이 계산·암기
> 문제로 반복 출제된다. 해당 unit에 계산 예제와 계층별 매핑 표를 반드시 넣을 것.

<br>

## 4. 참고 사이트 (2)

| #   | 자료                                   | 링크                                                  | 활용 포인트                     |
| --- | -------------------------------------- | ------------------------------------------------------- | ------------------------------- |
| 35  | Cloudflare Learning — HTTP/2 vs HTTP/1.1 | https://www.cloudflare.com/learning/performance/http2-vs-http1.1/ | 영문 공식 설명·도식 |
| 36  | MDN Web Docs (HTTP 레퍼런스)           | https://developer.mozilla.org/ko/docs/Web/HTTP         | 헤더·상태코드 정확한 정의       |

<br>

---

<br>

# 2차 수집분 (30건)

1차에서 빠진 **TCP 제어 기법 · 하위 프로토콜 · HTTP 캐시 · 소켓 · 네트워크 장비**를 보강했다.

<br>

## 5. TCP 흐름제어 · 혼잡제어 (4)

| #   | 제목                                | 링크                                                                | 관련 UNIT |
| --- | ----------------------------------- | --------------------------------------------------------------------- | --------- |
| 37  | TCP (흐름제어 / 혼잡제어 / 오류제어) | https://velog.io/@jsj3282/TCP-흐름제어혼잡제어-오류제어              | TCP unit  |
| 38  | TCP 흐름제어 기법 살펴보기          | https://velog.io/@haero_kim/TCP-흐름제어-기법-살펴보기               | TCP unit  |
| 39  | [TCP] 흐름제어와 혼잡제어 (+ 오류제어) | https://velog.io/@itonse/TCP-흐름제어와-혼잡제어-에러제어           | TCP unit  |
| 40  | TCP의 흐름제어·오류제어·혼잡제어 개요 | https://roka88.dev/114                                              | TCP unit  |

> ⚠️ TCP unit이 3-way handshake에서 끝나면 절반만 다룬 것이다. **슬라이딩 윈도우(흐름제어)** 와
> **AIMD·슬로우 스타트(혼잡제어)** 까지 넣어야 "TCP는 신뢰성 있는 프로토콜"이라는 서술이 완성된다.

<br>

## 6. 하위 프로토콜 — ARP · DHCP (5)

| #   | 제목                                | 링크                                                                    | 관련 UNIT |
| --- | ----------------------------------- | ------------------------------------------------------------------------- | --------- |
| 41  | **DHCP 상세 동작 원리 (NETMANIAS)** | https://netmanias.com/ko/?id=techdocs&m=view&no=5180                     | IP unit   |
| 42  | DHCP 프로토콜 기본 원리 (NETMANIAS) | https://www.netmanias.com/ko/post/blog/5348/dhcp-ip-allocation-network-protocol/understanding-the-basic-operations-of-dhcp | IP unit |
| 43  | 주소 결정 프로토콜 ARP (위키백과)   | https://ko.wikipedia.org/wiki/주소_결정_프로토콜                          | IP unit   |
| 44  | 동적 호스트 구성 프로토콜 DHCP (위키백과) | https://ko.wikipedia.org/wiki/동적_호스트_구성_프로토콜               | IP unit   |
| 45  | IP / ARP / Port / TCP / DHCP 아키텍처 흐름 | https://velog.io/@wnss1575/IP-ARP-Port-TCP-DHCP                     | unit01~03 |

> 💡 NETMANIAS는 국내 네트워크 전문 기술 문서 사이트로, **패킷 단위 도식이 정확**하다.
> 노트 도식을 새로 그릴 때 참고 원본으로 삼기에 가장 적합하다. DHCP의 **DORA(Discover-Offer-
> Request-Acknowledge)** 4단계는 정보처리기사 빈출이다.

<br>

## 7. HTTP 캐시 (4)

| #   | 제목                                          | 링크                                                                             | 관련 UNIT   |
| --- | --------------------------------------------- | ---------------------------------------------------------------------------------- | ----------- |
| 46  | **Cache-Control (MDN 한국어 공식)**           | https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Headers/Cache-Control     | HTTP unit   |
| 47  | HTTP 캐시 (Cache-Control, 조건부 요청) — hudi | https://hudi.blog/http-cache/                                                     | HTTP unit   |
| 48  | 알아둬야 할 HTTP 쿠키 & 캐시 헤더 (ZeroCho)   | https://www.zerocho.com/category/HTTP/post/5b594dd3c06fa2001b89feb9                | HTTP unit   |
| 49  | HTTP 캐싱 이해하기 — Last-Modified, ETag      | https://www.yolog.co.kr/post/http-cache/                                          | HTTP unit   |

<br>

## 8. 소켓 프로그래밍 (3)

| #   | 제목                                | 링크                                                            | 관련 UNIT   |
| --- | ----------------------------------- | ----------------------------------------------------------------- | ----------- |
| 50  | 네트워크 소켓 프로그래밍 정리       | https://losskatsu.github.io/os-kernel/network-socket/            | unit17 후보 |
| 51  | TCP 기반 소켓 프로그래밍            | https://velog.io/@white-jelly/TCP-기반-프로그래밍                 | unit17 후보 |
| 52  | [소켓 프로그래밍] 블로킹 소켓       | https://velog.io/@jinh2352/소켓-프로그래밍-블로킹-소켓            | unit17 후보 |

<br>

## 9. 네트워크 장비 · 보안 장비 (7)

| #   | 제목                                        | 링크                                                                | 관련 UNIT   |
| --- | ------------------------------------------- | --------------------------------------------------------------------- | ----------- |
| 53  | 네트워크 스위치 L2, L3, L4, L7 정리         | https://velog.io/@younge/네트워크-스위치-L2-L3-L4-L7-정리             | 장비 unit   |
| 54  | 스위치의 개념과 종류 (L2, L3, L4, L7)       | https://velog.io/@zxcvbnm5288/스위치의-개념과-스위치의-종류L2-L3-L4-L7 | 장비 unit   |
| 55  | 스위치(네트워크) — 나무위키                 | https://namu.wiki/w/스위치(네트워크)                                  | 장비 unit   |
| 56  | 네트워크 보안장비 (방화벽, IDS, IPS, UTM)   | https://blog.1nfra.kr/194                                            | 보안 unit   |
| 57  | 방화벽 / IDS / IPS 정의 및 차이             | https://velog.io/@jungwoo343/방화벽IDSIPS-정의-및-차이                | 보안 unit   |
| 58  | IPS, IDS 그리고 방화벽(Firewall) (brunch)   | https://brunch.co.kr/@sangjinkang/49                                 | 보안 unit   |
| 59  | IDS / IPS (ITPE JackerLab)                  | https://itpe.jackerlab.com/entry/IDSIPS                              | 보안 unit   |

> 💡 **L2(MAC) → L3(IP) → L4(포트) → L7(URL·콘텐츠)** 로 이어지는 스위치 계층 구분은
> unit01(OSI 7계층)의 실물 대응 예시로 쓰기에 가장 좋다.

<br>

## 10. 종합 — "URL 입력 시 일어나는 일" (4)

전 unit의 지식을 하나로 꿰는 대표 시나리오다. 마지막 unit 또는 unit01 도입부에 배치하면 효과적이다.

| #   | 제목                                             | 링크                                                                                  | 관련 UNIT |
| --- | ------------------------------------------------ | --------------------------------------------------------------------------------------- | --------- |
| 60  | **웹 브라우저에 URL을 입력하면? (AWS 한국 블로그)** | https://aws.amazon.com/ko/blogs/korea/what-happens-when-you-type-a-url-into-your-browser/ | 종합 |
| 61  | 브라우저에 url을 입력하면 어떤 일이 벌어질까?    | https://velog.io/@khy226/브라우저에-url을-입력하면-어떤일이-벌어질까                    | 종합      |
| 62  | 브라우저에 URL을 입력하면 일어나는 일 (nokiahub) | https://www.nokiahub.name/posts/how-browser-works                                      | 종합      |
| 63  | [기술 면접] 브라우저에 URL을 입력하면 일어나는 일 | https://velog.io/@jjh0526/기술-면접-준비-네트워크-브라우저에-URL을-입력하면-일어나는-일 | 종합      |

<br>

## 11. 서적 · 로드맵 추가 (3)

| #   | 자료                                          | 링크                                                              | 활용 포인트                |
| --- | --------------------------------------------- | ------------------------------------------------------------------- | -------------------------- |
| 64  | **면접을 위한 CS 전공지식 노트** (길벗/주홍철) | https://www.gilbut.co.kr/book/view?bookcode=BN003386               | 네트워크 챕터 목차 참고    |
| 65  | 면접을 위한 CS 전공지식 노트 — 온라인 열람    | https://thebook.io/080326/                                         | 설명 방식·도식 참고        |
| 66  | TeachYourselfCS-KR (한국어 번역)              | https://github.com/minnsane/TeachYourselfCS-KR/blob/main/README.md | 표준 네트워크 교재 목록    |
