## 요청 처리 파이프라인

NestJS는 HTTP 요청이 컨트롤러 핸들러에 도달하기까지 **미들웨어(Middleware) → 가드(Guard) → 인터셉터(Interceptor) → 파이프(Pipe) → 핸들러**를 거치고, 어디서든 예외가 나면 **예외 필터(Exception Filter)**가 받아낸다. 각 구성요소의 역할이 겹쳐 보이기 때문에 "인증은 어디에, 검증은 어디에, 로깅은 어디에" 같은 판단을 위해 **실행 순서와 책임 경계**를 정확히 알아야 한다.

<br>

### 1. 전체 실행 순서

```
요청 ─▶ ① 미들웨어 ─▶ ② 가드 ─▶ ③ 인터셉터(전) ─▶ ④ 파이프 ─▶ ⑤ 핸들러
                                                                 │
응답 ◀──────────────── ⑥ 인터셉터(후) ◀──────────────────────────┘
                              │
     (②~⑥ 어디서든 예외) ────▶ ⑦ 예외 필터 ─▶ 에러 응답
```

| **단계**          | **바인딩 범위 순서**                       | **핵심 역할**                              | **컨텍스트 접근**                    |
| ----------------- | ------------------------------------------ | ------------------------------------------ | ------------------------------------ |
| **① 미들웨어**    | 등록 순서 (전역 → 모듈)                     | 요청 전처리, 로깅, CORS, 바디 파싱          | `req`, `res`만. **어떤 핸들러인지 모름** |
| **② 가드**        | 전역 → 컨트롤러 → 메서드                    | **인가(권한 판단)**, 통과/거부              | `ExecutionContext` (핸들러 메타데이터 가능) |
| **③ 인터셉터(전)** | 전역 → 컨트롤러 → 메서드                    | 핸들러 전 로직, 캐시 히트 시 조기 응답       | `ExecutionContext` + `CallHandler`     |
| **④ 파이프**      | 전역 → 컨트롤러 → 메서드 → 파라미터          | **입력 변환·검증**                          | 파라미터 값과 메타데이터               |
| **⑤ 핸들러**      | —                                          | 비즈니스 로직 호출                          | —                                    |
| **⑥ 인터셉터(후)** | **메서드 → 컨트롤러 → 전역 (역순)**          | 응답 변환, 소요 시간 기록                    | RxJS 스트림                           |
| **⑦ 예외 필터**   | **메서드 → 컨트롤러 → 전역 (구체적 것 우선)** | 예외를 HTTP 응답으로 변환                    | `ArgumentsHost`                       |

> 💡 "가드가 인터셉터보다 먼저"라는 순서가 중요하다. 인증에 실패한 요청은 인터셉터의 로깅·캐싱 로직에 도달하지도 않으며, 파이프의 검증도 실행되지 않는다. 즉 **거부할 요청에는 비용을 쓰지 않는** 구조다.

<br>

### 2. 미들웨어(Middleware)

Express 미들웨어와 동일한 `(req, res, next)` 시그니처다. 라우트 매칭 **이전**에 실행되므로 어떤 컨트롤러·핸들러가 처리할지 알 수 없고, 따라서 데코레이터 메타데이터도 읽을 수 없다.

```typescript
@Injectable()
export class RequestIdMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    req.headers['x-request-id'] ??= randomUUID();
    next(); // 호출하지 않으면 요청이 영원히 멈춤
  }
}

export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(RequestIdMiddleware).forRoutes('*'); // 또는 exclude(...)로 제외
  }
}
```

- 전역 미들웨어는 `app.use()`로 등록하며 함수형만 가능(DI 불가). 클래스 미들웨어는 `configure()`로 모듈에 바인딩해 DI를 사용함
- **적합한 일**: 요청 ID 부여, 원시 바디 파싱, 압축, `helmet` 같은 Express 생태계 통합
- **부적합한 일**: 권한 판단(핸들러 메타데이터를 모름), 응답 변환(핸들러 결과에 접근 불가)

<br>

### 3. 가드(Guard)

`CanActivate` 인터페이스의 `canActivate()`가 `true`를 반환하면 통과, `false`면 `ForbiddenException`(403), 예외를 던지면 그 예외가 필터로 전달된다. 미들웨어와 달리 **`ExecutionContext`**로 "어떤 핸들러가 실행될 예정인지"를 알 수 있어 **데코레이터 기반 인가**가 가능하다.

```typescript
export const Roles = (...roles: string[]) => SetMetadata('roles', roles);

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(ctx: ExecutionContext): boolean {
    const required = this.reflector.getAllAndOverride<string[]>('roles', [
      ctx.getHandler(), ctx.getClass(),   // 메서드 → 클래스 순으로 메타데이터 조회
    ]);
    if (!required) return true;
    const { user } = ctx.switchToHttp().getRequest();
    if (!user) throw new UnauthorizedException(); // 401 — false 반환(403)과 구분
    return required.some((r) => user.roles.includes(r));
  }
}

@Roles('admin')
@UseGuards(AuthGuard, RolesGuard) // 나열 순서대로 실행
@Delete(':id')
remove(@Param('id') id: string) {}
```

- 인증(누구인가, `AuthGuard`)과 인가(할 수 있는가, `RolesGuard`)를 **가드 두 개로 분리**하고 순서대로 배치하는 것이 관례
- 전역 가드를 `app.useGlobalGuards()`로 등록하면 DI를 못 쓰므로, DI가 필요하면 `{ provide: APP_GUARD, useClass: ... }`로 모듈에 등록함

<br>

### 4. 인터셉터(Interceptor)

인터셉터는 **핸들러 실행 전과 후 모두**에 개입하는 유일한 구성요소다. `intercept(ctx, next)`에서 `next.handle()`을 호출하면 RxJS `Observable`이 반환되고, 여기에 연산자를 붙여 응답을 가공한다.

```typescript
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
    const started = Date.now();                       // (전) 핸들러 실행 전
    return next.handle().pipe(                        // 핸들러 호출
      map((data) => ({ success: true, data })),       // (후) 응답 형태 통일
      timeout(5000),                                  // 5초 초과 시 TimeoutError
      catchError((err) =>
        throwError(() => (err instanceof TimeoutError ? new RequestTimeoutException() : err)),
      ),
      tap(() => console.log(`${ctx.getHandler().name} ${Date.now() - started}ms`)),
    );
  }
}
```

- **적합한 일**: 응답 포맷 통일, 실행 시간 로깅, 캐시(`CacheInterceptor`), 타임아웃, 결과에서 민감 필드 제거(`ClassSerializerInterceptor`)
- `next.handle()`을 호출하지 않고 `of(cachedValue)`를 반환하면 핸들러를 건너뛰고 즉시 응답할 수 있음(캐시 히트)
- 후처리 순서는 **역순**임. 전역 → 컨트롤러 → 메서드 순으로 `next.handle()`이 중첩되므로, 응답은 메서드 인터셉터의 `map`을 먼저 거쳐 전역으로 나감

> ⚠️ 인터셉터의 `catchError`는 **핸들러와 파이프에서 난 예외**만 잡는다. 가드는 인터셉터보다 먼저 실행되므로 가드의 예외는 인터셉터를 거치지 않고 곧장 예외 필터로 간다. "모든 에러를 인터셉터에서 로깅"하려다 401·403이 빠지는 실수가 흔하다.

<br>

### 5. 파이프(Pipe)

파이프는 핸들러 **파라미터 단위**로 동작하며 두 가지 일을 한다. **변환**(문자열 `"42"` → 숫자 `42`)과 **검증**(조건 미달이면 `BadRequestException`).

```typescript
// 내장 파이프: 변환 + 검증
@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number) {}   // "abc" → 400 Bad Request

// ValidationPipe + class-validator: DTO 검증 (전역 등록이 일반적)
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,            // DTO에 없는 속성 제거
  forbidNonWhitelisted: true, // 없는 속성이 오면 400
  transform: true,            // 기본 타입·DTO 인스턴스로 변환
}));

export class CreateUserDto {
  @IsEmail() email: string;
  @Length(8, 64) password: string;
}
```

- 실행 순서는 전역 → 컨트롤러 → 메서드 → **파라미터 파이프**이며, 같은 파라미터에 여러 파이프가 있으면 나열 순서대로 값이 흘러감
- `ValidationPipe`는 파라미터의 **타입 메타데이터(DTO 클래스)**를 보고 검증하므로, 타입을 `any`나 인터페이스로 선언하면 검증이 동작하지 않음
- 가드보다 **뒤**에 실행되므로, 인증되지 않은 요청의 바디를 검증하느라 CPU를 쓰지 않음

<br>

### 6. 예외 필터(Exception Filter)

파이프라인 어디서든 던져진 예외는 예외 필터가 받아 HTTP 응답으로 바꾼다. 기본 내장 필터는 `HttpException` 계열은 상태 코드·메시지 그대로, 그 외 예외는 **500 Internal Server Error**로 응답한다.

```typescript
@Catch()   // 인자를 비우면 모든 예외, @Catch(HttpException)처럼 타입 지정 가능
export class AllExceptionsFilter implements ExceptionFilter {
  constructor(private readonly logger: Logger) {}

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const res = ctx.getResponse<Response>();
    const status = exception instanceof HttpException
      ? exception.getStatus()
      : HttpStatus.INTERNAL_SERVER_ERROR;

    if (status >= 500) this.logger.error(exception); // 500만 에러 로그, 4xx는 소음
    res.status(status).json({
      statusCode: status,
      path: ctx.getRequest<Request>().url,
      message: exception instanceof HttpException ? exception.getResponse() : 'Internal server error',
    });
  }
}
// 등록: { provide: APP_FILTER, useClass: AllExceptionsFilter } (DI 사용 가능)
```

- 필터는 **가장 구체적인 것이 우선**함. 메서드에 바인딩된 필터가 잡으면 컨트롤러·전역 필터는 실행되지 않음
- `@Catch(EntityNotFoundError)`처럼 ORM 예외를 도메인 예외로 변환하는 필터를 두면 서비스 계층에서 HTTP 예외를 직접 던지지 않아도 됨
- 미들웨어에서 던진 예외는 라우트 컨텍스트가 없으므로 메서드·컨트롤러 범위 필터에는 도달하지 않음(전역 처리 경로만 적용됨 — 버전에 따라 세부 동작이 다를 수 있음)
- 필터가 잡지 못한 예외, 특히 비동기 콜백 밖에서 발생한 예외는 프로세스 수준 처리로 넘어감 → unit08 참고

<br>

### 7. 선택 기준

| **하고 싶은 일**                        | **적합한 위치**   | **이유**                                                        |
| --------------------------------------- | ----------------- | --------------------------------------------------------------- |
| 요청 ID 부여, 원시 바디 파싱, Express 플러그인 | **미들웨어**      | 라우트와 무관, 가장 먼저 실행                                   |
| 로그인 여부·역할·소유권 확인             | **가드**          | 핸들러 메타데이터 접근, 실패 시 뒤 단계 비용 절감                |
| 응답 포맷 통일, 실행 시간 측정, 캐시, 타임아웃 | **인터셉터**      | 전후 모두 개입, 핸들러 결과 접근                                 |
| DTO 검증, 타입 변환, ID 존재 확인        | **파이프**        | 파라미터 단위, 변환된 값을 핸들러에 전달                         |
| 예외 → 응답 형식 변환, 에러 로깅          | **예외 필터**     | 모든 단계의 예외를 한곳에서 처리                                 |
| 트랜잭션 경계                            | **서비스 계층**   | 파이프라인이 아닌 비즈니스 로직의 책임 (인터셉터로 감싸는 방식도 있으나 범위가 모호해짐) |

> 💡 면접에서 "미들웨어와 가드의 차이"를 물으면 **`ExecutionContext` 유무**로 답한다. 미들웨어는 Express 호환성을 위한 저수준 훅이고, 가드는 어떤 핸들러가 실행될지 알기 때문에 데코레이터 메타데이터 기반의 선언적 인가가 가능하다.

<br>

### 8. 정리

- 실행 순서: **미들웨어 → 가드 → 인터셉터(전) → 파이프 → 핸들러 → 인터셉터(후) → 예외 필터**
- 바인딩 범위: 전처리는 **전역 → 컨트롤러 → 메서드**, 인터셉터 후처리와 예외 필터는 **역순(구체적인 것 우선)**
- 미들웨어는 `req/res`만, 가드·인터셉터·파이프는 `ExecutionContext`로 핸들러 메타데이터에 접근 가능
- 가드는 인가, 파이프는 변환·검증, 인터셉터는 전후 가공, 필터는 예외 응답 변환으로 **책임을 분리**
- 가드 실패 시 인터셉터·파이프는 실행되지 않으며, 가드의 예외는 인터셉터의 `catchError`를 거치지 않음
- 전역 구성요소에 DI가 필요하면 `APP_GUARD`·`APP_INTERCEPTOR`·`APP_PIPE`·`APP_FILTER` 토큰으로 등록함
- 프로바이더 스코프가 이 구성요소들에도 적용된다는 점(요청 스코프 가드 등)은 **unit06** 참고
