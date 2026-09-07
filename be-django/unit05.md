## 마이그레이션

**마이그레이션(Migration)** 은 모델 변경을 DB 스키마 변경으로 옮기는 버전 관리 체계다. 개발 환경에서는 `makemigrations`·`migrate` 두 명령으로 끝나지만, 실서비스에서는 "구버전 코드와 신버전 스키마가 동시에 살아 있는 순간"을 견디도록 **변경 순서를 설계**해야 한다. 이 유닛은 마이그레이션의 동작 원리, 스키마 변경 전략, 무중단 배포 시 순서, 데이터 마이그레이션을 다룬다.

<br>

### 1. 마이그레이션의 동작 원리

```
models.py 변경
   │  makemigrations  (모델 상태 vs 마지막 마이그레이션 상태 비교 → 차이를 operations로 기록)
   ▼
app/migrations/0007_add_phone.py    (dependencies + operations)
   │  migrate         (미적용 파일을 의존성 순서로 실행, django_migrations 테이블에 기록)
   ▼
DB 스키마 변경 + django_migrations 행 추가
```

- 마이그레이션 파일은 **모델의 이력(그래프)** 이다. Django는 파일들을 순서대로 재생해 "코드 기준 현재 모델 상태"를 계산하고, 실제 `models.py`와 비교해 차이를 새 파일로 만든다
- 적용 여부는 DB의 `django_migrations` 테이블로 추적함. 파일은 있는데 테이블에 없으면 "미적용", 반대면 "파일 유실"로 판단함
- `dependencies`는 앱 간·파일 간 순서를 정의함. 같은 앱에 리프 노드가 둘 이상이면(브랜치 병합 후 흔함) `makemigrations --merge`로 병합 파일을 만들어야 함

| **명령**                          | **역할**                                                       |
| --------------------------------- | -------------------------------------------------------------- |
| **`makemigrations`**              | 모델 변경을 감지해 마이그레이션 파일 생성                       |
| **`makemigrations --check`**      | 생성할 변경이 있으면 **비정상 종료** → CI에서 누락 감지          |
| **`migrate`**                     | 미적용 마이그레이션 실행                                        |
| **`migrate app 0006`**            | 해당 번호까지 **되돌리기**(역방향 실행)                          |
| **`sqlmigrate app 0007`**         | 실제 실행될 **SQL 출력** (배포 전 검토 필수)                     |
| **`showmigrations`**              | 적용 상태 확인                                                  |
| **`migrate --fake`**              | 실행 없이 적용된 것으로 기록 (수동 반영 후 동기화용, 남용 금지)  |
| **`squashmigrations`**            | 여러 파일을 하나로 압축                                         |

> 💡 CI 파이프라인에 `makemigrations --check`를 넣으면 "모델은 바꿨는데 마이그레이션을 커밋하지 않은" 사고를 원천 차단할 수 있다. 실무에서 가장 가성비 높은 안전장치다.

<br>

### 2. 마이그레이션과 트랜잭션

- PostgreSQL·SQLite는 DDL이 트랜잭션에 포함되므로 마이그레이션 하나가 **원자적으로** 실행됨(중간 실패 시 전체 롤백)
- **MySQL·Oracle은 DDL이 암묵적 커밋**을 유발하므로, 중간에 실패하면 절반만 적용된 상태가 남을 수 있음 → 한 파일에 여러 스키마 변경을 넣지 않고 작게 나누는 것이 안전함
- 마이그레이션 클래스에 `atomic = False`를 지정하면 트랜잭션 없이 실행됨. PostgreSQL의 `CREATE INDEX CONCURRENTLY`처럼 **트랜잭션 안에서 실행할 수 없는 작업**에 필요함

```python
from django.contrib.postgres.operations import AddIndexConcurrently
from django.db import migrations, models


class Migration(migrations.Migration):
    atomic = False                      # CONCURRENTLY는 트랜잭션 밖에서만 가능
    dependencies = [("orders", "0011_previous")]
    operations = [
        AddIndexConcurrently(
            model_name="order",
            index=models.Index(fields=["created_at"], name="order_created_idx"),
        ),
    ]
```

> ⚠️ 일반 `AddIndex`는 PostgreSQL에서 테이블에 쓰기 잠금을 걸고 인덱스를 만든다. 수백만 행 테이블이면 그동안 INSERT·UPDATE가 모두 대기한다. 운영 DB에서는 **`AddIndexConcurrently` + `atomic = False`** 조합이 기본이다.

<br>

### 3. 스키마 변경별 위험도

같은 "컬럼 하나"라도 변경 종류에 따라 락 시간과 호환성 위험이 크게 다르다.

| **변경**                          | **DB 락·비용**                                            | **구버전 코드와 호환**                     | **권장 전략**                                     |
| --------------------------------- | --------------------------------------------------------- | ------------------------------------------ | ------------------------------------------------- |
| **nullable 컬럼 추가**            | 낮음 (메타데이터 변경)                                     | **호환** (구코드는 해당 컬럼을 모름)        | 그대로 진행                                        |
| **NOT NULL + default 컬럼 추가**  | PostgreSQL 11+·MySQL 8.0+는 빠름, 구버전은 **테이블 재작성** | 호환 (default로 채워짐)                    | DB 버전 확인, 대형 테이블은 nullable로 추가 후 변환 |
| **컬럼 삭제**                     | 낮음                                                       | **비호환** — 구코드가 해당 컬럼을 SELECT    | 코드에서 참조 제거 배포 **후** 삭제                |
| **컬럼 이름 변경**                | 낮음                                                       | **비호환** — 구코드·신코드 중 하나는 실패   | 추가 → 이중 쓰기 → 백필 → 전환 → 삭제              |
| **타입 변경**                     | 대부분 **테이블 재작성**                                   | 상황에 따라 다름                            | 새 컬럼 추가 후 이전 방식                          |
| **인덱스 추가**                   | PostgreSQL 일반 방식은 **쓰기 잠금**                       | 호환                                        | `AddIndexConcurrently`                            |
| **유니크·FK 제약 추가**           | 기존 데이터 전체 검증                                      | 위반 데이터 있으면 실패                     | 사전 정리 후 추가, PostgreSQL은 `NOT VALID` 고려    |

- Django는 `default=`를 **파이썬 수준 기본값**으로만 취급해, 컬럼 추가 시 임시로 DB default를 넣어 채운 뒤 **제거**함. DB 수준 기본값을 유지하려면 Django 5.0의 **`db_default`** 를 사용함
- 구버전 코드는 모델에 없는 컬럼을 SELECT하지 않으므로(Django는 컬럼을 명시적으로 나열함) **컬럼 추가는 안전**하지만, **삭제·이름 변경은 구코드가 죽는다**

<br>

### 4. 무중단 배포 시 순서

롤링 배포 중에는 **구버전 코드와 신버전 코드가 같은 DB를 동시에 사용**한다. 따라서 마이그레이션은 "양쪽 코드 모두와 호환되는 상태"만 만들어야 한다. 이를 **확장·수축(Expand & Contract)** 패턴이라 한다.

```
시간 →
              ┌────── 롤링 배포 구간 ──────┐
DB 스키마 :  v1 ─── v1+확장(추가) ──────────────── 수축(삭제)
앱 코드   :  v1 ─── v1 ‖ v2 (혼재) ────── v2 ──────── v2'
             ▲                             ▲            ▲
        ① 확장 마이그레이션            ② 코드 v2 배포   ③ 수축 마이그레이션
        (nullable 추가 등, v1과 호환)   (새 컬럼 사용)  (구 컬럼 삭제, v2와 호환)
```

**컬럼 추가**: ① 마이그레이션 먼저(nullable 또는 `db_default`) → ② 코드 배포
**컬럼 삭제**: ① 코드에서 참조 제거 후 배포 → ② 마이그레이션(`RemoveField`)
**컬럼 이름 변경** (가장 위험, 5단계):

```
1) 새 컬럼 추가 (nullable)                     ── 마이그레이션
2) 코드: 쓰기 시 구·신 컬럼 모두 기록, 읽기는 구 컬럼   ── 배포
3) 기존 데이터 백필 (배치)                       ── 데이터 마이그레이션 or 커맨드
4) 코드: 읽기를 신 컬럼으로 전환, 구 컬럼 쓰기 중단   ── 배포
5) 구 컬럼 삭제                                 ── 마이그레이션
```

❗️**`RenameField`를 운영에서 바로 쓰지 말 것**: SQL 한 줄(`ALTER ... RENAME`)이라 편해 보이지만, 실행 순간 구버전 코드는 옛 이름을, 신버전 코드는 새 이름을 찾으므로 어느 쪽이든 반드시 오류가 난다.

> 💡 "마이그레이션을 코드보다 먼저 실행할지, 나중에 실행할지"가 면접 단골 질문이다. 정답은 **"변경 종류에 따라 다르다"** — 추가는 먼저, 삭제는 나중, 이름 변경은 양쪽으로 쪼갠다.

<br>

### 5. 데이터 마이그레이션

스키마가 아니라 **데이터 자체를 변환**해야 할 때(백필, 코드값 변환, 기본 행 삽입) `RunPython`을 사용한다.

```python
from django.db import migrations


def backfill_display_name(apps, schema_editor):
    User = apps.get_model("accounts", "User")      # 이 시점의 모델 상태를 사용
    batch = []
    for user in User.objects.filter(display_name="").iterator(chunk_size=1000):
        user.display_name = user.username
        batch.append(user)
        if len(batch) >= 1000:
            User.objects.bulk_update(batch, ["display_name"])
            batch.clear()
    if batch:
        User.objects.bulk_update(batch, ["display_name"])


class Migration(migrations.Migration):
    dependencies = [("accounts", "0012_user_display_name")]
    operations = [
        migrations.RunPython(backfill_display_name, migrations.RunPython.noop),
    ]
```

- **`apps.get_model()`을 반드시 사용**한다. `from accounts.models import User`처럼 직접 임포트하면 "현재 코드의 모델"이 실행되어, 나중에 추가된 필드를 참조하다 실패하거나 사용자 정의 메서드·시그널이 예기치 않게 동작함
- 이력 모델(historical model)에는 사용자 정의 메서드·`save()` 오버라이드·시그널이 **없다**. 순수 필드 조작만 가능함
- 두 번째 인자(역방향 함수)를 주지 않으면 되돌리기가 불가능함. 되돌릴 게 없다면 `RunPython.noop`을 명시함
- 스키마 마이그레이션과 데이터 마이그레이션은 **파일을 분리**함. 한 파일에 섞으면 PostgreSQL에서도 "같은 트랜잭션에서 ALTER한 테이블을 다시 갱신할 수 없다"는 오류가 나거나, 실패 시 복구가 어려워짐

> ⚠️ 수백만 행의 백필을 마이그레이션 안에서 돌리면 배포 파이프라인이 수십 분간 멈추고, 트랜잭션 하나로 묶여 락과 Undo/WAL이 폭증한다. 대형 백필은 **관리 명령(management command)으로 분리해 배치·재시도 가능하게** 만들고, 마이그레이션은 스키마만 담당하게 한다.

<br>

### 6. 운영 체크리스트

- 배포 전 `sqlmigrate`로 **실제 SQL을 눈으로 확인**한다 (테이블 재작성·락 여부)
- 마이그레이션은 **작게, 되돌릴 수 있게**: 파일 하나에 한 가지 의도, 역방향 함수 제공
- 마이그레이션 실행 주체를 하나로 고정한다(배포 잡 또는 특정 인스턴스). 여러 인스턴스가 동시에 `migrate`를 실행하면 `django_migrations` 경쟁·중복 실행 위험이 있음
- 마이그레이션 파일은 **자동 생성이라도 리뷰 대상**이다. 특히 `AlterField`가 예상치 못한 타입 변경(테이블 재작성)을 만들 수 있음
- 파일이 수십 개로 늘면 `squashmigrations`로 압축하되, **모든 환경에 원본이 적용된 뒤**에만 원본을 삭제한다
- 테스트 DB 생성 시간이 길어지면 마이그레이션 수를 의심한다 — 테스트는 마이그레이션을 처음부터 전부 재생하기 때문

<br>

### 7. 면접·실무 체크포인트

| **질문**                                          | **핵심 답변**                                                                                  |
| ------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| **마이그레이션 적용 여부는 어떻게 추적하나?**      | `django_migrations` 테이블. 파일 그래프와 비교해 미적용분만 실행                                 |
| **컬럼 추가와 삭제, 배포 순서가 다른 이유는?**     | 구코드는 새 컬럼을 몰라도 되지만 **삭제된 컬럼은 계속 SELECT**하므로 삭제는 코드 배포 후           |
| **컬럼 이름 변경을 무중단으로 하려면?**            | `RenameField` 대신 **추가 → 이중 쓰기 → 백필 → 읽기 전환 → 삭제** 5단계                          |
| **RunPython에서 모델을 직접 import하면 안 되는 이유?** | 현재 코드의 모델이 실행되어 **그 시점에 없는 필드**를 참조하거나 시그널·메서드가 동작함           |
| **인덱스 추가가 서비스를 멈추게 하는 경우?**       | PostgreSQL 일반 `CREATE INDEX`는 쓰기 잠금 → `AddIndexConcurrently` + `atomic=False`             |
| **MySQL에서 마이그레이션이 절반만 적용된 이유?**   | **DDL이 암묵적 커밋**되어 트랜잭션 롤백이 불가 → 파일을 작게 나눔                                 |
| **`db_default`와 `default`의 차이?**               | `default`는 파이썬 기본값(DB 컬럼에는 남지 않음), `db_default`(5.0+)는 **DB 수준 기본값**          |
