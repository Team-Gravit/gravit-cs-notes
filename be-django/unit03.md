## 조회 최적화와 N+1

ORM은 관련 객체를 편하게 탐색하게 해주지만, 그 편의가 반복문 안에서 **N+1 쿼리 문제**로 되돌아온다. 이 유닛은 N+1이 생기는 원리와 `select_related`·`prefetch_related`·`only`/`defer`·`annotate`를 언제 어떻게 골라 써야 하는지를 다룬다. unit02(지연 평가)의 캐시 개념을 전제로 한다.

<br>

### 1. N+1 문제란

목록을 1번의 쿼리로 가져온 뒤, 각 행의 관련 객체에 접근할 때마다 **행 수(N)만큼 추가 쿼리**가 발생하는 현상이다. 코드에는 반복문 하나뿐이라 눈에 띄지 않지만, 데이터가 늘수록 쿼리 수가 선형으로 증가한다.

```python
# 안티패턴: 게시글 100개 → 쿼리 101회
posts = Post.objects.all()                 # SELECT * FROM post            (1회)
for post in posts:
    print(post.author.name)                # SELECT * FROM user WHERE id=? (N회)
```

```
쿼리 타임라인 (N=3)
① SELECT * FROM post                       ← 목록
② SELECT * FROM user WHERE id = 7          ← post[0].author
③ SELECT * FROM user WHERE id = 3          ← post[1].author
④ SELECT * FROM user WHERE id = 7          ← post[2].author (같은 사용자여도 다시 조회)
```

- 정참조(`post.author`)는 인스턴스별로 캐시되지만, **인스턴스가 다르면 캐시도 다르므로** 같은 저자를 여러 번 조회함
- 역참조·다대다(`post.comments.all()`)는 매번 새 QuerySet이라 항상 쿼리가 발생함
- DRF의 `ModelSerializer`가 중첩 시리얼라이저나 `SerializerMethodField`에서 관련 객체를 건드리면 **뷰 코드에는 반복문이 없어도** N+1이 발생함 (unit07 참고)

> 💡 N+1은 로컬의 소량 데이터로는 체감되지 않는다. 개발 단계에서 **django-debug-toolbar**로 쿼리 수를 눈으로 확인하거나, 테스트에서 `self.assertNumQueries(2)`로 쿼리 수를 고정해 회귀를 막는 습관이 중요하다.

<br>

### 2. select_related — JOIN으로 한 번에 가져오기

`select_related`는 SQL **JOIN**을 사용해 관련 객체를 같은 쿼리에서 함께 조회한다. 결과 행에 관련 객체의 컬럼이 포함되므로 쿼리는 **1회**로 끝난다.

```python
posts = Post.objects.select_related("author", "category")
for post in posts:
    print(post.author.name, post.category.title)   # 추가 쿼리 없음
```

```sql
SELECT post.*, user.*, category.*
FROM   post
INNER JOIN user     ON post.author_id = user.id
LEFT OUTER JOIN category ON post.category_id = category.id
```

- **ForeignKey·OneToOneField**(정참조)와 OneToOne 역참조에만 사용 가능함 — "행 하나에 관련 객체 하나"인 관계여야 JOIN 결과가 부풀지 않음
- `null=True`인 FK는 `LEFT OUTER JOIN`, 아니면 `INNER JOIN`으로 생성됨
- 이중 언더스코어로 깊이 탐색 가능: `select_related("author__profile")`

❗️**다대다·역참조 FK에는 쓸 수 없다**: `select_related("comments")`는 `FieldError`를 발생시킨다. 1:N 관계를 JOIN하면 부모 행이 N배로 복제되기 때문에 Django가 허용하지 않는다.

<br>

### 3. prefetch_related — 별도 쿼리 후 파이썬에서 결합

`prefetch_related`는 본 쿼리를 먼저 실행하고, 관련 객체를 **`WHERE id IN (...)` 형태의 별도 쿼리 1회**로 가져온 뒤 파이썬 메모리에서 짝을 맞춘다. 관계 종류에 제한이 없다.

```python
posts = Post.objects.prefetch_related("tags", "comments")
for post in posts:
    print(post.tags.all())        # prefetch 캐시 사용, 쿼리 없음
    print(post.comments.all())    # prefetch 캐시 사용, 쿼리 없음
```

```
① SELECT * FROM post
② SELECT * FROM post_tags JOIN tag ... WHERE post_id IN (1,2,3,...)
③ SELECT * FROM comment WHERE post_id IN (1,2,3,...)
→ 총 3회 (N과 무관)
```

<br>

### 3-1. Prefetch 객체로 관련 쿼리 제어

관련 객체에 필터·정렬·추가 최적화를 적용하려면 `Prefetch`를 사용한다.

```python
from django.db.models import Prefetch

posts = Post.objects.prefetch_related(
    Prefetch(
        "comments",
        queryset=Comment.objects.filter(is_deleted=False)
                                .select_related("writer")
                                .order_by("-created_at"),
        to_attr="visible_comments",   # 리스트 속성으로 저장
    )
)
for post in posts:
    for c in post.visible_comments:   # post.comments.all()이 아님
        print(c.writer.name)
```

> ⚠️ prefetch한 관계에 `.filter()`·`.exclude()`·`.order_by()`를 다시 붙이면 **캐시를 버리고 새 쿼리**를 실행해 N+1이 되살아난다. 조건이 필요하면 반드시 `Prefetch(queryset=...)`로 미리 걸고, 파이썬에서 조건 분기가 필요하면 `to_attr` 리스트를 순회한다.

<br>

### 3-2. select_related vs prefetch_related 선택 기준

| **항목**           | **select_related**                       | **prefetch_related**                             |
| ------------------ | ---------------------------------------- | ------------------------------------------------ |
| **동작 방식**      | SQL **JOIN**, 쿼리 1회                    | 별도 쿼리 + **파이썬에서 결합**, 쿼리 1+관계 수    |
| **적용 관계**      | FK·OneToOne (정참조, OneToOne 역참조)     | **모든 관계** (M2M, 역참조 FK, GenericRelation 포함) |
| **결과 크기**      | 행마다 관련 컬럼 반복 → 넓은 행           | 관계별 결과를 따로 가져와 중복 적음               |
| **추가 조건**      | 불가 (JOIN 조건 고정)                    | `Prefetch(queryset=...)`로 가능                   |
| **적합한 상황**    | 1:1·N:1 관계, 관련 객체가 작을 때         | 1:N·M:N 관계, 관련 객체가 많거나 조건이 필요할 때 |

- 두 가지는 **함께 쓸 수 있음**: `Post.objects.select_related("author").prefetch_related("tags")`
- `prefetch_related("comments__writer")`처럼 prefetch 안에서 깊이 탐색도 가능하며, 단계마다 쿼리가 1회씩 추가됨
- 이미 조회된 인스턴스 리스트에는 `prefetch_related_objects(instances, "tags")`로 나중에 prefetch를 적용할 수 있음

<br>

### 4. only·defer — 컬럼 단위 최적화

모든 컬럼이 필요하지 않을 때 **가져올 컬럼을 제한**한다. `only()`는 지정한 필드만, `defer()`는 지정한 필드를 제외하고 조회한다.

```python
# 목록 화면에 본문(body)이 필요 없을 때
posts = Post.objects.defer("body")                 # body 제외
posts = Post.objects.only("id", "title", "author") # 세 필드만
```

```python
# 안티패턴: 지연 필드에 접근하면 인스턴스마다 추가 쿼리 → 또 다른 N+1
for post in Post.objects.only("title"):
    print(post.body)     # SELECT body FROM post WHERE id=?  (N회)
```

- 지연된 필드에 접근하면 Django가 **그 인스턴스만을 위한 쿼리**를 자동 실행하므로, 오히려 N+1을 만들 수 있음
- `only`·`defer`는 모델 인스턴스가 필요할 때(메서드·프로퍼티 호출 등) 쓰고, 단순 값만 필요하면 `values()`·`values_list()`가 더 가볍고 안전함
- 큰 `TextField`·`JSONField`·바이너리 컬럼을 목록 조회에서 제외하는 것이 가장 효과가 큰 사용 사례임

> 💡 Django 5.x는 `Post.objects.only("title").select_related("author")`처럼 결합할 때 관련 모델의 필드도 `only("author__name")` 형식으로 제한할 수 있다. 이때 관련 모델의 기본키는 자동 포함된다.

<br>

### 5. annotate — 집계를 DB에 맡기기

관련 객체를 "세기 위해서" 가져오는 것은 낭비다. `annotate`는 `COUNT`·`SUM`·`EXISTS`·서브쿼리를 **SQL로 계산해 컬럼처럼 붙인다**.

```python
from django.db.models import Count, Exists, OuterRef, Subquery

# 안티패턴: 댓글 수를 세려고 댓글 전체를 prefetch 하거나 매번 count()
for post in Post.objects.all():
    n = post.comments.count()            # N회 COUNT 쿼리

# 개선 ①: GROUP BY 집계
posts = Post.objects.annotate(comment_count=Count("comments"))

# 개선 ②: 상관 서브쿼리 — 최신 댓글 작성자, 좋아요 여부
latest = Comment.objects.filter(post=OuterRef("pk")).order_by("-created_at")
posts = Post.objects.annotate(
    last_writer_id=Subquery(latest.values("writer_id")[:1]),
    liked_by_me=Exists(Like.objects.filter(post=OuterRef("pk"), user_id=user.id)),
)
```

- `annotate(Count("comments"))`는 JOIN + `GROUP BY`로 변환되며, **두 개 이상의 1:N 관계를 동시에 Count하면 조인 결과가 곱해져 값이 부풀어 오름** — 이때는 `Count("comments", distinct=True)`를 쓰거나 `Subquery`로 분리함
- `Exists`·`Subquery`는 조인 없이 행마다 계산되어 GROUP BY 부작용이 없고, `filter(Exists(...))`처럼 조건으로도 사용 가능함
- 집계값은 파이썬 필드가 아니라 **쿼리 결과 컬럼**이므로, 정렬(`order_by("-comment_count")`)과 필터(`filter(comment_count__gt=0)`)에 그대로 쓸 수 있음

<br>

### 6. 최적화 순서와 판단 기준

```
1) 쿼리 수 측정 (debug-toolbar / assertNumQueries / connection.queries)
      │
2) N+1 발견 → 관계 종류 확인
      ├─ FK·OneToOne(정참조)      → select_related
      ├─ 역참조 FK·M2M            → prefetch_related (+ Prefetch로 조건)
      └─ 개수·존재 여부만 필요     → annotate(Count/Exists) 또는 Subquery
      │
3) 행이 너무 넓다 → defer / only / values
      │
4) 그래도 느리다 → 인덱스·쿼리 플랜(EXPLAIN) 확인, 캐시·비정규화 검토
```

| **증상**                                  | **1차 대응**                              | **주의점**                                          |
| ----------------------------------------- | ----------------------------------------- | --------------------------------------------------- |
| **쿼리 수가 행 수에 비례**                | `select_related` / `prefetch_related`     | prefetch 후 `.filter()` 금지                        |
| **개수·합계만 필요한데 객체를 다 가져옴** | `annotate(Count/Sum)`, `aggregate`        | 다중 1:N Count는 `distinct=True` 또는 Subquery      |
| **행이 넓고 큰 컬럼 포함**                | `defer` / `only` / `values`               | 지연 필드 접근 시 N+1                               |
| **쿼리 1회지만 JOIN 결과가 거대**         | `select_related` → `prefetch_related` 전환 | 1:N 관계를 JOIN하면 행 수 폭증                      |

<br>

### 7. 면접·실무 체크포인트

- **N+1이란?** 목록 1회 + 행마다 관련 객체 조회 N회. ORM의 지연 로딩과 인스턴스 단위 캐시 구조가 원인
- **select_related vs prefetch_related?** 전자는 **JOIN·정참조 전용·쿼리 1회**, 후자는 **별도 IN 쿼리·모든 관계·파이썬 결합**
- **prefetch 후 `.filter()`를 붙이면?** 캐시를 무시하고 새 쿼리 → `Prefetch(queryset=...)`로 조건을 미리 건다
- **`only()`가 오히려 느려지는 경우?** 지연 필드에 반복 접근할 때. 값만 필요하면 `values()`
- **annotate로 두 관계를 동시에 Count하면 값이 커지는 이유?** JOIN이 곱집합을 만들기 때문 → `distinct=True` 또는 `Subquery`
- **회귀 방지 방법?** `assertNumQueries`로 테스트에 쿼리 수를 고정하고, DRF 시리얼라이저의 중첩 필드는 뷰의 `get_queryset`에서 prefetch를 책임진다
