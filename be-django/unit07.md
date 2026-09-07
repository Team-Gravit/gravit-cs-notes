## DRF 직렬화와 검증

Django REST Framework(DRF)의 **Serializer**는 모델·파이썬 객체를 JSON으로 바꾸는 **직렬화(serialization)** 와, 클라이언트 입력을 검증해 파이썬 값으로 바꾸는 **역직렬화·검증(deserialization & validation)** 을 담당한다. `is_valid()` 한 줄 뒤에서 검증이 어떤 순서로 실행되는지, 그리고 `ModelSerializer`의 자동 생성이 어디까지 책임지고 어디서부터 직접 써야 하는지를 알아야 API 계층의 버그와 성능 문제를 설명할 수 있다.

<br>

### 1. Serializer의 두 방향

```
직렬화 (출력)    Model 인스턴스 ─▶ serializer.data ─▶ JSON
                 to_representation()

역직렬화 (입력)  JSON ─▶ serializer(data=...) ─▶ is_valid() ─▶ validated_data ─▶ save()
                                                 to_internal_value()  create()/update()
```

- `Serializer(instance)`는 출력용, `Serializer(data=request.data)`는 입력용, 둘 다 넘기면 **수정(update)** 용임
- `serializer.data`는 `is_valid()` 없이도 접근 가능하지만, 입력 검증 전에 접근하면 `AssertionError`가 발생함
- `context={"request": request}`로 뷰의 요청 정보를 넘길 수 있으며, `HyperlinkedModelSerializer`나 현재 사용자 기반 로직에 필요함

<br>

### 2. 검증 흐름 — is_valid()가 하는 일

`is_valid()`는 내부적으로 `run_validation(data)`를 호출하며, 검증은 **필드 단위 → 시리얼라이저 단위** 순서로 진행된다. 앞 단계에서 오류가 하나라도 있으면 뒤 단계는 실행되지 않는다.

```
is_valid()
  └─ run_validation(data)
       ① 빈 값 처리      required / allow_null / default 확인
       ② to_internal_value(data)          ── 필드 단위
            각 필드마다:
              a. Field.to_internal_value()  타입 변환 (문자열 → 날짜 등)
              b. Field.validators           max_length, UniqueValidator 등
              c. validate_<필드명>(value)    시리얼라이저에 정의한 필드 훅
            → 오류가 있으면 필드별로 모아 ValidationError, 여기서 중단
       ③ run_validators(attrs)            ── Meta.validators (UniqueTogetherValidator 등)
       ④ validate(attrs)                  ── 객체 단위, 필드 간 교차 검증
  └─ 성공: validated_data / 실패: errors
```

```python
from rest_framework import serializers


class EventSerializer(serializers.ModelSerializer):
    class Meta:
        model = Event
        fields = ["id", "title", "starts_at", "ends_at", "capacity"]

    def validate_capacity(self, value):            # ② c. 필드 훅
        if value <= 0:
            raise serializers.ValidationError("정원은 1 이상이어야 합니다.")
        return value                               # 반환값이 validated_data에 들어감

    def validate(self, attrs):                     # ④ 교차 검증
        if attrs["starts_at"] >= attrs["ends_at"]:
            raise serializers.ValidationError({"ends_at": "종료는 시작 이후여야 합니다."})
        return attrs
```

- `validate_<필드명>`은 **변환된 값**을 받고, 반드시 값을 **반환**해야 함. 반환을 빼먹으면 `validated_data`에 `None`이 들어감
- `validate()`는 모든 필드 검증이 통과한 뒤 `attrs`(OrderedDict)를 받으며, 특정 필드에 오류를 붙이려면 `{"필드명": "메시지"}` 형태로 던짐
- `partial=True`(PATCH)면 `attrs`에 일부 키만 있으므로 `validate()`에서 `attrs.get()`으로 접근하거나 `self.instance` 값과 합쳐 판단해야 함

> 💡 필드 훅에서 오류가 나면 `validate()`는 실행되지 않는다. "필드 오류와 교차 검증 오류가 한 번에 모두 보이길 원한다"는 요구는 DRF 기본 흐름과 맞지 않으므로, 프런트와 오류 응답 형식을 합의할 때 이 점을 미리 공유해야 한다.

<br>

### 3. 저장 흐름 — save()와 create/update

```python
serializer = EventSerializer(data=request.data, context={"request": request})
serializer.is_valid(raise_exception=True)          # 실패 시 400 응답 자동 처리
event = serializer.save(organizer=request.user)    # 추가 kwargs는 validated_data에 병합
```

- `save()`는 `self.instance` 유무에 따라 **`create(validated_data)`** 또는 **`update(instance, validated_data)`** 를 호출함
- `save(**kwargs)`로 넘긴 값은 검증을 거치지 않고 `validated_data`에 합쳐짐 → 요청자·서버 시각처럼 **클라이언트가 정해선 안 되는 값**을 주입하는 표준 방법
- `is_valid(raise_exception=True)`는 `ValidationError`를 던지고, DRF 예외 핸들러가 400 응답으로 변환함

<br>

### 4. ModelSerializer의 자동 생성과 한계

`ModelSerializer`는 모델 필드로부터 시리얼라이저 필드·검증기·`create`/`update`를 자동 생성한다. 편리하지만 "자동"이 감추는 한계를 알아야 한다.

| **자동으로 해주는 것**                                 | **해주지 않는 것 (한계)**                                                    |
| ------------------------------------------------------ | ---------------------------------------------------------------------------- |
| 모델 필드 → 시리얼라이저 필드 매핑 (타입·`max_length` 등) | **`Model.clean()`·`full_clean()` 호출 안 함** — 모델 검증 로직은 무시됨      |
| `unique=True` → `UniqueValidator` (DB 조회 1회)        | **중첩 관계 쓰기(create/update) 미지원** — 직접 구현 필요                     |
| `unique_together`·`UniqueConstraint` → `UniqueTogetherValidator` | `depth` 옵션의 중첩은 **읽기 전용**이며 N+1을 유발하기 쉬움           |
| 단순 `create()`·`update()`                             | 비즈니스 규칙(재고·상태 전이 등)은 모름                                       |
| `read_only_fields`·`extra_kwargs`로 옵션 조정          | `fields = "__all__"`은 새 필드가 **자동 노출**되는 보안 위험                   |

<br>

### 4-1. 중첩 쓰기는 직접 구현해야 한다

```python
class OrderSerializer(serializers.ModelSerializer):
    items = OrderItemSerializer(many=True)   # 중첩 시리얼라이저

    class Meta:
        model = Order
        fields = ["id", "items", "memo"]

    # 안티패턴: 이대로 save() 호출 → AssertionError
    # "The `.create()` method does not support writable nested fields by default."

    # 개선: 트랜잭션 안에서 명시적으로 생성
    def create(self, validated_data):
        items_data = validated_data.pop("items")
        with transaction.atomic():
            order = Order.objects.create(**validated_data)
            OrderItem.objects.bulk_create(
                [OrderItem(order=order, **item) for item in items_data]
            )
        return order
```

- 중첩 `update`는 더 어렵다 — 기존 자식 중 무엇을 삭제·수정·추가할지 정책이 필요하므로, 자식 리소스를 **별도 엔드포인트**로 분리하는 설계가 흔히 더 낫다
- 쓰기용과 읽기용 시리얼라이저를 **분리**(`OrderWriteSerializer`·`OrderReadSerializer`)하면 각각 단순해짐. 뷰의 `get_serializer_class()`에서 `self.action`에 따라 선택함

> ⚠️ `ModelSerializer`는 **모델의 `clean()`을 호출하지 않는다**(DRF 3.0 이후). 모델에 검증을 두었다고 안심하면 API 경로로는 우회된다. 검증은 시리얼라이저에 두거나, 시리얼라이저 `validate()`에서 모델 인스턴스를 만들어 `full_clean()`을 명시적으로 호출한다.

<br>

### 4-2. 직렬화 단계의 N+1

```python
class PostSerializer(serializers.ModelSerializer):
    author_name = serializers.CharField(source="author.name")     # FK 접근
    comment_count = serializers.SerializerMethodField()

    def get_comment_count(self, obj):
        return obj.comments.count()          # 게시글마다 COUNT 쿼리
```

- `source="author.name"`, 중첩 시리얼라이저, `SerializerMethodField`의 관련 객체 접근은 **행마다 쿼리**를 만든다. 뷰 코드에는 반복문이 없어 발견이 늦음
- 해결은 시리얼라이저가 아니라 **뷰의 `get_queryset()`** 에서: `select_related("author").annotate(comment_count=Count("comments"))` 후 시리얼라이저는 `IntegerField(read_only=True)`로 값을 그대로 노출 (unit03 참고)
- 시리얼라이저는 "받은 객체를 그리는 일"만 하고, **데이터를 어떻게 가져올지는 뷰·서비스 계층이 책임**진다는 원칙을 지키면 N+1이 구조적으로 줄어듦

<br>

### 5. Serializer vs ModelSerializer 선택 기준

| **상황**                                          | **권장**                              | **이유**                                                   |
| ------------------------------------------------- | ------------------------------------- | ---------------------------------------------------------- |
| 단일 모델의 단순 CRUD                             | **ModelSerializer**                   | 필드·검증기 자동 생성, 보일러플레이트 최소                  |
| 모델과 무관한 입력 (검색 조건, 로그인 폼, 액션 요청) | **Serializer**                       | 모델 매핑이 없으므로 `create()`도 불필요, 순수 검증기로 사용 |
| 여러 모델을 조합한 응답                            | **Serializer** 또는 읽기 전용 ModelSerializer | 자동 매핑이 오히려 방해                              |
| 중첩 생성·복잡한 비즈니스 규칙                     | ModelSerializer + `create()` 오버라이드, 또는 **서비스 함수 호출** | 트랜잭션·도메인 규칙을 시리얼라이저 밖에서 관리 |

```python
# 모델 없는 입력 검증 — 순수 Serializer
class TransferSerializer(serializers.Serializer):
    from_account = serializers.IntegerField()
    to_account = serializers.IntegerField()
    amount = serializers.DecimalField(max_digits=12, decimal_places=2, min_value=1)

    def validate(self, attrs):
        if attrs["from_account"] == attrs["to_account"]:
            raise serializers.ValidationError("같은 계좌로 이체할 수 없습니다.")
        return attrs

# 뷰에서는 검증만 맡기고 실행은 서비스 계층으로
serializer = TransferSerializer(data=request.data)
serializer.is_valid(raise_exception=True)
transfer_money(**serializer.validated_data)     # 트랜잭션은 서비스 함수가 관리 (unit04 참고)
```

> 💡 시리얼라이저의 `create()`에 비즈니스 로직을 몰아넣으면 관리자 명령·배치·시그널 등 **API 외 경로에서 재사용할 수 없다**. "시리얼라이저는 입력 검증과 출력 형식, 서비스 함수는 도메인 규칙"으로 역할을 나누는 것이 실무 표준이다.

<br>

### 6. 면접·실무 체크포인트

| **질문**                                              | **핵심 답변**                                                                                  |
| ----------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| **`is_valid()`의 검증 순서는?**                        | 필드 변환·필드 검증기·`validate_<필드>` → `Meta.validators` → `validate()`. 앞 단계 실패 시 뒤는 생략 |
| **`validate_<필드>`와 `validate()`의 차이는?**         | 전자는 **단일 필드**, 후자는 모든 필드 통과 후 **교차 검증**. 둘 다 값을 반환해야 함              |
| **`save(owner=request.user)`처럼 넘긴 값은 검증되나?** | 아니다. `validated_data`에 **그대로 병합** → 서버가 정하는 값 주입용                              |
| **ModelSerializer가 모델 `clean()`을 호출하나?**       | 호출하지 않음. 검증은 **시리얼라이저에** 두거나 `full_clean()`을 직접 호출                        |
| **중첩 시리얼라이저로 바로 생성이 되나?**              | 기본 미지원 → `create()`/`update()` 오버라이드, 트랜잭션으로 감쌈                                 |
| **시리얼라이저에서 N+1이 나는 이유와 해결?**           | `source="fk.field"`·MethodField가 행마다 쿼리 → 뷰 `get_queryset()`에서 `select_related`·`annotate` |
| **읽기·쓰기 시리얼라이저를 분리하는 이유?**            | 노출 필드·검증 규칙이 다르고, 중첩 읽기(N+1 주의)와 중첩 쓰기 정책을 **독립적으로** 관리하기 위해   |
