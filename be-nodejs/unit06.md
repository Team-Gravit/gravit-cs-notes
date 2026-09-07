## 모듈과 프로바이더 스코프

NestJS는 **모듈(Module)** 단위로 코드를 조직하고, **의존성 주입(DI, Dependency Injection)** 컨테이너가 **프로바이더(Provider)**의 생명주기를 관리한다. 프로바이더는 기본적으로 앱 전체에서 하나만 존재하는 싱글톤이지만, 스코프 설정과 모듈 구성에 따라 **인스턴스가 여러 개 생기거나 요청마다 새로 만들어질 수 있다**. 이 유닛은 세 가지 스코프의 동작, 싱글톤이 깨지는 조건, 순환 종속(Circular Dependency) 해결을 다룬다.

<br>

### 1. 모듈과 DI 컨테이너

```typescript
@Module({
  imports: [DatabaseModule],           // 다른 모듈의 export를 가져옴
  controllers: [UsersController],      // HTTP 진입점
  providers: [UsersService, UsersRepository], // 이 모듈이 소유·생성하는 프로바이더
  exports: [UsersService],             // 다른 모듈에서 쓸 수 있게 공개
})
export class UsersModule {}
```

- 앱 부팅 시 Nest는 모듈 그래프를 순회하며 각 프로바이더의 생성자 파라미터(TypeScript 메타데이터)를 읽어 **의존성 그래프**를 만들고 인스턴스를 생성함
- 모듈은 **캡슐화** 단위임. `exports`하지 않은 프로바이더는 다른 모듈에서 주입할 수 없음
- `@Global()` 모듈의 export는 어디서나 주입 가능하지만, 남용하면 의존 관계가 불투명해짐 (설정·로거 정도에만 권장)

**프로바이더 등록 형태**

| **형태**        | **문법**                                            | **용도**                          |
| --------------- | --------------------------------------------------- | --------------------------------- |
| **클래스**      | `UsersService` (= `{ provide: UsersService, useClass: UsersService }`) | 일반적인 서비스                    |
| **값**          | `{ provide: 'CONFIG', useValue: {...} }`             | 설정 객체, 목(mock)                |
| **팩토리**      | `{ provide: 'DB', useFactory: async (cfg) => ..., inject: [ConfigService] }` | 비동기 초기화, 조건부 생성          |
| **별칭**        | `{ provide: 'Alias', useExisting: UsersService }`    | 같은 인스턴스를 다른 토큰으로 노출  |

> 💡 주입 토큰이 인터페이스일 수는 없다. TypeScript 인터페이스는 컴파일 후 사라지므로, 인터페이스 기반 주입은 **문자열·Symbol 토큰 + `@Inject(TOKEN)`** 조합으로 구현한다.

<br>

### 2. 프로바이더 스코프 3종

```typescript
@Injectable({ scope: Scope.REQUEST })   // DEFAULT | REQUEST | TRANSIENT
export class RequestContextService {
  constructor(@Inject(REQUEST) private readonly req: Request) {}
  get userId() { return this.req.headers['x-user-id']; }
}
```

| **스코프**      | **인스턴스 생성 시점**               | **개수**                          | **적합한 경우**                                        |
| --------------- | ------------------------------------ | --------------------------------- | ------------------------------------------------------ |
| **DEFAULT**     | 앱 부팅 시 한 번                      | **앱 전체에 1개** (싱글톤)         | 대부분의 서비스·리포지토리. 상태를 갖지 않거나 공유해도 되는 상태 |
| **REQUEST**     | **HTTP 요청마다** 새로 생성            | 요청 수만큼, 요청 종료 후 GC       | 요청 정보(사용자, 테넌트, 트레이스 ID)를 상태로 가져야 할 때 |
| **TRANSIENT**   | **주입받는 곳마다** 새로 생성          | 소비자 수만큼                      | 소비자별로 독립 상태가 필요한 유틸(로거 컨텍스트 등)     |

- **DEFAULT**: 인스턴스가 하나이므로 클래스 필드는 **모든 요청이 공유**함. 요청별 데이터를 필드에 저장하면 다른 사용자의 데이터가 섞이는 사고가 남
- **REQUEST**: 요청 컨텍스트를 안전하게 담을 수 있지만, 요청마다 DI 서브트리를 생성하므로 **성능 비용**이 있음
- **TRANSIENT**: `A`와 `B`가 같은 TRANSIENT 프로바이더를 주입받으면 서로 다른 인스턴스를 받음. 단, 한 소비자 안에서는 같은 인스턴스가 유지됨

<br>

### 3. 스코프 버블링(Scope Bubbling)

스코프는 의존 그래프를 따라 **위로 전파**된다. 어떤 프로바이더가 REQUEST 스코프 프로바이더에 의존하면, 그 프로바이더도 요청마다 새로 만들어야 하므로 **암묵적으로 REQUEST 스코프가 된다**.

```
RequestContextService (REQUEST)
        ▲ 의존
   UsersService (DEFAULT로 선언했지만 → REQUEST로 승격)
        ▲ 의존
  UsersController (→ REQUEST로 승격, 요청마다 컨트롤러 인스턴스 생성)
```

- 승격은 연쇄적이며, 최상단 컨트롤러까지 요청 스코프가 됨. 요청 하나에 생성되는 객체 수가 크게 늘어남
- 반대로 DEFAULT 프로바이더가 TRANSIENT를 주입받는 것은 문제 없음 (한 번 만들어 계속 사용)
- **Durable 프로바이더**(`durable: true` + `ContextIdStrategy`, Nest 9+)를 쓰면 "테넌트별로 하나" 같은 중간 단위로 요청 스코프 인스턴스를 재사용해 비용을 줄일 수 있음

> ⚠️ 요청 스코프 프로바이더는 **부팅 시 `app.get(UsersService)`로 꺼낼 수 없다**. 요청 컨텍스트가 없기 때문이며, `moduleRef.resolve(UsersService, contextId)`로 컨텍스트를 지정해 얻어야 한다. 테스트 코드에서 자주 겪는 오류다.

<br>

### 4. 싱글톤이 깨지는 경우

"프로바이더는 싱글톤"이라는 전제가 무너지는 상황을 알면 "왜 캐시가 두 벌인가", "왜 카운터가 초기화되는가" 같은 문제를 빠르게 찾을 수 있다.

| **원인**                                   | **현상**                                                 | **해결**                                                       |
| ------------------------------------------ | -------------------------------------------------------- | -------------------------------------------------------------- |
| **같은 클래스를 여러 모듈 `providers`에 등록** | 모듈마다 **별도 인스턴스** 생성 (Nest 싱글톤은 "모듈당" 개념) | 한 모듈에서만 `providers` + `exports`, 나머지는 `imports`       |
| **REQUEST 스코프 의존 (버블링)**            | 싱글톤이라 믿은 서비스가 요청마다 재생성, 필드 캐시가 매번 초기화 | 요청 데이터는 별도 REQUEST 프로바이더나 `AsyncLocalStorage`로 분리 |
| **TRANSIENT 선언**                          | 주입받는 곳마다 다른 인스턴스                              | 의도가 아니면 DEFAULT로                                         |
| **동적 모듈 `forRoot()`를 여러 모듈에서 호출** | 호출마다 새로운 모듈 인스턴스·프로바이더 생성               | 루트 모듈에서 한 번만 `forRoot`, 나머지는 `forFeature`·`@Global` |
| **`useFactory`가 매번 새 객체 반환 (TRANSIENT일 때)** | 팩토리 호출 횟수만큼 생성                                | 팩토리 밖에서 생성해 클로저로 반환                              |

```typescript
// 안티패턴: 두 모듈이 각각 CacheService를 providers에 등록 → 캐시가 두 벌
@Module({ providers: [CacheService], controllers: [UsersController] })
export class UsersModule {}
@Module({ providers: [CacheService], controllers: [OrdersController] })
export class OrdersModule {}

// 개선: 소유 모듈 하나에서 export, 나머지는 import
@Module({ providers: [CacheService], exports: [CacheService] })
export class CacheModule {}
@Module({ imports: [CacheModule], controllers: [UsersController] })
export class UsersModule {}
```

> 💡 Node.js의 모듈 캐시(unit04)는 "파일 단위 싱글톤"이고, Nest의 DI 컨테이너는 "모듈 컨텍스트 단위 싱글톤"이다. 클래스 정의는 하나여도 컨테이너가 두 컨텍스트에서 각각 `new`를 호출하면 인스턴스는 둘이 된다.

<br>

### 5. 순환 종속(Circular Dependency)

`A → B → A`처럼 서로를 주입하면 Nest는 부팅 시 한쪽 클래스가 아직 정의되지 않은(`undefined`) 상태를 만나 `Nest can't resolve dependencies of ...` 오류를 낸다. 파일 간 `import` 순환과 DI 순환이 겹치면 더 헷갈린다.

<br>

### 5-1. 프로바이더 간 순환과 forwardRef

```typescript
// 임시 해결: forwardRef로 "나중에 평가할 참조"를 넘김
@Injectable()
export class UsersService {
  constructor(
    @Inject(forwardRef(() => OrdersService))
    private readonly ordersService: OrdersService,
  ) {}
}

@Injectable()
export class OrdersService {
  constructor(
    @Inject(forwardRef(() => UsersService))
    private readonly usersService: UsersService,
  ) {}
}
```

- 모듈 간 순환(`UsersModule ↔ OrdersModule`)도 `imports: [forwardRef(() => OrdersModule)]`로 **양쪽 모두** 감싸야 함
- `forwardRef`는 "증상 완화"임. 순환은 대개 **책임 분리가 잘못됐다는 신호**이므로 구조 개선을 먼저 검토함

<br>

### 5-2. 구조적으로 해결하기

```
    (순환)                        (개선 1: 공통 추출)          (개선 2: 이벤트로 역전)
Users ──▶ Orders             Users ──▶ Shared ◀── Orders    Users ──emit──▶ EventBus ◀──on── Orders
  ▲          │                                                 Orders는 Users를 모름
  └──────────┘
```

- **공통 모듈 추출**: 둘 다 필요로 하는 로직을 제3의 모듈(`SharedModule`)로 빼고 양쪽이 그것만 의존함
- **이벤트 기반 역전**: 한쪽이 상대를 직접 호출하는 대신 이벤트를 발행(`@nestjs/event-emitter`, `EventEmitter2`)하고 상대가 구독함 → 컴파일 시점 의존이 사라짐
- **호출 방향 단일화**: "주문 서비스가 사용자 서비스를 호출"만 남기고, 반대 방향 호출은 컨트롤러 계층이나 상위 서비스(파사드)로 올림
- **`ModuleRef`로 지연 조회**: 생성자 주입 대신 필요한 시점에 `moduleRef.get(OrdersService)`로 꺼내면 부팅 시 순환이 사라짐 (의존이 숨겨지므로 최후 수단)

> ⚠️ 배럴 파일(`index.ts`)로 재export하면 DI 순환이 없어도 **파일 import 순환**이 생겨 `forwardRef`를 써도 `undefined` 오류가 날 수 있다. 순환 오류가 나면 클래스 참조를 배럴이 아닌 **직접 경로**로 import하는지 먼저 확인한다. `madge --circular` 같은 도구로 파일 순환을 검출할 수 있다.

<br>

### 6. 면접·실무 체크포인트

- 프로바이더 스코프 3종: **DEFAULT(앱당 1개) · REQUEST(요청당 1개) · TRANSIENT(소비자당 1개)**
- REQUEST 스코프는 의존 그래프를 따라 **컨트롤러까지 버블링**되어 요청마다 인스턴스를 만들므로 성능 비용을 감안하고, 필요하면 Durable 프로바이더·`AsyncLocalStorage`로 대체함
- 싱글톤이 깨지는 대표 원인: **여러 모듈에 중복 등록**, **REQUEST 버블링**, **TRANSIENT**, **`forRoot` 중복 호출**
- 싱글톤 서비스의 필드에 요청별 데이터를 저장하면 사용자 간 데이터가 섞임 → 요청 상태는 요청 스코프 객체에만
- 순환 종속은 `forwardRef`로 우회할 수 있지만, **공통 모듈 추출·이벤트 역전·호출 방향 단일화**로 구조를 고치는 것이 정답
- DI 순환과 파일 import 순환(배럴 파일)은 별개 문제이며, 둘 다 확인해야 함
- 요청 처리 중 가드·인터셉터 등이 어떤 순서로 실행되는지는 **unit07** 참고
