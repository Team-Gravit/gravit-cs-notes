## 트랜잭션 관리

Django는 기본적으로 쿼리 하나하나를 즉시 커밋하는 **자동 커밋(autocommit)** 모드로 동작하며, 여러 쿼리를 하나의 작업 단위로 묶으려면 `transaction.atomic`을 명시해야 한다. 이 유닛은 `atomic` 블록의 중첩 규칙(세이브포인트), 커밋 이후에만 실행돼야 하는 작업을 위한 `on_commit` 훅, 요청 단위 트랜잭션 설정을 다룬다. 트랜잭션·격리 수준 자체의 개념은 database 과목 unit13·unit17을 참고할 것.

<br>

### 1. Django의 기본 동작 — 자동 커밋

- Django는 DB 연결을 **autocommit 모드**로 열기 때문에, `save()`·`update()`·`delete()`가 실행되는 즉시 각각 커밋됨
- 따라서 "재고 차감 → 주문 생성" 같은 두 쿼리를 그냥 나열하면 중간에 예외가 나도 앞의 쿼리는 이미 반영된 상태가 됨
- 여러 쿼리를 **모두 성공하거나 모두 취소**되게 하려면 `transaction.atomic`으로 묶어야 함

```python
from django.db import transaction

# 안티패턴: 두 번째 save()가 실패하면 재고만 줄어든 채로 남음
product.stock -= 1
product.save()
Order.objects.create(product=product, user=user)

# 개선: 하나의 작업 단위로 묶기
with transaction.atomic():
    product.stock -= 1
    product.save()
    Order.objects.create(product=product, user=user)   # 실패 시 재고 변경도 롤백
```

- `atomic`은 **컨텍스트 매니저**와 **데코레이터** 양쪽으로 사용 가능함
- 블록이 정상 종료되면 커밋, 예외가 빠져나가면 롤백됨. 예외는 삼키지 않고 그대로 다시 던져짐

> 💡 `select_for_update()`(비관적 잠금)는 반드시 `atomic` 블록 안에서 평가해야 한다. 자동 커밋 모드에서 호출하면 `TransactionManagementError`가 발생한다. 락은 트랜잭션이 끝날 때 해제되므로, 블록 밖에서는 잠금 자체가 의미가 없기 때문이다.

<br>

### 2. atomic 블록 중첩과 세이브포인트

`atomic`은 중첩할 수 있다. **가장 바깥 블록만 실제 트랜잭션(BEGIN/COMMIT)** 을 열고, 안쪽 블록은 **세이브포인트(SAVEPOINT)** 로 변환된다.

```
with atomic():                    ── BEGIN
    A.save()
    with atomic():                ── SAVEPOINT sp1
        B.save()
        raise ValueError          ── ROLLBACK TO SAVEPOINT sp1  (B만 취소)
    # 예외를 여기서 잡으면 바깥 트랜잭션은 계속 유효
    C.save()
                                  ── COMMIT  (A, C 반영)
```

```python
def place_order(user, product):
    with transaction.atomic():                  # 바깥: 진짜 트랜잭션
        order = Order.objects.create(user=user, product=product)
        try:
            with transaction.atomic():          # 안쪽: 세이브포인트
                apply_coupon(order)             # 실패해도 주문 자체는 유지
        except CouponError:
            order.note = "쿠폰 적용 실패"
            order.save()
        return order
```

| **항목**            | **바깥 atomic**                 | **안쪽 atomic (중첩)**                       |
| ------------------- | ------------------------------- | -------------------------------------------- |
| **SQL**             | `BEGIN` / `COMMIT` / `ROLLBACK` | `SAVEPOINT` / `RELEASE` / `ROLLBACK TO`      |
| **예외 발생 시**    | 전체 롤백                       | **세이브포인트까지만** 롤백, 바깥은 계속 진행 |
| **`savepoint=False`** | 의미 없음                     | 세이브포인트 생략 → 실패 시 **바깥까지 오염** |
| **`durable=True`**  | 정상                            | 중첩된 위치에서 사용하면 `RuntimeError`      |

- `atomic(savepoint=False)`는 세이브포인트 비용을 아끼지만, 안쪽에서 예외가 나면 바깥 트랜잭션 전체가 롤백 예정 상태가 됨
- `atomic(durable=True)`(Django 3.2+)는 "이 블록이 **반드시 가장 바깥 트랜잭션**이어야 한다"를 강제함. 서비스 함수가 다른 atomic 안에서 호출되어 커밋이 지연되는 것을 막고 싶을 때 사용함

<br>

### 3. 흔한 함정 — atomic 안에서 DB 예외 삼키기

`atomic` 블록 안에서 `IntegrityError` 같은 **DB 예외를 try/except로 잡고 계속 쿼리를 실행**하면, 다음 쿼리에서 `TransactionManagementError`가 발생한다. DB 입장에서 트랜잭션은 이미 실패 상태이며, Django도 롤백 필요 플래그를 세워 두기 때문이다.

```python
# 안티패턴
with transaction.atomic():
    try:
        Tag.objects.create(name="python")      # 유니크 위반 → IntegrityError
    except IntegrityError:
        pass
    Tag.objects.get(name="python")             # TransactionManagementError!

# 개선: 실패할 수 있는 구간을 안쪽 atomic(세이브포인트)으로 격리
with transaction.atomic():
    try:
        with transaction.atomic():
            Tag.objects.create(name="python")
    except IntegrityError:
        pass                                   # 세이브포인트까지만 롤백됨
    Tag.objects.get(name="python")             # 정상 동작
```

> ⚠️ "예외를 잡았으니 괜찮다"는 생각이 가장 흔한 실수다. **DB 예외는 세이브포인트 단위로 격리**해야 하며, 그렇지 않으면 블록이 끝날 때까지 어떤 쿼리도 실행할 수 없다. `get_or_create()`가 내부적으로 안쪽 atomic을 사용하는 이유도 바로 이것이다.

<br>

### 4. on_commit — 커밋 이후에만 실행할 작업

이메일 발송, 외부 API 호출, Celery 태스크 큐잉처럼 **되돌릴 수 없는 사이드이펙트**는 트랜잭션이 실제로 커밋된 뒤에 실행해야 한다. 블록 안에서 바로 실행하면 롤백되어도 이미 발송된 뒤가 된다.

```python
from django.db import transaction

# 안티패턴: 커밋 전에 태스크 발행 → 워커가 아직 존재하지 않는 order_id를 조회
with transaction.atomic():
    order = Order.objects.create(...)
    send_receipt.delay(order.id)       # 롤백되면 유령 주문에 메일 발송

# 개선: 커밋된 뒤에만 실행
with transaction.atomic():
    order = Order.objects.create(...)
    transaction.on_commit(lambda: send_receipt.delay(order.id))
```

```
BEGIN ─ INSERT order ─ on_commit(콜백 등록) ─ COMMIT ─▶ 콜백 실행
BEGIN ─ INSERT order ─ on_commit(콜백 등록) ─ ROLLBACK ─▶ 콜백 폐기
```

- 콜백은 **가장 바깥 트랜잭션이 커밋된 직후**, 등록 순서대로 실행됨. 안쪽 세이브포인트가 롤백되면 그 안에서 등록된 콜백은 버려짐
- 트랜잭션 밖(자동 커밋 상태)에서 호출하면 **즉시 실행**됨
- 콜백 안에서 예외가 나면 이후 콜백은 실행되지 않음. Django 4.2+의 `on_commit(fn, robust=True)`는 예외를 로깅만 하고 다음 콜백을 계속 실행함
- 콜백은 커밋 이후에 실행되므로 그 안의 DB 작업은 **새 트랜잭션**이며, 실패해도 원래 커밋은 되돌릴 수 없음

> 💡 Celery 태스크 발행 시 "워커에서 `DoesNotExist`가 간헐적으로 난다"는 문제는 대부분 커밋 전에 태스크를 발행한 경우다. **워커가 DB에서 객체를 읽어야 하는 태스크는 예외 없이 `on_commit`으로 감싼다**는 팀 규칙을 두는 것이 좋다.

<br>

### 4-1. 테스트에서의 on_commit

`TestCase`는 각 테스트를 트랜잭션으로 감싸고 끝나면 롤백하므로, 커밋이 일어나지 않아 **on_commit 콜백이 실행되지 않는다**.

```python
class OrderTest(TestCase):
    def test_receipt_sent(self):
        with self.captureOnCommitCallbacks(execute=True) as callbacks:
            place_order(self.user, self.product)
        self.assertEqual(len(callbacks), 1)
```

- Django 3.2+의 `captureOnCommitCallbacks(execute=True)`로 콜백을 수집하고 즉시 실행할 수 있음
- 실제 커밋이 필요한 시나리오는 `TransactionTestCase`를 사용함 (느리므로 최소화)

<br>

### 5. 요청 단위 트랜잭션 — ATOMIC_REQUESTS

뷰 전체를 하나의 트랜잭션으로 감싸는 설정이다. `DATABASES` 설정에서 DB별로 지정한다.

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "ATOMIC_REQUESTS": True,   # 모든 뷰를 atomic으로 감쌈
    }
}
```

```python
from django.db import transaction

@transaction.non_atomic_requests      # 특정 뷰만 제외
def long_running_export(request):
    ...
```

| **항목**          | **ATOMIC_REQUESTS = True**                         | **명시적 atomic (기본)**                          |
| ----------------- | -------------------------------------------------- | ------------------------------------------------- |
| **적용 범위**     | 미들웨어 이후 **뷰 함수 전체**                     | 개발자가 지정한 블록만                            |
| **장점**          | 빼먹을 일이 없음, 단순한 CRUD 앱에 편리            | 트랜잭션 범위 최소화, 성능·락 경합 제어 용이       |
| **단점**          | 뷰가 길면 **트랜잭션·락 보유 시간 증가**, 외부 API 호출도 트랜잭션 안 | 매번 명시해야 함                    |
| **예외 처리**     | 뷰가 예외를 던지면 전체 롤백 (응답 500)            | 블록 단위 롤백                                    |
| **on_commit**     | 응답 반환 직전에 커밋 → 콜백 실행                  | 블록 종료 시 커밋 → 콜백 실행                     |

- `ATOMIC_REQUESTS`는 **미들웨어에는 적용되지 않음** — 뷰 함수만 감싸므로, 미들웨어에서 실행한 쿼리는 별도로 커밋됨
- 뷰 안에서 예외를 잡아 정상 응답(예: 400)을 돌려주면 **롤백되지 않고 커밋**됨. 실패 응답이면서 데이터를 되돌리고 싶다면 `transaction.set_rollback(True)`를 호출해야 함
- DRF에서도 동일하게 동작하며, `APIView`가 예외를 핸들러로 변환해 응답을 만들면 커밋됨 — DRF는 `atomic` 안에서 `APIException`이 발생하면 롤백 처리를 해 주지만, 직접 잡은 예외는 그렇지 않음

<br>

### 6. 트랜잭션 설계 원칙

```
요청 처리 흐름에서 트랜잭션의 위치 (권장)

[요청 파싱·검증]  ──▶  [ atomic: DB 변경만 ]  ──▶  [on_commit: 외부 호출·큐잉]  ──▶  [응답]
   트랜잭션 밖              짧고 좁게                커밋 이후                      
```

- **짧게**: 외부 API 호출·파일 업로드·긴 계산은 트랜잭션 밖으로 뺀다. 트랜잭션이 길수록 락 보유 시간과 MVCC 버전 유지 비용이 늘어남
- **경계를 서비스 계층에**: 뷰가 아니라 `place_order()` 같은 서비스 함수에 `@transaction.atomic`을 붙이면 재사용·테스트가 쉬워짐. 외부에서 다시 감싸이는 것을 막으려면 `durable=True`
- **사이드이펙트는 on_commit**: 메일·푸시·Celery·캐시 무효화는 커밋 이후
- **DB 예외는 세이브포인트로 격리**: 실패 가능한 삽입은 안쪽 `atomic`으로 감싸고 잡는다
- **동시 갱신 보호**: 재고·잔액처럼 경쟁이 있는 값은 `select_for_update()` 또는 `F()` 표현식으로 처리하고, 격리 수준의 동작은 DB별로 확인함 (MySQL InnoDB는 REPEATABLE READ, PostgreSQL은 READ COMMITTED가 기본값)

<br>

### 7. 면접·실무 체크포인트

| **질문**                                              | **핵심 답변**                                                                                     |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| **Django의 기본 트랜잭션 모드는?**                     | **autocommit**. 쿼리마다 즉시 커밋되므로 원자성이 필요하면 `atomic`으로 묶어야 함                   |
| **atomic을 중첩하면?**                                 | 바깥만 진짜 트랜잭션, 안쪽은 **세이브포인트**. 안쪽 실패는 안쪽까지만 롤백                          |
| **atomic 안에서 IntegrityError를 잡으면 왜 오류가?**   | 트랜잭션이 실패 상태라 이후 쿼리 불가 → **안쪽 atomic으로 격리** 후 잡아야 함                       |
| **on_commit은 언제 쓰나?**                             | 메일·태스크 큐·외부 API처럼 **롤백할 수 없는 작업**. 바깥 트랜잭션 커밋 직후 실행, 롤백 시 폐기       |
| **TestCase에서 on_commit이 안 도는 이유는?**           | 테스트가 롤백으로 끝나 커밋이 없음 → `captureOnCommitCallbacks(execute=True)`                       |
| **ATOMIC_REQUESTS의 단점은?**                          | 뷰 전체가 트랜잭션이라 **락 보유 시간 증가**, 외부 호출까지 트랜잭션 안에 들어감                    |
| **durable=True의 용도는?**                             | 해당 블록이 **최상위 트랜잭션임을 보장** — 다른 atomic 안에서 호출되면 예외                          |
