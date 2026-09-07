## 네트워크 계층

안드로이드 앱의 HTTP 통신은 사실상 **OkHttp + Retrofit** 조합으로 표준화되어 있다. 두 라이브러리는 자주 함께 쓰이지만 역할이 명확히 다르며, **인터셉터(Interceptor)**로 인증·로깅·재시도를 어디에 끼워 넣을지, 네트워크 오류를 UI까지 어떻게 전달할지가 실무 품질을 가른다. 이 유닛은 계층별 역할과 인터셉터 체인의 동작 원리, 재시도·에러 처리 전략을 정리한다.

<br>

### 1. 계층 구조와 역할 분담

```
ViewModel / UseCase
      │  suspend fun getUser(id): Result<User>
      ▼
Repository  ─────────── 도메인 모델 변환, 캐시 정책, 오류 → 도메인 오류 매핑
      │
      ▼
Retrofit  ────────────── 인터페이스 → HTTP 요청 매핑, 직렬화(Converter), suspend/Call 어댑터
      │
      ▼
OkHttp  ──────────────── 커넥션 풀, 인터셉터 체인, 캐시, TLS, HTTP/2, 타임아웃, 재시도
      │
      ▼
소켓 / TLS / TCP  ─────── (network unit09·unit19 참고)
```

| **항목**            | **OkHttp**                                              | **Retrofit**                                                |
| ------------------- | ------------------------------------------------------- | ----------------------------------------------------------- |
| **위치**            | **HTTP 클라이언트** (실제 전송 담당)                    | OkHttp 위의 **타입 안전 REST 어댑터**                        |
| **주요 책임**       | 커넥션 풀·Keep-Alive, TLS, HTTP/2, 캐시, 타임아웃, 인터셉터 | 인터페이스 메서드 ↔ URL·메서드·파라미터 매핑, JSON 직렬화, 코루틴 지원 |
| **입출력 단위**     | `Request` / `Response` (바이트·헤더)                    | 코틀린 함수 호출 / 데이터 클래스                            |
| **없으면?**         | Retrofit이 동작 불가 (Retrofit은 OkHttp에 의존)          | OkHttp만으로도 통신 가능하지만 보일러플레이트 급증           |
| **설정 위치**       | 인증 헤더, 로깅, 재시도, 캐시, 인증서 고정              | Base URL, 컨버터(Moshi·Kotlinx Serialization·Gson), 호출 어댑터 |

> 💡 "Retrofit이 네트워크 통신을 한다"는 표현은 부정확하다. Retrofit은 **인터페이스를 HTTP 요청으로 번역**할 뿐이고, 실제 소켓을 열고 바이트를 보내는 것은 OkHttp다. 면접에서 둘의 관계를 물으면 "Retrofit은 OkHttp의 클라이언트 래퍼이며, 통신 관련 설정(타임아웃·인터셉터)은 전부 OkHttp에 있다"고 답한다.

<br>

### 2. 기본 구성

```kotlin
interface UserApi {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: Long): UserDto          // Retrofit 2.6+: suspend 지원

    @POST("users")
    suspend fun createUser(@Body body: CreateUserRequest): Response<UserDto>  // 상태 코드까지 다루려면 Response<T>
}

val okHttpClient = OkHttpClient.Builder()
    .connectTimeout(10, TimeUnit.SECONDS)
    .readTimeout(30, TimeUnit.SECONDS)
    .addInterceptor(AuthInterceptor(tokenProvider))       // 애플리케이션 인터셉터
    .addInterceptor(HttpLoggingInterceptor().apply {
        level = if (BuildConfig.DEBUG) HttpLoggingInterceptor.Level.BODY
                else HttpLoggingInterceptor.Level.NONE
    })
    .build()

val retrofit = Retrofit.Builder()
    .baseUrl("https://api.example.com/v1/")               // 반드시 '/'로 끝나야 함
    .client(okHttpClient)
    .addConverterFactory(MoshiConverterFactory.create())
    .build()

val userApi: UserApi = retrofit.create(UserApi::class.java)
```

- `OkHttpClient`와 `Retrofit`은 **앱 전체에서 하나만** 만들어 공유한다. 매 요청마다 새로 만들면 커넥션 풀·스레드 풀이 매번 생성되어 성능과 메모리가 나빠진다
- `suspend` 함수는 `Dispatchers.IO`로 자동 전환되므로 호출 측에서 `withContext`를 감쌀 필요가 없다
- `Response<T>`를 반환하면 4xx·5xx도 예외가 아닌 값으로 받고, `T`를 직접 반환하면 실패 시 `HttpException`이 던져진다

<br>

### 3. 인터셉터 체인

**인터셉터**는 요청이 나가고 응답이 들어오는 경로에 끼어들어 **요청을 수정·관찰·재시도·단락(short-circuit)**할 수 있는 훅이다. OkHttp는 인터셉터를 **체인**으로 연결하며, 각 인터셉터는 `chain.proceed(request)`를 호출해 다음 단계로 넘긴다.

```
  addInterceptor()                 addNetworkInterceptor()
  ┌────────────────────┐     ┌─────────────────────────────┐
요청 → 애플리케이션 인터셉터 → [재시도·리다이렉트·캐시·커넥션] → 네트워크 인터셉터 → 서버
응답 ← 애플리케이션 인터셉터 ← [           OkHttp 코어         ] ← 네트워크 인터셉터 ← 서버
```

| **항목**                | **애플리케이션 인터셉터**                        | **네트워크 인터셉터**                              |
| ----------------------- | ------------------------------------------------ | -------------------------------------------------- |
| **호출 횟수**           | 요청당 **정확히 1회** (리다이렉트·재시도 무관)   | 실제 네트워크 왕복마다 (리다이렉트 시 여러 번)     |
| **캐시 응답 시**        | 호출됨                                           | **호출 안 됨** (네트워크를 안 타므로)              |
| **볼 수 있는 것**       | 앱이 만든 원본 요청, 최종 응답                   | OkHttp가 헤더를 추가한 실제 전송 요청, 압축된 원본 응답 |
| **적합한 용도**         | 인증 헤더, 공통 파라미터, 로깅, **재시도 로직**  | 전송 바이트 분석, 네트워크 레벨 진단               |

```kotlin
class AuthInterceptor(private val tokenProvider: TokenProvider) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val original = chain.request()
        val token = tokenProvider.accessToken() ?: return chain.proceed(original)
        val authed = original.newBuilder()
            .header("Authorization", "Bearer $token")
            .build()
        return chain.proceed(authed)
    }
}
```

**Authenticator — 401 전용 갱신 훅**

`Interceptor`와 별도로 OkHttp는 **401 응답을 받았을 때만** 호출되는 `Authenticator`를 제공한다. 토큰 갱신 후 새 요청을 반환하면 OkHttp가 자동으로 재요청한다.

```kotlin
class TokenAuthenticator(private val tokenProvider: TokenProvider) : Authenticator {
    override fun authenticate(route: Route?, response: Response): Request? {
        if (responseCount(response) >= 2) return null                 // 무한 갱신 루프 방지
        val newToken = synchronized(this) { tokenProvider.refreshBlocking() } ?: return null
        return response.request.newBuilder()
            .header("Authorization", "Bearer $newToken")
            .build()
    }

    private fun responseCount(response: Response): Int =
        generateSequence(response) { it.priorResponse }.count()
}
```

> ⚠️ 여러 요청이 동시에 401을 받으면 **토큰 갱신이 병렬로 여러 번** 일어나 서버가 이전 리프레시 토큰을 무효화할 수 있다. `synchronized`나 `Mutex`로 직렬화하고, 락을 얻은 뒤 "이미 다른 스레드가 갱신했는지" 다시 확인해야 한다. 인터셉터 안에서 `runBlocking`으로 suspend 함수를 부르는 것은 OkHttp 스레드 풀을 막으므로 피한다.

<br>

### 4. 재시도 전략

### 4-1. OkHttp가 기본으로 하는 것

- `retryOnConnectionFailure(true)`(기본값): **커넥션 수립 실패, 풀에서 꺼낸 죽은 커넥션** 등 "요청이 서버에 도달하지 않았다고 확신할 수 있는" 경우만 조용히 재시도한다
- 리다이렉트(3xx)는 `followRedirects(true)`(기본값)에 따라 최대 20회까지 자동 추적한다
- **HTTP 5xx·타임아웃 후 재시도는 하지 않는다** — 요청이 서버에 도달했을 수 있으므로 멱등성을 개발자가 판단해야 한다

<br>

### 4-2. 애플리케이션 레벨 재시도

```kotlin
suspend fun <T> retryWithBackoff(
    times: Int = 3,
    initialDelayMs: Long = 500,
    maxDelayMs: Long = 5_000,
    shouldRetry: (Throwable) -> Boolean = { it is IOException },   // 네트워크 계열만 재시도
    block: suspend () -> T
): T {
    var delayMs = initialDelayMs
    repeat(times - 1) {
        try {
            return block()
        } catch (e: Throwable) {
            if (!shouldRetry(e)) throw e
        }
        delay(delayMs + Random.nextLong(0, delayMs / 2))             // 지터로 동시 재시도 분산
        delayMs = (delayMs * 2).coerceAtMost(maxDelayMs)
    }
    return block()                                                    // 마지막 시도는 예외를 그대로 전파
}
```

**재시도 판단 기준**

| **상황**                              | **재시도**            | **이유**                                             |
| ------------------------------------- | --------------------- | ---------------------------------------------------- |
| **`IOException`(연결 끊김, 타임아웃)** | GET 등 멱등 요청만    | 서버 도달 여부 불명 — POST 재시도는 중복 생성 위험    |
| **HTTP 503·429**                      | `Retry-After` 존중 후 | 서버가 일시 과부하임을 명시                          |
| **HTTP 500·502**                      | 제한적으로            | 게이트웨이 오류는 일시적일 가능성                    |
| **HTTP 400·404·422**                  | **하지 않음**         | 클라이언트 오류 — 같은 요청은 같은 실패              |
| **HTTP 401**                          | 토큰 갱신 후 1회      | `Authenticator`가 담당                               |

- 재시도에는 **지수 백오프(Exponential Backoff) + 지터(Jitter)**를 적용해 서버 복구 중 동시 폭주를 막는다
- 서버가 **멱등성 키(Idempotency-Key)**를 지원하면 POST도 안전하게 재시도할 수 있다

<br>

### 5. 에러 처리 — 네트워크 예외를 UI까지 전달하기

Retrofit 호출은 크게 세 부류의 실패를 낸다. 이를 Repository 경계에서 **도메인 오류 타입**으로 변환해 ViewModel이 HTTP 세부 사항을 모르게 한다 (unit09 참고).

```kotlin
sealed interface NetworkError {
    data object NoConnection : NetworkError
    data object Timeout : NetworkError
    data class Http(val code: Int, val message: String?) : NetworkError
    data class Unknown(val cause: Throwable) : NetworkError
}

suspend fun <T> safeApiCall(call: suspend () -> T): Result<T> = try {
    Result.success(call())
} catch (e: HttpException) {                       // 4xx·5xx
    Result.failure(ApiException(NetworkError.Http(e.code(), e.message())))
} catch (e: SocketTimeoutException) {
    Result.failure(ApiException(NetworkError.Timeout))
} catch (e: IOException) {                         // UnknownHost, Connect 등
    Result.failure(ApiException(NetworkError.NoConnection))
} catch (e: CancellationException) {
    throw e                                        // 코루틴 취소는 절대 삼키지 않는다
}
```

- **`CancellationException`을 catch로 삼키면** 화면이 닫혀도 코루틴이 끝나지 않고 결과가 사라진 UI에 전달되려 한다. 항상 다시 던진다
- `HttpException`은 Retrofit이 `T`를 직접 반환할 때만 발생하며, `Response<T>`를 쓰면 `isSuccessful`·`errorBody()`로 직접 분기한다
- 오프라인 판단은 예외에만 의존하지 말고 `ConnectivityManager`의 네트워크 콜백으로 **사전 상태**를 확인해 UI에 오프라인 배너를 띄우는 편이 사용자 경험이 좋다

> 💡 로깅 인터셉터의 `Level.BODY`는 요청·응답 본문을 전부 메모리에 올려 문자열로 만든다. 릴리스 빌드에 남기면 **성능 저하와 토큰·개인정보 노출**로 이어지므로 반드시 `BuildConfig.DEBUG`로 분기한다 (unit10 참고).

<br>

### 6. 정리 — 면접·실무 체크포인트

| **질문**                                              | **핵심 답변**                                                                                   |
| ----------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| **Retrofit과 OkHttp의 역할 차이는?**                  | OkHttp가 **실제 HTTP 클라이언트**, Retrofit은 인터페이스를 요청으로 번역하는 **타입 안전 어댑터** |
| **애플리케이션 vs 네트워크 인터셉터 차이는?**         | 애플리케이션은 요청당 1회·캐시 응답도 통과, 네트워크는 **실제 전송마다** 호출·캐시 시 미호출     |
| **토큰 만료 처리는 어디서?**                          | `Authenticator`(401 전용)에서 갱신 후 재요청, 동시 갱신은 **락으로 직렬화**                       |
| **OkHttp는 어떤 재시도를 자동으로 하는가?**           | 커넥션 실패 등 서버 미도달이 확실한 경우만. **5xx·타임아웃은 앱이 판단**                          |
| **POST를 재시도해도 되는가?**                         | 멱등성이 보장될 때만 (멱등성 키·서버 설계 확인). 기본은 재시도 금지                               |
| **OkHttpClient를 매번 생성하면?**                     | 커넥션 풀·스레드 풀 중복 생성 → 성능·메모리 낭비. **싱글턴**으로 공유                             |

- 통신 설정(타임아웃·인터셉터·재시도)은 OkHttp, API 정의와 직렬화는 Retrofit이 담당한다
- 오류는 Repository 경계에서 **도메인 오류로 변환**해 상위 계층이 HTTP를 모르게 한다 (**unit09** 참고)
- 인증서 고정·평문 통신 차단 등 네트워크 보안은 **unit10**을 참고할 것
