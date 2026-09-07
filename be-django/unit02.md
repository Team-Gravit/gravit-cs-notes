## QuerySet 지연 평가

Django ORM의 **QuerySet**은 만들어지는 순간 SQL을 실행하지 않고, 실제로 데이터가 필요한 시점까지 실행을 미루는 **지연 평가(Lazy Evaluation)** 를 따른다. "쿼리가 언제 실행되는가"와 "결과가 어디에 캐시되는가"를 정확히 알아야 불필요한 중복 쿼리와 메모리 낭비를 피할 수 있다.

<br>

### 1. 지연 평가란

QuerySet은 "어떤 데이터를 가져올지"를 담은 **쿼리 설명서(Query 객체)** 이지, 데이터 그 자체가 아니다. `filter()`·`exclude()`·`order_by()` 같은 메서드는 새 QuerySet을 반환할 뿐 DB에 접근하지 않는다.

```python
qs = Post.objects.filter(is_published=True)   # 쿼리 실행 안 됨
qs = qs.exclude(author__is_staff=True)        # 여전히 실행 안 됨
qs = qs.order_by("-created_at")               # 여전히 실행 안 됨

for post in qs:                               # 여기서 SELECT 1회 실행
    print(post.title)
```

- 메서드 체이닝마다 **새 QuerySet 객체**가 만들어지며, 원본은 변경되지 않음 (불변에 가까운 설계)
- 조건을 조립하는 비용은 파이썬 객체 생성 비용뿐이고, **DB 왕복은 평가 시점에 단 한 번** 발생함
- 덕분에 뷰·서비스 계층에서 조건을 나눠 붙이거나, 조건부로 `filter`를 추가하는 코드를 자연스럽게 쓸 수 있음

> 💡 `str(qs.query)`를 출력하면 실행 없이 생성될 SQL을 확인할 수 있다. 다만 파라미터가 인용 없이 치환되므로 디버깅 용도로만 사용한다.

<br>

### 2. 쿼리가 실제로 실행되는 시점

다음 동작이 일어나는 순간 QuerySet이 **평가(evaluate)** 되어 SQL이 실행된다.

| **트리거**                | **예시**                                | **설명**                                                        |
| ------------------------- | --------------------------------------- | --------------------------------------------------------------- |
| **반복(iteration)**       | `for p in qs`, `list(qs)`               | 전체 결과를 가져와 캐시에 저장                                  |
| **step 슬라이싱**         | `qs[::2]`                               | 파이썬 슬라이싱 규칙상 전체 조회 후 잘라냄                       |
| **인덱스 접근**           | `qs[0]`                                 | `LIMIT 1` 쿼리 실행. **캐시에 저장되지 않음**                   |
| **`len()`**               | `len(qs)`                               | 전체 조회 후 파이썬에서 길이 계산                               |
| **`bool()`**              | `if qs:`                                | 전체 조회 (첫 행만 필요해도 전체를 가져옴)                      |
| **`repr()`**              | 셸에서 `qs` 출력                         | 앞 21개를 가져와 출력                                           |
| **`pickle`·캐싱**         | `cache.set("k", qs)`                    | 직렬화를 위해 전체 조회                                         |
| **집계·단건 메서드**      | `count()`, `exists()`, `get()`, `first()` | 즉시 별도 쿼리 실행. **캐시를 만들지 않음**                    |

반면 **step 없는 슬라이싱**(`qs[:10]`)은 `LIMIT`·`OFFSET`이 붙은 **새 QuerySet**을 반환할 뿐 평가하지 않는다.

```
Post.objects.filter(...)      # QuerySet 생성 ── 실행 X
   .order_by("-id")            # 새 QuerySet   ── 실행 X
   [:10]                       # LIMIT 10 추가 ── 실행 X
   ─────────────────────────────────────────────
   list(...) / for ... in     # 평가 ────────── SELECT 실행, 결과 캐시
```

<br>

### 3. 결과 캐시(Result Cache)

QuerySet은 평가되면 결과를 내부 속성 `_result_cache`에 **리스트로 보관**한다. 같은 QuerySet 객체를 다시 순회하면 캐시를 재사용해 DB에 가지 않는다.

<br>

### 3-1. 캐시가 재사용되는 경우

```python
posts = Post.objects.filter(is_published=True)

titles = [p.title for p in posts]    # 쿼리 1회 실행, 캐시 채움
authors = [p.author_id for p in posts]  # 캐시 재사용, 쿼리 없음
print(len(posts))                    # 캐시 재사용, 쿼리 없음
```

<br>

### 3-2. 캐시가 재사용되지 않는 경우 (흔한 함정)

```python
# 안티패턴 ①: 인덱스 접근은 매번 LIMIT 1 쿼리
qs = Post.objects.all()
first = qs[0]      # SELECT ... LIMIT 1
second = qs[1]     # SELECT ... LIMIT 1 OFFSET 1  (또 실행)

# 개선: 한 번 평가한 뒤 캐시에서 꺼냄
posts = list(Post.objects.all()[:2])
first, second = posts[0], posts[1]
```

```python
# 안티패턴 ②: 존재 확인 + 순회를 다른 방식으로 두 번
qs = Post.objects.filter(author=user)
if qs.exists():          # SELECT ... LIMIT 1  (캐시 없음)
    for p in qs:         # SELECT ... (전체 다시 조회)
        ...

# 개선 A: 결과를 어차피 쓸 거라면 bool()로 평가해 캐시를 공유
posts = Post.objects.filter(author=user)
if posts:                # 전체 조회 1회, 캐시 채움
    for p in posts:      # 캐시 재사용
        ...

# 개선 B: 존재 여부만 필요하면 exists()만 사용
if Post.objects.filter(author=user).exists():
    ...
```

- `count()`·`exists()`는 이미 캐시가 있으면 캐시를 이용하지만, **캐시가 없으면 별도 쿼리**를 날리고 캐시를 만들지 않음
- `filter()`처럼 새 QuerySet을 만들면 캐시는 **공유되지 않음** — 새 객체이므로 처음부터 다시 평가됨

> ⚠️ 템플릿에서 `{% if posts %}` 뒤에 `{% for p in posts %}`를 쓰는 것은 같은 객체를 쓰므로 쿼리 1회지만, `{{ posts.count }}`를 먼저 쓰면 `COUNT` 쿼리 1회 + 순회 쿼리 1회로 **2회**가 된다. 개수와 목록이 모두 필요하면 먼저 순회(또는 `len()`)로 평가한 뒤 `len`을 쓰는 것이 유리하다.

<br>

### 4. 관련 객체 캐시와 QuerySet 캐시의 차이

QuerySet 결과 캐시와 **모델 인스턴스의 관련 객체 캐시**는 별개다.

```python
post = Post.objects.get(pk=1)
post.author        # SELECT author ... (1회)
post.author        # 인스턴스 속성에 캐시됨 → 쿼리 없음

post.comments.all()    # 새 QuerySet 생성 (역참조 매니저)
post.comments.all()    # 또 새 QuerySet → 순회하면 매번 쿼리
```

- 정참조(`post.author`)는 한 번 접근하면 **인스턴스에 캐시**됨
- 역참조·다대다 매니저(`post.comments`)는 `all()`을 호출할 때마다 **새 QuerySet**이므로 캐시가 없음. 반복문 안에서 호출하면 N+1 문제로 이어짐 (unit03 참고)
- `prefetch_related`로 미리 가져오면 `post.comments.all()`이 prefetch 캐시를 사용해 쿼리를 내지 않음. 단, 그 뒤에 `.filter()`를 붙이면 캐시를 버리고 새 쿼리를 실행함

<br>

### 5. 대량 데이터와 `iterator()`

평가된 결과는 전부 메모리에 올라간다. 수십만 행을 순회해야 한다면 `iterator()`로 **캐시 없이 스트리밍**한다.

```python
# 안티패턴: 100만 행을 리스트로 전부 적재
for row in BigLog.objects.all():
    export(row)

# 개선: 서버 사이드 커서 + 청크 단위로 가져오며 캐시를 만들지 않음
for row in BigLog.objects.all().iterator(chunk_size=2000):
    export(row)
```

| **항목**            | **일반 평가 (`for p in qs`)**       | **`iterator(chunk_size=N)`**                  |
| ------------------- | ----------------------------------- | --------------------------------------------- |
| **결과 캐시**       | `_result_cache`에 전부 보관         | **보관 안 함** (재순회 시 다시 쿼리)          |
| **메모리**          | 전체 행 크기에 비례                 | 청크 크기에 비례                              |
| **재사용**          | 여러 번 순회 가능                   | 한 번만 순회                                  |
| **prefetch_related** | 정상 동작                          | Django 5.0부터 **`chunk_size` 필수**(미지정 시 오류) |

- PostgreSQL·Oracle은 서버 사이드 커서를 사용하고, MySQL·SQLite는 드라이버가 전체를 가져온 뒤 파이썬 쪽에서만 청크로 나누므로 DB 드라이버 수준의 메모리 절감 효과는 DB마다 다름
- 결과가 크고 일부 필드만 필요하면 `values()`·`values_list()`로 모델 인스턴스 생성 비용도 함께 줄일 수 있음

<br>

### 6. 지연 평가가 만드는 실무 함정

**함정 ① 트랜잭션·시점 불일치** — QuerySet을 만든 시점과 평가 시점 사이에 데이터가 바뀔 수 있다. 특히 QuerySet을 반환한 뒤 뷰나 템플릿에서 평가되면, 서비스 함수가 의도한 시점과 다른 데이터를 볼 수 있다.

**함정 ② 뷰 밖으로 새어 나가는 지연 쿼리** — 서비스 함수가 QuerySet을 반환하고 시리얼라이저·템플릿에서 평가하면, 쿼리 실행 위치가 흩어져 성능 분석이 어렵다. 경계를 넘길 때는 `list()`로 평가하거나 반환 타입을 명확히 정한다.

**함정 ③ `bool()` 검사** — `if qs:`는 전체를 조회한다. 결과를 이어서 쓰지 않을 거라면 `exists()`가 옳다.

```python
# 지연 쿼리를 명시적으로 평가해 시점을 고정하는 예
def get_active_products_snapshot() -> list[Product]:
    return list(Product.objects.filter(is_active=True))  # 여기서 실행·고정
```

> 💡 Django 4.1 이상에서는 `async for post in qs`, `await qs.acount()`, `await qs.afirst()` 등 **비동기 평가 메서드**가 추가되었다. 지연 평가 규칙은 동일하며, 비동기 뷰에서 동기 평가(`list(qs)`)를 하면 `SynchronousOnlyOperation` 예외가 발생한다(unit09 참고).

<br>

### 7. 정리

| **질문**                                     | **핵심 답변**                                                                                 |
| -------------------------------------------- | --------------------------------------------------------------------------------------------- |
| **QuerySet은 언제 실행되는가?**              | 반복·`len()`·`bool()`·`list()`·pickle·`repr()`·step 슬라이싱 등 **평가 시점**에 실행            |
| **`filter()`를 여러 번 호출하면 쿼리도 여러 번?** | 아니다. 새 QuerySet만 만들고 **평가 시 SQL 1회**로 합쳐짐                                   |
| **같은 QuerySet을 두 번 순회하면?**          | `_result_cache` 덕분에 **1회만** 실행. 단 새 QuerySet(`filter` 등)은 캐시를 공유하지 않음        |
| **`qs[0]`이 매번 쿼리인 이유는?**            | 인덱스 접근은 `LIMIT 1` 쿼리를 실행하고 **캐시를 만들지 않음**                                  |
| **`exists()` vs `bool(qs)`**                 | 결과를 안 쓸 거면 `exists()`, 이어서 순회할 거면 `bool()`로 캐시를 채움                         |
| **대량 순회 시 메모리 대책은?**              | `iterator(chunk_size=N)` — 캐시를 만들지 않고 청크로 스트리밍                                   |

- QuerySet은 **쿼리 설명서**이며 평가 전까지 DB에 가지 않음
- 캐시는 **QuerySet 객체 단위**로 관리되므로, 같은 객체를 재사용하는지 새 객체를 만드는지가 쿼리 수를 결정함
- 조회 최적화(`select_related`·`prefetch_related`·`only`·`annotate`)는 unit03에서 이어서 다룸
