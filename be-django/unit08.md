## 시그널과 암묵적 결합

Django **시그널(Signal)** 은 "어떤 일이 일어났다"를 발신자(sender)가 알리면 미리 등록된 **수신자(receiver)** 함수가 실행되는 옵저버(Observer) 패턴 구현체다. 앱 간 결합을 줄이려는 의도로 설계되었지만, 실무에서는 **호출 흐름이 코드에 드러나지 않는 암묵적 결합**을 만들어 사이드이펙트 추적을 어렵게 하는 주범이 되곤 한다. 이 유닛은 시그널의 동작 원리, 사용을 지양해야 하는 근거, 그리고 대안을 다룬다.

<br>

### 1. 시그널의 동작 원리

```
Order.save()
   │
   ├─ pre_save.send(sender=Order, instance=order)      ── 수신자들 동기 실행
   │
   ├─ INSERT / UPDATE 실행
   │
   └─ post_save.send(sender=Order, instance=order, created=True)
          │
          ├─ receiver ①  재고 차감
          ├─ receiver ②  알림 발송          ← 등록된 순서대로, 같은 스레드에서
          └─ receiver ③  통계 갱신
   ▼
save() 반환  (모든 수신자가 끝난 뒤)
```

```python
# orders/signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver


@receiver(post_save, sender=Order, dispatch_uid="orders.notify_on_create")
def notify_on_create(sender, instance, created, **kwargs):
    if created:
        send_order_notification(instance.id)


# orders/apps.py — 수신자 등록은 AppConfig.ready()에서
class OrdersConfig(AppConfig):
    name = "orders"

    def ready(self):
        from . import signals  # noqa: F401  (임포트 자체가 등록)
```

- 수신자는 **동기적으로, 발신자와 같은 스레드에서, `save()`가 반환되기 전에** 실행됨. "비동기 이벤트"가 아니라 사실상 **숨겨진 함수 호출**임
- 수신자에서 예외가 나면 `send()`는 그대로 전파해 **`save()` 자체가 실패**함. `send_robust()`는 예외를 잡아 반환값으로 돌려줌
- `dispatch_uid`를 지정하지 않으면 모듈이 두 번 임포트될 때 **같은 수신자가 중복 등록**되어 두 번 실행될 수 있음
- Django 5.0부터 `asend()`·`asend_robust()`와 비동기 수신자를 지원함(unit09 참고)

> 💡 시그널의 발신은 `Signal.send()`가 수신자 리스트를 순회하며 **그냥 함수를 차례로 호출**하는 것에 불과하다. 메시지 큐처럼 별도 프로세스에서 나중에 처리되는 것이 아니므로, "시그널로 빼면 응답이 빨라진다"는 기대는 성립하지 않는다.

| **내장 시그널**                          | **발생 시점**                                     | **주요 인자**                            |
| ---------------------------------------- | ------------------------------------------------- | ---------------------------------------- |
| **pre_save / post_save**                 | `Model.save()` 전후                               | `instance`, `created`, `update_fields`, `raw` |
| **pre_delete / post_delete**             | `delete()` 전후 (QuerySet.delete 포함, 객체마다)    | `instance`                               |
| **m2m_changed**                          | 다대다 `add`·`remove`·`clear`                      | `action`, `pk_set`                       |
| **pre_migrate / post_migrate**           | `migrate` 전후                                    | 초기 데이터·권한 생성 등                 |
| **request_started / request_finished**   | 요청 처리 시작·종료                                |                                          |
| **user_logged_in / user_logged_out**     | `login()`·`logout()` 호출 시                       | `request`, `user`                        |

<br>

### 2. 시그널이 발생하지 않는 경우

시그널을 믿고 로직을 걸어 두면 **아래 경로에서는 실행되지 않아** 데이터 정합성이 깨진다.

| **경로**                          | **pre/post_save** | **pre/post_delete** | **비고**                                |
| --------------------------------- | ----------------- | ------------------- | --------------------------------------- |
| `instance.save()`                 | 발생              | -                   |                                         |
| **`QuerySet.update()`**           | **발생 안 함**    | -                   | SQL UPDATE 직접 실행                    |
| **`bulk_create()` / `bulk_update()`** | **발생 안 함** | -                   | 대량 처리 최적화 경로                   |
| `QuerySet.delete()`               | -                 | 발생 (객체 수집 후) | 단, 수신자가 없으면 **fast delete**로 수집 생략 |
| **원시 SQL·DB 직접 수정**         | **발생 안 함**    | **발생 안 함**      | 배치·마이그레이션·외부 도구             |
| `loaddata`(fixture)               | 발생 (`raw=True`) | -                   | 수신자에서 `raw` 확인 필요              |

> ⚠️ "post_save로 재고를 차감한다"는 설계는 누군가 `Order.objects.filter(...).update(status=...)`를 쓰는 순간 조용히 깨진다. **시그널은 모델 변경의 완전한 훅이 아니다.** 반드시 실행되어야 하는 규칙은 시그널에 둘 수 없다.

<br>

### 3. 사이드이펙트 추적이 곤란한 이유

```python
# views.py — 이 코드만 보면 "주문을 저장한다"뿐이다
order = Order.objects.create(user=user, total=total)
```

실제로는 등록된 수신자 수만큼 **보이지 않는 작업**이 일어난다. 문제는 다음과 같이 구체화된다.

- **호출 관계가 코드에 없다**: `create()` 호출부에서 정의로 이동해도 재고·알림·통계 로직이 보이지 않음. 전체 코드베이스에서 `post_save` 수신자를 grep 해야 흐름을 알 수 있음
- **실행 순서가 보장되지 않는다**: 수신자는 **등록(임포트) 순서**대로 실행되며, 앱 로딩 순서는 `INSTALLED_APPS`와 임포트 경로에 좌우됨. 수신자 간 의존이 생기면 미묘한 순서 버그가 됨
- **트랜잭션 경계와 어긋난다**: `post_save`는 "저장됨"이 아니라 "**INSERT 문이 실행됨**"이다. `atomic` 안이면 아직 커밋 전이며, 이후 롤백될 수 있음. 여기서 메일을 보내거나 태스크를 큐에 넣으면 유령 주문에 대한 알림이 나감 (unit04 참고)
- **재귀·중복 실행**: 수신자 안에서 `instance.save()`를 부르면 다시 `post_save`가 발생함. 중복 임포트로 수신자가 두 번 등록되기도 함
- **테스트가 무거워진다**: 단순히 `Order`를 만드는 픽스처마다 알림·외부 API 수신자가 딸려 옴. 매 테스트에서 `disconnect`하거나 `mock`을 걸어야 함
- **성능 비용이 숨는다**: 모든 `save()`에 수신자 비용이 붙으며, 관련 객체를 조회하는 수신자는 대량 저장 루프에서 N+1을 만듦

```
증상: "주문 하나 저장했는데 API 응답이 2초"
원인 추적 경로:
  views.py ─▶ Order.save() ─▶ (post_save) ─▶ ??? 
                                     └─ analytics/signals.py: 외부 통계 API 동기 호출
                                     └─ notifications/signals.py: 푸시 발송
   → 호출부 어디에도 analytics·notifications가 등장하지 않음
```

<br>

### 4. 대안 — 명시적 호출

**원칙**: 한 유스케이스에서 일어나는 일은 **한 함수 안에서 위에서 아래로 읽히게** 만든다.

```python
# 안티패턴: 시그널로 흩어진 사이드이펙트
@receiver(post_save, sender=Order)
def on_order_saved(sender, instance, created, **kwargs):
    if created:
        Stock.objects.filter(product=instance.product).update(qty=F("qty") - instance.qty)
        send_push.delay(instance.user_id)        # 커밋 전 발행 → 유령 알림 위험
```

```python
# 개선: 서비스 함수에 흐름을 명시
from django.db import transaction


def place_order(user, product, qty) -> Order:
    with transaction.atomic():
        stock = Stock.objects.select_for_update().get(product=product)
        if stock.qty < qty:
            raise OutOfStock()
        stock.qty -= qty
        stock.save(update_fields=["qty"])
        order = Order.objects.create(user=user, product=product, qty=qty)
        transaction.on_commit(lambda: send_push.delay(user.id))   # 커밋 후 발송
    return order
```

| **관심사**                          | **시그널 대신 권장하는 방법**                                              |
| ----------------------------------- | -------------------------------------------------------------------------- |
| **유스케이스 흐름(재고·주문·알림)** | **서비스 함수**에서 명시적으로 순서대로 호출                                |
| **모델 자체의 불변식** (슬러그 생성, 정규화) | `save()` 오버라이드 또는 `clean()` — 단, `update()`·`bulk_create` 경로는 여전히 우회됨을 인지 |
| **커밋 이후 사이드이펙트**          | `transaction.on_commit()`                                                  |
| **여러 앱이 반응해야 하는 도메인 이벤트** | 명시적 **이벤트 발행 함수**(예: `events.publish(OrderPlaced(...))`)와 등록 목록을 한 곳에 모아 관리, 또는 메시지 큐 |
| **파생 데이터 갱신(통계·검색 인덱스)** | 배치·비동기 태스크로 분리, 원본 저장 경로와 결합하지 않음                  |

> 💡 "시그널을 쓰면 앱 간 결합이 줄어든다"는 주장은 **의존 방향**만 바뀌었을 뿐 결합 자체는 그대로다. 주문 앱은 알림 앱을 몰라도 되지만, 알림 앱이 주문 모델의 저장 방식에 묶이고, 그 사실을 아무도 모르게 된다. 결합은 줄이는 것보다 **보이게 하는 것**이 먼저다.

<br>

### 5. 그래도 시그널이 적절한 경우

시그널을 전면 금지할 필요는 없다. **자신이 소유하지 않은 코드의 이벤트에 반응**해야 할 때는 시그널이 사실상 유일한 훅이다.

- `user_logged_in`처럼 **프레임워크·서드파티 앱이 발생시키는 이벤트**에 반응할 때 (마지막 로그인 IP 기록 등)
- 재사용 가능한 라이브러리를 만들며, 사용자 프로젝트의 모델 저장에 끼어들어야 할 때 (감사 로그, 캐시 무효화 라이브러리)
- `post_migrate`로 초기 데이터·권한을 생성할 때

시그널을 쓰기로 했다면 다음 규칙을 지킨다.

- `dispatch_uid`를 항상 지정하고, 수신자는 앱당 `signals.py` 한 곳에 모아 `ready()`에서 등록함
- 수신자 안에서는 **DB 쓰기·외부 호출을 하지 않고**, 하더라도 `on_commit` 안에서 함
- 수신자에서 `raw` 인자를 확인해 fixture 로딩 시 건너뜀
- 시그널에 의존하는 규칙이 있다면 `update()`·`bulk_create()` 사용을 코드 리뷰에서 막거나, 해당 규칙을 서비스 함수로 옮김

<br>

### 6. 면접·실무 체크포인트

| **질문**                                          | **핵심 답변**                                                                                   |
| ------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| **시그널은 비동기인가?**                           | 아니다. **같은 스레드에서 동기적으로**, `save()`가 반환되기 전에 실행되는 숨겨진 함수 호출          |
| **`update()`·`bulk_create()`에서 post_save가 발생하나?** | **발생하지 않음**. 시그널은 모델 변경의 완전한 훅이 아님                                     |
| **post_save에서 메일을 보내면 안 되는 이유?**       | `atomic` 안이면 **커밋 전**이라 롤백 시 유령 알림 → `on_commit` 사용                              |
| **시그널이 추적을 어렵게 하는 이유?**               | 호출부에 흐름이 없고, 실행 순서가 임포트 순서에 의존하며, 중복 등록·재귀 위험                       |
| **시그널 대신 무엇을 쓰나?**                        | **서비스 함수의 명시적 호출**, 모델 `save()` 오버라이드, `on_commit`, 명시적 이벤트 발행           |
| **그래도 시그널이 맞는 경우는?**                     | **소유하지 않은 코드**(프레임워크·서드파티)의 이벤트에 반응할 때, 재사용 라이브러리, `post_migrate` |
| **수신자가 두 번 실행되는 원인은?**                  | 모듈 중복 임포트로 중복 등록 → `dispatch_uid` 지정, `ready()`에서만 등록                          |
