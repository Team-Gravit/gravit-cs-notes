## 동기와 비동기 혼용

Django는 3.0에서 ASGI를, 3.1에서 비동기 뷰를, 4.1부터 비동기 ORM 인터페이스를 도입하며 **동기 코드와 비동기 코드가 한 프로젝트 안에 공존**하는 구조가 되었다. 문제는 그 경계다 — 비동기 뷰에서 동기 ORM을 부르면 예외가 나고, 동기 미들웨어 하나가 비동기 뷰의 이점을 지우며, 블로킹 호출 하나가 이벤트 루프 전체를 멈춘다. 이 유닛은 ASGI·async view·`sync_to_async`의 동작 원리와 혼용 규칙을 다룬다.

<br>

### 1. WSGI vs ASGI

| **항목**            | **WSGI**                                          | **ASGI**                                                       |
| ------------------- | ------------------------------------------------- | -------------------------------------------------------------- |
| **처리 모델**       | 요청 1개 = 워커 스레드/프로세스 1개, **동기**       | 이벤트 루프 위에서 **코루틴**으로 다수 요청 동시 처리             |
| **진입점**          | `wsgi.py` → `get_wsgi_application()`              | `asgi.py` → `get_asgi_application()`                           |
| **서버**            | gunicorn, uWSGI                                   | uvicorn, daphne, hypercorn, gunicorn + uvicorn 워커              |
| **프로토콜**        | HTTP만                                            | HTTP + **WebSocket** 등 장기 연결(channels)                      |
| **비동기 뷰**       | 동작은 하지만 요청마다 이벤트 루프를 새로 만들어 **이점 없음** | 네이티브 실행                                            |
| **동기 뷰**         | 네이티브 실행                                     | 스레드 풀에서 실행 (`sync_to_async`로 자동 변환)                 |

```
WSGI (gunicorn 워커 4개)              ASGI (uvicorn 워커 1개)
┌────┐┌────┐┌────┐┌────┐             ┌─────────────── 이벤트 루프 ───────────────┐
│req1││req2││req3││req4│ 대기: req5  │ req1 ⟲ req2 ⟲ req3 ⟲ req4 ⟲ req5 ⟲ ...    │
└────┘└────┘└────┘└────┘             │   (I/O 대기 중인 요청은 루프에 제어권 반납)  │
동시 처리 수 = 워커 수                └───────────┬────────────────────────────────┘
                                                  │ 동기 코드(ORM 등)는 스레드 풀로
                                                  ▼
                                      [스레드 ①][스레드 ②][스레드 ③] ...
```

- ASGI의 이점은 **I/O 대기 시간이 긴 요청**(외부 API 여러 개 호출, 장기 연결)에서 나타남
- CPU 위주 작업은 이벤트 루프를 점유하므로 비동기로 바꿔도 빨라지지 않음. 파이썬의 GIL 특성상 프로세스 수를 늘리는 것이 답

> 💡 "ASGI로 바꾸면 무조건 빨라지나요?"에 대한 답은 **아니다**. 요청 대부분이 DB 쿼리 몇 개로 끝나는 전형적인 CRUD API라면, Django 5.x의 ORM은 내부적으로 여전히 동기 드라이버를 스레드에서 실행하므로 처리량 차이가 거의 없다. 외부 HTTP 호출을 병렬화하거나 WebSocket이 필요할 때 비로소 의미가 있다.

<br>

### 2. 비동기 뷰

```python
import asyncio
import httpx
from django.http import JsonResponse


async def dashboard(request):
    async with httpx.AsyncClient(timeout=3) as client:
        weather, news = await asyncio.gather(          # 두 외부 호출을 동시에
            client.get("https://api.example.com/weather"),
            client.get("https://api.example.com/news"),
        )
    return JsonResponse({"weather": weather.json(), "news": news.json()})
```

- `async def`로 선언하면 Django가 코루틴 함수로 인식해 ASGI 핸들러에서 직접 `await`함
- 클래스 기반 뷰는 `get`·`post` 등 **핸들러 메서드가 모두 async**여야 함. 섞으면 `ImproperlyConfigured`
- `asyncio.gather`로 **독립적인 I/O를 병렬화**하는 것이 비동기 뷰의 대표적 용도

❗️**블로킹 호출은 루프 전체를 멈춘다**: 비동기 뷰 안에서 `requests.get()`·`time.sleep()`·동기 파일 I/O를 호출하면 그 워커가 처리 중인 **모든 요청**이 멈춘다. 동기 뷰의 블로킹은 그 스레드 하나만 막지만, 비동기 뷰의 블로킹은 이벤트 루프를 독점하기 때문이다.

<br>

### 3. ORM과 SynchronousOnlyOperation

Django ORM은 비동기 컨텍스트(이벤트 루프가 실행 중인 스레드)에서 **동기 쿼리 실행을 감지하면 `SynchronousOnlyOperation`을 던진다**. 커넥션이 스레드에 묶여 있고 동기 드라이버가 루프를 블로킹하기 때문이다.

```python
# 안티패턴: 비동기 뷰에서 동기 ORM 평가
async def post_list(request):
    posts = list(Post.objects.all())      # SynchronousOnlyOperation!
    ...

# 개선 ①: 비동기 ORM 인터페이스 (Django 4.1+)
async def post_list(request):
    posts = [p async for p in Post.objects.select_related("author")]
    total = await Post.objects.acount()
    first = await Post.objects.afirst()
    ...

# 개선 ②: 동기 함수를 스레드로 감싸기
from asgiref.sync import sync_to_async

@sync_to_async
def load_posts():
    return list(Post.objects.select_related("author"))   # 여기서는 동기 ORM 정상

async def post_list(request):
    posts = await load_posts()
    ...
```

| **동기 API**                 | **비동기 API (4.1+)**                | **비고**                                        |
| ---------------------------- | ------------------------------------ | ----------------------------------------------- |
| `for p in qs`                | `async for p in qs`                  | 지연 평가 규칙은 동일 (unit02)                   |
| `qs.get()` / `.first()`      | `await qs.aget()` / `.afirst()`      |                                                 |
| `qs.count()` / `.exists()`   | `await qs.acount()` / `.aexists()`   |                                                 |
| `Model.objects.create()`     | `await Model.objects.acreate()`      |                                                 |
| `obj.save()` / `.delete()`   | `await obj.asave()` / `.adelete()`   | 4.2+                                            |
| `obj.author` (FK 지연 로딩)  | **비동기 버전 없음**                  | `select_related`로 미리 로드하거나 `sync_to_async`로 감쌈       |
| `transaction.atomic()`       | **비동기 버전 없음**                  | 트랜잭션 블록 전체를 `sync_to_async` 함수로 감쌈   |

- `a` 접두어 메서드는 내부적으로 `sync_to_async`로 동기 쿼리를 스레드에서 실행하는 **얇은 래퍼**다. Django 5.x 기준으로 진정한 비동기 DB 드라이버는 아직 아님 → 성능 이점보다 **문법 호환**의 의미가 큼
- `select_related`로 로드되지 않은 FK에 비동기 뷰에서 접근하면 지연 로딩 쿼리가 동기로 실행되어 예외가 남 → 필요한 관계는 **미리 로드**
- 트랜잭션은 커넥션·스레드에 묶이므로, **원자적이어야 하는 쿼리 묶음은 하나의 동기 함수로 만들어 `sync_to_async`로 감싼다**

> ⚠️ `DJANGO_ALLOW_ASYNC_UNSAFE=true` 환경 변수는 이 검사를 끈다. Jupyter처럼 이벤트 루프가 항상 떠 있는 환경을 위한 우회이며, **서버 코드에서 켜면 예외 대신 조용한 데이터 손상·루프 블로킹**으로 바뀔 뿐이다.

<br>

### 4. sync_to_async와 async_to_sync

두 세계를 잇는 어댑터는 `asgiref`가 제공한다.

| **함수**                                 | **방향**                       | **동작**                                                          |
| ---------------------------------------- | ------------------------------ | ----------------------------------------------------------------- |
| **`sync_to_async(fn)`**                  | 비동기 코드에서 동기 함수 호출 | 함수를 **스레드**에서 실행하고 코루틴으로 감쌈                     |
| **`async_to_sync(coro_fn)`**             | 동기 코드에서 비동기 함수 호출 | 새 이벤트 루프(또는 별도 스레드의 루프)에서 실행하고 결과 반환      |

**`thread_sensitive` 인자**

- `sync_to_async(fn, thread_sensitive=True)`(기본값): 같은 요청 안의 thread-sensitive 호출은 **하나의 스레드에서 순차 실행**됨. ORM 커넥션·트랜잭션처럼 스레드에 묶인 상태를 안전하게 쓰기 위한 기본값
- `thread_sensitive=False`: 스레드 풀의 아무 스레드에서나 실행됨. 순수 계산·스레드 상태와 무관한 동기 함수에만 사용

```python
from asgiref.sync import sync_to_async, async_to_sync


# 비동기 뷰에서 동기 서비스 함수(트랜잭션 포함) 호출
async def place_order_view(request):
    order = await sync_to_async(place_order)(request.user, product_id=1, qty=2)
    return JsonResponse({"id": order.id})


# 동기 코드(예: Celery 태스크)에서 비동기 클라이언트 사용
def sync_task():
    result = async_to_sync(fetch_all)(urls)
```

- `async_to_sync`는 **이미 이벤트 루프가 실행 중인 스레드에서 호출하면 `RuntimeError`** 가 남 — 비동기 뷰 안에서 "동기 함수 → 그 안에서 async_to_sync"처럼 중첩되는 구조를 만들지 않아야 함
- 요청당 동기 코드가 많다면 결국 스레드 풀에서 실행되므로, 스레드 풀 크기(기본 `min(32, cpu+4)`)가 동시 처리 한계가 됨

<br>

### 5. 미들웨어·DRF·시그널과의 혼용

```
ASGI 서버 ─▶ [async 미들웨어] ─▶ [sync 미들웨어] ─▶ [async 미들웨어] ─▶ async 뷰
                                    ▲ 스레드 전환                ▲ 다시 루프로 전환
                                    (요청·응답 단계 각각 1회씩 비용)
```

- Django는 미들웨어 체인에서 동기·비동기 경계가 바뀔 때마다 **자동으로 스레드 전환(adapt)** 함. 동작은 하지만 전환마다 비용이 들고, 동기 미들웨어가 하나라도 있으면 그 지점에서 요청이 스레드로 넘어감 (unit01 참고)
- 동기 전용 미들웨어(`async_capable = False`)와 비동기 뷰를 함께 쓰면 뷰의 이점이 대부분 사라짐. 비동기 경로를 진지하게 쓰려면 미들웨어 스택 전체를 `async_capable`로 맞춰야 함
- **DRF의 `APIView`·`ViewSet`은 동기 전제**다. `async def` 핸들러를 넣어도 정상 동작하지 않으며, 인증·권한·스로틀 클래스도 동기로 실행됨. 비동기 뷰가 필요하면 순수 Django 뷰를 쓰거나 서드파티 `adrf` 패키지를 사용함(공식 DRF의 비동기 지원 여부는 버전에 따라 확인 필요)
- 시그널: Django 5.0부터 `Signal.asend()`로 비동기 수신자를 호출할 수 있음. 동기 수신자와 섞이면 각각 스레드/루프로 적응되어 실행됨 (unit08 참고)
- 인증: Django 5.0의 `aauthenticate()`·`alogin()`, 5.1의 비동기 `login_required`·세션 비동기 API 등으로 인증 경로의 비동기 지원이 점차 확대되고 있음 (unit06 참고). 사용하는 버전의 릴리스 노트로 지원 범위를 확인할 것

> 💡 실무에서 가장 현실적인 구성은 "**동기 Django + WSGI를 기본으로 두고**, 외부 API 병렬 호출·스트리밍·WebSocket처럼 이점이 분명한 엔드포인트만 ASGI 비동기 뷰로 분리"하는 것이다. 전체를 비동기로 전환하는 것은 미들웨어·DRF·서드파티 전부를 검토해야 하는 큰 작업이다.

<br>

### 6. 혼용 규칙 요약

| **상황**                                     | **규칙**                                                                          |
| -------------------------------------------- | --------------------------------------------------------------------------------- |
| 비동기 뷰에서 ORM                            | `a` 접두어 메서드·`async for` 사용, 관련 객체는 **미리 로드**                       |
| 비동기 뷰에서 트랜잭션                       | 트랜잭션 블록을 **동기 함수로 묶고 `sync_to_async`**                                |
| 비동기 뷰에서 외부 HTTP                      | `httpx.AsyncClient`·`aiohttp` 등 **비동기 클라이언트**. `requests`는 금지           |
| 동기 코드에서 비동기 함수                    | `async_to_sync`, 단 **실행 중인 루프 안에서는 호출 불가**                           |
| 동기 함수를 비동기에서 호출                  | `sync_to_async`, 스레드 상태(ORM)가 있으면 `thread_sensitive=True`(기본값)          |
| 미들웨어                                     | 비동기 경로를 쓰면 스택 전체를 `async_capable=True`로                               |
| DRF                                          | 동기 유지, 비동기가 꼭 필요하면 순수 Django 뷰 또는 `adrf`                           |
| CPU 위주 작업                                | 비동기로 바꾸지 말고 **워커 프로세스 증설·태스크 큐**로 분리                         |

<br>

### 7. 면접·실무 체크포인트

- **WSGI와 ASGI의 차이?** WSGI는 요청당 스레드/프로세스의 동기 모델, ASGI는 이벤트 루프 위 코루틴으로 다수 요청을 동시에 처리하며 WebSocket 등 장기 연결 지원
- **비동기 뷰에서 `Post.objects.all()`을 순회하면?** 이벤트 루프 스레드에서 동기 쿼리 → **`SynchronousOnlyOperation`**. `async for`·`aget()` 또는 `sync_to_async`로 해결
- **Django의 비동기 ORM은 진짜 비동기인가?** 5.x 기준 **`sync_to_async` 래퍼**. 문법은 비동기지만 실제 쿼리는 스레드에서 동기 실행됨
- **`thread_sensitive=True`의 의미?** 같은 요청의 동기 호출을 **하나의 스레드에서 순차 실행**해 커넥션·트랜잭션 등 스레드 종속 상태를 보호
- **비동기 뷰에서 `requests.get()`을 쓰면?** 이벤트 루프가 블로킹되어 **그 워커의 모든 요청이 멈춤**
- **ASGI로 바꾸면 성능이 오르나?** I/O 대기가 긴 요청만. DB 위주 CRUD·CPU 작업은 거의 차이 없음
- **DRF에서 async 뷰를 쓸 수 있나?** 기본 `APIView`는 동기 전제. 필요하면 순수 Django 비동기 뷰나 `adrf` 사용
