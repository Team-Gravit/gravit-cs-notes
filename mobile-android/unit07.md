## 의존성 주입

**의존성 주입(DI, Dependency Injection)**은 객체가 필요한 협력 객체를 직접 만들지 않고 외부에서 받는 설계 원칙이며, **Hilt**는 안드로이드 컴포넌트의 생명주기에 맞춘 **표준 컨테이너·스코프**를 제공하는 Jetpack의 공식 DI 라이브러리다. 스코프 계층, `@Provides`와 `@Binds`의 차이, `@EntryPoint`의 용도를 이해하면 "어디에 어떤 수명으로 객체를 둘 것인가"를 설계할 수 있다.

<br>

### 1. DI가 필요한 이유

- 클래스 안에서 `Retrofit.Builder()...build()`처럼 의존 객체를 직접 생성하면 **생성 방법·수명·교체**가 그 클래스에 묶임
- 테스트에서 네트워크를 가짜로 바꿀 수 없고, 같은 객체를 여러 곳에서 각각 만들어 메모리·설정이 중복됨
- DI는 **생성의 책임을 컨테이너로 옮기고** 클래스는 인터페이스에만 의존하게 만들어 결합도를 낮춤 (테스트 활용은 unit10 참고)

```kotlin
// 안티패턴: 의존성을 직접 생성 → 교체·테스트 불가, 수명 관리 불가
class UserRepository {
    private val api = Retrofit.Builder().baseUrl(BASE_URL).build().create(UserApi::class.java)
}

// 개선: 생성자 주입 → 어떤 구현이든 밖에서 넣어 줄 수 있음
class UserRepository @Inject constructor(private val api: UserApi)
```

> 💡 Hilt는 컴파일 타임에 코드를 생성하는 **Dagger** 위에 안드로이드 전용 규칙(컴포넌트·스코프·진입점)을 얹은 것이다. 런타임 리플렉션이 없어 빠르고, 의존성 누락을 **빌드 시점**에 잡아낸다.

<br>

### 2. Hilt 컴포넌트와 스코프 계층

### 2-1. 컴포넌트 계층 구조

Hilt는 안드로이드 클래스별로 **컴포넌트(컨테이너)**를 미리 정의해 두고, 각 컴포넌트는 부모의 바인딩을 물려받는다.

```
SingletonComponent           (Application 수명)
   │
   ├─ ActivityRetainedComponent   (구성 변경을 넘어 Activity 수명)
   │      │
   │      ├─ ViewModelComponent   (ViewModel 수명)
   │      │
   │      └─ ActivityComponent    (Activity 인스턴스 수명)
   │             │
   │             ├─ FragmentComponent  (Fragment 수명)
   │             │      └─ ViewWithFragmentComponent
   │             └─ ViewComponent
   │
   └─ ServiceComponent           (Service 수명)
```

- 자식은 부모의 바인딩을 **사용할 수 있지만** 부모는 자식의 바인딩을 볼 수 없음
- 모듈은 `@InstallIn(컴포넌트::class)`로 **어느 컴포넌트에 설치할지** 지정함

<br>

### 2-2. 스코프 애너테이션

| **컴포넌트**                    | **스코프 애너테이션**       | **생성 시점**              | **소멸 시점**               | **기본 제공 바인딩**            |
| ------------------------------- | --------------------------- | -------------------------- | --------------------------- | ------------------------------- |
| **SingletonComponent**          | `@Singleton`                | `Application#onCreate`     | 프로세스 종료               | `Application`                   |
| **ActivityRetainedComponent**   | `@ActivityRetainedScoped`   | Activity 최초 생성         | Activity **완전** 종료      | `Application`                   |
| **ViewModelComponent**          | `@ViewModelScoped`          | ViewModel 생성             | ViewModel `onCleared`       | `SavedStateHandle`              |
| **ActivityComponent**           | `@ActivityScoped`           | `Activity#onCreate`        | `Activity#onDestroy`        | `Application`, `Activity`       |
| **FragmentComponent**           | `@FragmentScoped`           | `Fragment#onAttach`        | `Fragment#onDestroy`        | `Application`, `Activity`, `Fragment` |
| **ServiceComponent**            | `@ServiceScoped`            | `Service#onCreate`         | `Service#onDestroy`         | `Application`, `Service`        |

- **스코프가 없는 바인딩**은 주입할 때마다 **새 인스턴스**를 만듦 (기본 동작)
- 스코프를 붙이면 해당 컴포넌트 인스턴스 안에서 **하나만 만들어 공유**함
- `@ActivityRetainedScoped`는 화면 회전 후에도 같은 인스턴스를 유지하고, `@ActivityScoped`는 회전 시 새로 만들어짐 (ViewModel과 Activity의 관계와 동일, unit01 참고)

> ⚠️ **스코프는 필요할 때만 붙인다.** 무조건 `@Singleton`을 붙이면 상태가 있는 객체가 앱 전역에서 공유되어 예상치 못한 상태 공유 버그가 생기고, 메모리도 프로세스 종료까지 해제되지 않는다. "이 객체가 상태를 가지며 공유되어야 하는가"가 판단 기준이다.

<br>

### 3. @Provides와 @Binds

### 3-1. 두 방식의 차이

| **항목**            | **@Provides**                                             | **@Binds**                                            |
| ------------------- | --------------------------------------------------------- | ----------------------------------------------------- |
| **용도**            | **직접 생성 로직이 필요한** 객체 (빌더, 외부 라이브러리)  | **인터페이스 ↔ 구현체 연결**                          |
| **메서드 형태**     | 본문이 있는 일반 함수                                     | `abstract` 함수, 본문 없음                            |
| **모듈 형태**       | `object` 모듈(권장) 또는 클래스                           | `abstract class` 또는 `interface` 모듈                |
| **매개변수**        | 필요한 의존성 여러 개                                     | **정확히 1개**(구현체), 반환 타입이 인터페이스        |
| **생성 코드**       | 팩토리 클래스 생성 → 약간의 오버헤드                      | 코드 생성 최소 → **더 효율적**                        |

<br>

### 3-2. 사용 예시

```kotlin
// @Provides: 우리가 생성자를 소유하지 않는 외부 객체를 만드는 법을 알려 줌
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient =
        OkHttpClient.Builder().addInterceptor(HttpLoggingInterceptor()).build()

    @Provides
    @Singleton
    fun provideUserApi(client: OkHttpClient): UserApi =          // 매개변수는 그래프에서 주입됨
        Retrofit.Builder().baseUrl(BASE_URL).client(client).build().create(UserApi::class.java)
}

// @Binds: 구현체는 @Inject 생성자를 갖고 있고, 인터페이스 타입으로 노출만 하면 됨
interface UserRepository { suspend fun getUser(id: Long): User }

class DefaultUserRepository @Inject constructor(private val api: UserApi) : UserRepository { /* ... */ }

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    @Singleton
    abstract fun bindUserRepository(impl: DefaultUserRepository): UserRepository
}
```

- 선택 기준: **생성자에 `@Inject`를 붙일 수 있으면** 모듈 없이 생성자 주입이 최우선, 인터페이스 연결은 `@Binds`, 그 외(빌더 패턴·서드파티·기본형 설정값)는 `@Provides`
- 같은 타입을 여러 개 바인딩해야 하면 **`@Qualifier`**로 구분함 (`@Named("auth")` 또는 사용자 정의 `@AuthClient`)

<br>

### 4. ViewModel 주입

- ViewModel은 시스템(ViewModelProvider)이 생성하므로 일반 생성자 주입이 불가능함 → `@HiltViewModel` + `@Inject constructor`로 Hilt가 팩토리를 자동 생성함
- `SavedStateHandle`은 `ViewModelComponent`의 기본 바인딩이므로 생성자에 선언만 하면 주입됨

```kotlin
@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository,     // @Binds로 연결된 인터페이스
    private val savedStateHandle: SavedStateHandle,
) : ViewModel() { /* ... */ }

// Activity·Fragment: @AndroidEntryPoint가 있어야 Hilt가 주입 지점을 인식함
@AndroidEntryPoint
class UserActivity : AppCompatActivity() {
    private val viewModel: UserViewModel by viewModels()   // 팩토리를 직접 넘길 필요 없음
}

// Compose: 가장 가까운 ViewModelStoreOwner(NavBackStackEntry 등) 기준으로 획득
@Composable
fun UserRoute(viewModel: UserViewModel = hiltViewModel()) { /* ... */ }
```

- `@ViewModelScoped` 바인딩은 **같은 ViewModel 인스턴스 안에서 공유**되므로 UseCase 등 ViewModel 전용 협력 객체에 적합함
- 런타임 값(예: 화면 인자)을 ViewModel 생성자로 넘기려면 `SavedStateHandle`로 읽거나, 최근 버전의 `@AssistedInject` 지원을 사용함 (지원 여부는 Hilt 버전에 따라 다를 수 있음)

<br>

### 5. @EntryPoint — Hilt가 관리하지 않는 곳에서 꺼내 쓰기

- `@AndroidEntryPoint`를 붙일 수 없는 클래스(`ContentProvider`, 서드파티 라이브러리가 생성하는 객체, 동적 기능 모듈 등)는 필드 주입을 받을 수 없음
- **`@EntryPoint`** 인터페이스를 정의하면 컴포넌트에서 원하는 바인딩을 **직접 꺼내는 접근 창구**가 됨

```kotlin
@EntryPoint
@InstallIn(SingletonComponent::class)
interface AnalyticsEntryPoint {
    fun analytics(): AnalyticsTracker
}

// ContentProvider는 Application보다 먼저 생성되므로 @AndroidEntryPoint 사용 불가
class ExampleContentProvider : ContentProvider() {
    override fun query(/* ... */): Cursor? {
        val entryPoint = EntryPointAccessors.fromApplication(
            context!!.applicationContext, AnalyticsEntryPoint::class.java,
        )
        entryPoint.analytics().log("query")
        return null
    }
}
```

- `EntryPointAccessors.fromApplication`·`fromActivity`·`fromFragment` 등 컴포넌트에 맞는 접근자를 사용함
- **서비스 로케이터**처럼 동작하므로 남용하면 의존성이 코드에 숨겨짐. 정말로 주입이 불가능한 경계 지점에서만 사용해야 함

> 💡 "Hilt에서 ContentProvider나 WorkManager Worker에는 어떻게 주입하나요?"라는 질문에 EntryPoint(그리고 Worker는 `@HiltWorker` + `HiltWorkerFactory`)를 답할 수 있으면 실무 경험을 보여 줄 수 있다.

<br>

### 6. 흔한 함정

- **`@InstallIn` 컴포넌트 불일치**: `ActivityComponent`에 설치한 바인딩을 `@Singleton` 객체에서 요구하면 부모가 자식을 볼 수 없어 빌드 오류가 남. 의존 방향은 항상 **자식 → 부모**
- **Activity Context를 `@Singleton` 객체에 주입**: Activity가 종료돼도 싱글턴이 붙들고 있어 누수됨. `@ApplicationContext`를 사용함
- **`@Binds` 메서드에 본문 작성 또는 매개변수 2개**: 컴파일 오류. 생성 로직이 필요하면 `@Provides`로 전환
- **스코프 없는 무거운 객체**(OkHttpClient, Room DB)를 매번 새로 생성: 커넥션 풀·캐시가 분리되어 성능 저하. 이런 객체는 `@Singleton`이 적절함
- **테스트에서 모듈 교체 누락**: `@TestInstallIn(replaces = [...])`으로 가짜 모듈을 설치해야 실제 네트워크를 타지 않음 (unit10 참고)

<br>

### 7. 정리

| **질문**                                   | **답**                                                                                 |
| ------------------------------------------ | -------------------------------------------------------------------------------------- |
| **스코프의 의미**                          | 해당 컴포넌트 인스턴스 안에서 객체를 하나만 만들어 공유. 없으면 주입마다 새 인스턴스   |
| **`@ActivityRetainedScoped` vs `@ActivityScoped`** | 전자는 회전을 견디고(ViewModel처럼), 후자는 회전 시 새로 생성                  |
| **`@Provides` vs `@Binds`**                | 생성 로직이 필요하면 `@Provides`, 인터페이스-구현체 연결만이면 `@Binds`(더 효율적)     |
| **ViewModel 주입**                         | `@HiltViewModel` + `@Inject constructor`, 화면에서는 `by viewModels()`·`hiltViewModel()` |
| **`@EntryPoint`**                          | `@AndroidEntryPoint`를 못 쓰는 곳에서 컴포넌트의 바인딩을 꺼내는 접근 창구             |
| **의존 방향**                              | 자식 컴포넌트는 부모 바인딩을 사용 가능, 반대는 불가                                   |

- 생성자 주입을 기본으로 하고, 스코프는 **상태 공유가 필요한 객체에만** 붙이며, Activity Context를 긴 수명 객체에 넣지 않음
- Hilt로 의존성을 교체해 테스트하는 방법은 **unit10(테스트 가능한 구조)**를 참고할 것
