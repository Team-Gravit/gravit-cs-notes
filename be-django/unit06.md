## 인증과 권한

**인증(Authentication)** 은 "누구인가"를 확인하는 것이고, **인가(Authorization, 권한)** 는 "무엇을 할 수 있는가"를 판단하는 것이다. Django는 `django.contrib.auth`로 사용자 모델·인증 백엔드·모델 단위 권한을 제공하고, DRF는 그 위에 인증 클래스와 권한 클래스를 얹어 API 요청 단위로 검사한다. 이 유닛은 **Custom User Model 설계**와 **Permission 클래스** 구현에 집중한다. 세션·JWT 자체의 원리는 web-security 과목 unit06을 참고할 것.

<br>

### 1. 인증과 권한의 처리 흐름

```
요청 ─▶ AuthenticationMiddleware ─▶ request.user (지연 로딩, 세션 기반)
              │
              ▼  DRF APIView.initial()
        ① authentication_classes 순회 → request.user / request.auth 확정
        ② permission_classes.has_permission()    ── 뷰 진입 가능?
        ③ throttle 검사
              │
              ▼  뷰 본문
        get_object() → check_object_permissions() → has_object_permission()
```

- Django 미들웨어가 채운 `request.user`는 세션 기반이며, DRF는 자신의 인증 클래스로 **다시 판단**해 `request.user`를 덮어씀 (unit01 참고)
- 인증 실패는 **401**(인증 헤더가 있는 방식) 또는 **403**, 권한 실패는 **403**으로 응답됨

<br>

### 2. Custom User Model 설계

**왜 처음부터 커스텀 모델을 써야 하는가**

Django 공식 문서는 **프로젝트 시작 시점에 반드시 커스텀 사용자 모델을 설정**할 것을 강력히 권장한다. 기본 `auth.User`를 쓰다가 중간에 바꾸면 `auth_user` 테이블을 참조하는 모든 FK·M2M(권한, 세션, 관리자 로그 등)을 수동으로 옮겨야 하며, 마이그레이션 이력을 수술해야 한다.

```python
# settings.py — 첫 migrate 전에 설정해야 함
AUTH_USER_MODEL = "accounts.User"
```

> ⚠️ `AUTH_USER_MODEL`은 **첫 마이그레이션 이전**에 결정해야 한다. 이미 운영 중이라면 "테이블 이름을 `auth_user`로 맞춘 커스텀 모델 + 마이그레이션 fake" 같은 우회가 있지만 위험이 크므로, 필드가 당장 필요 없더라도 **빈 `AbstractUser` 상속 모델을 만들어 두는 것**이 표준 관행이다.

<br>

### 2-1. AbstractUser vs AbstractBaseUser

| **항목**              | **AbstractUser**                                        | **AbstractBaseUser + PermissionsMixin**                    |
| --------------------- | ------------------------------------------------------- | ---------------------------------------------------------- |
| **포함 필드**         | username, email, first/last_name, is_staff, is_active, date_joined 등 **전부** | password, last_login **만**. 나머지는 직접 정의     |
| **로그인 식별자**     | 기본 username (변경 가능)                               | `USERNAME_FIELD`로 **자유롭게 지정** (이메일·전화번호 등)   |
| **매니저**            | 기본 `UserManager` 제공                                 | `BaseUserManager` 상속해 **직접 작성**                      |
| **관리자 화면·폼**    | 기본 제공                                               | `UserAdmin`·생성/변경 폼 커스터마이징 필요                  |
| **적합한 경우**       | 기본 구조를 유지하며 필드만 추가                        | username 없이 이메일 로그인, 필드 구성을 완전히 통제할 때   |

```python
# accounts/models.py — 이메일을 식별자로 쓰는 커스텀 모델
from django.contrib.auth.models import AbstractBaseUser, BaseUserManager, PermissionsMixin
from django.db import models


class UserManager(BaseUserManager):
    def create_user(self, email, password=None, **extra):
        if not email:
            raise ValueError("이메일은 필수입니다.")
        user = self.model(email=self.normalize_email(email), **extra)
        user.set_password(password)          # 해시 저장 (평문 저장 금지)
        user.save(using=self._db)
        return user

    def create_superuser(self, email, password=None, **extra):
        extra.setdefault("is_staff", True)
        extra.setdefault("is_superuser", True)
        return self.create_user(email, password, **extra)


class User(AbstractBaseUser, PermissionsMixin):
    email = models.EmailField(unique=True)
    nickname = models.CharField(max_length=30)
    is_active = models.BooleanField(default=True)
    is_staff = models.BooleanField(default=False)   # 관리자 사이트 접근 여부

    objects = UserManager()

    USERNAME_FIELD = "email"          # 로그인 식별자
    REQUIRED_FIELDS = ["nickname"]    # createsuperuser가 추가로 묻는 필드
```

- `PermissionsMixin`은 `is_superuser`·`groups`·`user_permissions`와 `has_perm()` 계열 메서드를 제공함. 빼면 Django 권한 체계와 관리자 사이트를 쓸 수 없음
- `is_active=False`인 사용자는 `ModelBackend`가 인증을 거부하고 `has_perm()`도 항상 `False`를 반환함

<br>

### 2-2. 사용자 모델 참조 규칙

```python
# models.py — FK는 문자열 설정값으로
from django.conf import settings

class Post(models.Model):
    author = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)

# 그 외(뷰·서비스·시리얼라이저) — 호출 시점에 모델 클래스 획득
from django.contrib.auth import get_user_model
User = get_user_model()
```

- 모델 정의에서는 `settings.AUTH_USER_MODEL`(문자열)을 쓴다. `get_user_model()`을 모듈 최상단에서 호출하면 앱 로딩 순서에 따라 `AppRegistryNotReady`가 날 수 있음
- 재사용 가능한 앱·라이브러리를 만들 때 `from django.contrib.auth.models import User`를 직접 임포트하면 커스텀 모델을 쓰는 프로젝트에서 깨짐

<br>

### 3. Django 권한 체계

- 모델마다 `add`·`change`·`delete`·`view` 네 가지 권한이 **자동 생성**됨 (`Permission` 테이블, 코드명 `app_label.action_modelname`)
- `Meta.permissions = [("publish_post", "게시글 발행 가능")]`로 사용자 정의 권한 추가 가능
- 사용자에게 직접 부여하거나 **그룹(Group)** 에 묶어 부여함. `user.has_perm("blog.publish_post")`로 검사하며, 슈퍼유저는 항상 `True`
- `has_perm()` 결과는 요청 동안 사용자 인스턴스에 **캐시**되므로, 권한을 바꾼 뒤 같은 인스턴스로 검사하면 예전 값을 볼 수 있음 → DB에서 다시 조회한 인스턴스로 확인

> 💡 모델 단위 권한은 "이 사용자가 게시글을 수정할 수 있는가"까지만 답한다. "**자기** 게시글만 수정"처럼 **객체 단위 권한**은 Django 기본 백엔드가 지원하지 않으므로 DRF의 `has_object_permission`이나 django-guardian 같은 별도 구현이 필요하다.

<br>

### 4. DRF 권한 클래스 — 두 단계 검사

`BasePermission`은 두 메서드를 가진다. **둘 다 통과해야** 요청이 허용된다.

| **메서드**                    | **호출 시점**                                 | **인자**                | **호출되지 않는 경우**                                   |
| ----------------------------- | --------------------------------------------- | ----------------------- | -------------------------------------------------------- |
| **`has_permission`**          | 뷰 진입 전 (`initial()`)                      | `request, view`         | 없음 — 항상 호출                                          |
| **`has_object_permission`**   | `get_object()` 내부 `check_object_permissions` | `request, view, obj`    | **목록(list) 뷰**, `get_object()`를 안 쓰는 직접 조회       |

```python
from rest_framework import permissions


class IsOwnerOrReadOnly(permissions.BasePermission):
    message = "작성자만 수정할 수 있습니다."

    def has_permission(self, request, view):
        # 읽기는 누구나, 쓰기는 로그인 필요
        return request.method in permissions.SAFE_METHODS or request.user.is_authenticated

    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:
            return True
        return obj.author_id == request.user.id
```

```python
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.select_related("author")
    serializer_class = PostSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly & IsOwnerOrReadOnly]
```

- DRF 3.9+는 `&`·`|`·`~` 연산자로 권한 클래스를 **조합**할 수 있음. 리스트에 여러 개를 나열하면 AND와 같음
- `has_object_permission`은 **`has_permission`이 통과한 뒤에만** 호출됨. 객체 검사만 구현하고 `has_permission`을 생략하면 기본값 `True`라 뷰에는 진입함

❗️**직접 조회하면 객체 권한이 검사되지 않는다**: `get_object()` 대신 `Post.objects.get(pk=pk)`로 꺼내 쓰면 `has_object_permission`이 호출되지 않는다. 직접 조회할 때는 반드시 `self.check_object_permissions(request, obj)`를 호출해야 한다.

<br>

### 4-1. 내장 권한 클래스와 선택 기준

| **클래스**                        | **동작**                                                     | **적합한 상황**                                  |
| --------------------------------- | ------------------------------------------------------------ | ------------------------------------------------ |
| **AllowAny**                      | 항상 허용                                                    | 공개 API. 명시적으로 적어 의도를 드러냄           |
| **IsAuthenticated**               | 로그인 사용자만                                              | 대부분의 내부 API 기본값                          |
| **IsAuthenticatedOrReadOnly**     | 읽기는 모두, 쓰기는 로그인                                    | 게시판·댓글 등 공개 읽기 서비스                   |
| **IsAdminUser**                   | `is_staff=True`만                                            | 운영 도구 API                                     |
| **DjangoModelPermissions**        | HTTP 메서드를 모델 권한(add/change/delete)에 매핑             | 관리자 그룹별 세밀한 제어. `queryset` 속성 필요    |
| **DjangoObjectPermissions**       | 위 + 객체 단위 권한 백엔드(django-guardian 등)                | 객체 권한 백엔드를 도입한 경우                    |

- `DjangoModelPermissions`의 기본 `perms_map`은 GET·HEAD·OPTIONS에 권한을 요구하지 않음(버전에 따라 다를 수 있음). 읽기에도 `view` 권한을 요구하려면 서브클래스에서 `perms_map`을 덮어씀
- 전역 기본값은 `REST_FRAMEWORK["DEFAULT_PERMISSION_CLASSES"]`로 지정함. **기본값을 `IsAuthenticated`로 두고 공개 API만 `AllowAny`를 명시**하는 것이 실수를 줄이는 방향임

<br>

### 5. 인증 클래스와의 관계

- 인증 클래스는 `request.user`를 **결정**하고, 권한 클래스는 그 사용자를 **판정**함. 인증 클래스가 비어 있으면 `request.user`는 `AnonymousUser`가 되어 `IsAuthenticated`는 항상 실패함
- `SessionAuthentication`은 세션 + CSRF 검사를 수행함(브라우저 클라이언트용). 토큰·JWT 방식은 헤더 기반이며 CSRF 검사가 없으므로 토큰 노출 방지가 핵심
- Django 5.0부터 `aauthenticate()`·`alogin()`·`alogout()` 등 비동기 인증 함수가, 5.1부터 `login_required` 데코레이터의 비동기 뷰 지원과 모든 뷰에 로그인을 강제하는 `LoginRequiredMiddleware`가 추가됨. DRF의 인증·권한 클래스는 동기 전제이므로 비동기 뷰와 섞을 때는 unit09를 참고할 것

> 💡 "권한 검사를 미들웨어에서 할까, 권한 클래스에서 할까"는 **적용 범위**로 결정한다. 모든 요청에 예외 없이 적용되면 미들웨어(예: 5.1의 `LoginRequiredMiddleware`), 뷰·객체마다 다르면 권한 클래스가 맞다.

<br>

### 6. 면접·실무 체크포인트

| **질문**                                          | **핵심 답변**                                                                                       |
| ------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| **커스텀 User 모델을 처음부터 만드는 이유는?**     | 중간 변경 시 `auth_user`를 참조하는 모든 FK·마이그레이션을 수동 수술해야 함. **빈 `AbstractUser`라도 미리** |
| **AbstractUser vs AbstractBaseUser?**             | 전자는 기본 필드 유지 + 확장, 후자는 **비밀번호·last_login만** 주고 식별자·매니저를 직접 설계             |
| **`USERNAME_FIELD`와 `REQUIRED_FIELDS`?**         | 로그인 식별자 / `createsuperuser`가 추가로 요구하는 필드 (USERNAME_FIELD·password 제외)               |
| **FK에서 User를 어떻게 참조하나?**                | 모델에서는 `settings.AUTH_USER_MODEL` 문자열, 그 외는 `get_user_model()`                              |
| **`has_permission` vs `has_object_permission`?**  | 뷰 진입 전 전역 검사 / `get_object()` 시 **객체 단위** 검사. 목록 뷰·직접 조회에서는 후자가 호출 안 됨    |
| **인증과 인가의 실패 응답 코드는?**                | 인증 실패 401(또는 403), 권한 실패 **403**                                                            |
| **모델 권한으로 "내 글만 수정"을 구현할 수 있나?** | 불가. 모델 권한은 모델 단위 → **객체 권한**은 `has_object_permission`으로 직접 구현                     |
