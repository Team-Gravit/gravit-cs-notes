## IoC/DI와 빈 생명주기

**제어의 역전(IoC, Inversion of Control)**과 **의존성 주입(DI, Dependency Injection)**은 객체의 생성·조립·소멸 책임을 개발자 코드에서 스프링 컨테이너로 넘기는 설계 원칙으로, 스프링의 모든 기능(AOP·트랜잭션·시큐리티)이 이 위에서 동작한다. 컨테이너가 빈(Bean)을 어떤 순서로 만들고 연결하고 폐기하는지 알아야 순환 참조·프록시·스코프 문제를 원인부터 설명할 수 있다.

<br>

### 1. IoC와 DI — 왜 제어를 넘기는가

- 전통적인 코드는 `new OrderService(new OrderRepository())`처럼 **사용하는 쪽이 의존 객체를 직접 생성**함 → 구현체가 바뀌면 사용하는 코드도 함께 수정해야 함
- IoC는 "누가 객체를 만들고 연결하는가"의 **제어 흐름을 컨테이너로 역전**시키는 원칙이고, DI는 그 원칙을 구현하는 구체적 기법임 (생성자·세터·필드로 의존 객체를 **밖에서 넣어 줌**)
- DI의 효과: 인터페이스에만 의존하므로 구현 교체가 쉬움, 테스트에서 가짜 객체(Mock)를 주입하기 쉬움, 객체 조립 코드가 한 곳(설정)에 모임

```java
// 안티패턴: 사용하는 쪽이 구현체를 직접 생성 → 결합도 높음, 테스트 시 교체 불가
public class OrderService {
    private final OrderRepository repository = new JdbcOrderRepository();
}

// 개선: 인터페이스에 의존하고 구현체는 컨테이너가 주입
@Service
public class OrderService {
    private final OrderRepository repository;

    public OrderService(OrderRepository repository) {   // 생성자가 하나면 @Autowired 생략 가능 (Spring 4.3+)
        this.repository = repository;
    }
}
```

> 💡 "IoC와 DI의 차이"는 단골 질문이다. IoC는 **원칙(무엇)**, DI는 **기법(어떻게)**이라고 구분하고, DI 외에도 서블릿 컨테이너가 `doGet()`을 호출하는 것, 템플릿 메서드 패턴처럼 프레임워크가 내 코드를 호출하는 구조 모두가 IoC라고 답하면 된다.

<br>

### 2. ApplicationContext — 스프링 컨테이너

**BeanFactory**는 빈을 생성·조회하는 최소 기능의 컨테이너이고, **ApplicationContext**는 BeanFactory를 상속하면서 메시지 국제화·이벤트 발행·환경 변수(`Environment`)·리소스 로딩·AOP 통합까지 갖춘 실무용 컨테이너다. 스프링 부트의 `SpringApplication.run()`이 반환하는 것도 ApplicationContext다.

```
① 설정 읽기        @Configuration / @ComponentScan / 자동 설정(AutoConfiguration)
        ↓
② BeanDefinition   빈의 "설계도" 등록 (클래스·스코프·의존 관계·초기화 메서드)
        ↓
③ BeanFactoryPostProcessor   설계도 수정 (예: ${...} 프로퍼티 치환)
        ↓
④ 빈 인스턴스화 → 의존성 주입 → 초기화 콜백   (싱글톤은 컨텍스트 기동 시 전부 미리 생성)
        ↓
⑤ 사용 (getBean / 주입)  →  컨텍스트 종료 시 소멸 콜백
```

- 싱글톤 빈은 기본적으로 **기동 시점에 미리 생성(pre-instantiation)**되므로 설정 오류·순환 참조를 애플리케이션 시작 단계에서 즉시 발견할 수 있음
- `@Lazy`를 붙이면 최초 조회 시점으로 생성을 미룰 수 있으나, 오류 발견 시점도 함께 늦어짐

<br>

### 3. 빈 스코프(Scope)

| **스코프**      | **생존 범위**                       | **사용 환경**         | **비고**                                        |
| --------------- | ----------------------------------- | --------------------- | ----------------------------------------------- |
| **singleton**   | 컨테이너당 **인스턴스 1개** (기본값) | 모든 환경             | 상태(필드)를 가지면 스레드 안전성 문제 발생     |
| **prototype**   | 조회할 때마다 **새 인스턴스**       | 모든 환경             | 컨테이너는 생성·주입까지만 관리, **소멸 콜백 호출 안 함** |
| **request**     | HTTP 요청 하나                      | 웹                    | 요청별 데이터(로그 추적 ID 등) 보관             |
| **session**     | HTTP 세션 하나                      | 웹                    | 로그인 사용자 정보 등                            |
| **application** | ServletContext 하나                 | 웹                    | 싱글톤과 유사하나 서블릿 컨텍스트 단위          |

싱글톤 빈이 프로토타입·request 빈을 주입받으면 **주입 시점에 한 번 결정된 인스턴스가 계속 재사용**되어 스코프가 무의미해진다. 이를 해결하려면 `ObjectProvider`로 조회 시점을 늦추거나, `@Scope(proxyMode = ScopedProxyMode.TARGET_CLASS)`로 프록시를 주입받아 실제 호출 시점에 진짜 빈을 찾게 한다.

```java
@Component
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestLogger {
    private final String traceId = UUID.randomUUID().toString();
    public String traceId() { return traceId; }
}

@Service
public class OrderService {
    private final RequestLogger logger;   // 실제로는 프록시가 주입됨 → 요청마다 다른 빈에 위임
    public OrderService(RequestLogger logger) { this.logger = logger; }
}
```

> ⚠️ 싱글톤 빈에 요청별 상태를 **인스턴스 필드**로 저장하면 여러 스레드가 같은 객체를 공유하므로 데이터가 뒤섞인다. 싱글톤 빈은 무상태(stateless)로 설계하고, 요청 단위 데이터는 지역 변수·파라미터·request 스코프 빈으로 다룬다.

<br>

### 4. 빈 생명주기 콜백

```
인스턴스화 (생성자 호출)
   ↓
의존성 주입 (필드 / 세터)
   ↓
Aware 인터페이스 콜백 (BeanNameAware, ApplicationContextAware ...)
   ↓
BeanPostProcessor.postProcessBeforeInitialization()
   ↓
@PostConstruct  →  InitializingBean.afterPropertiesSet()  →  @Bean(initMethod)
   ↓
BeanPostProcessor.postProcessAfterInitialization()   ← AOP 프록시가 여기서 원본을 감싸 교체됨 (unit02 참고)
   ↓
사용
   ↓
@PreDestroy  →  DisposableBean.destroy()  →  @Bean(destroyMethod)   (컨텍스트 종료 시)
```

- 초기화 작업(캐시 예열, 외부 연결)은 **의존성 주입이 끝난 뒤** 실행돼야 하므로 생성자가 아니라 `@PostConstruct`에서 수행함
- `@PostConstruct`/`@PreDestroy`는 Spring 6·Boot 3부터 `jakarta.annotation` 패키지를 사용함 (`javax.annotation` 아님)
- 스프링 부트는 `@Bean` 메서드의 반환 객체에 `close()`나 `shutdown()`이 있으면 이를 소멸 메서드로 **자동 추론**함

> 💡 AOP 프록시가 `postProcessAfterInitialization` 단계에서 만들어진다는 사실은, "생성자 안에서는 왜 `@Transactional`이 동작하지 않는가", "왜 프록시가 아닌 원본 객체가 `this`인가" 같은 질문의 근거가 된다.

<br>

### 5. 의존성 주입 방식과 생성자 주입을 쓰는 이유

| **항목**              | **생성자 주입**                     | **세터 주입**              | **필드 주입**                     |
| --------------------- | ----------------------------------- | -------------------------- | --------------------------------- |
| **불변성**            | `final` 가능 → **불변 보장**         | 불가                       | 불가                              |
| **필수 의존성 강제**  | 객체 생성 시점에 **강제**            | 누락 가능                  | 누락 시 런타임 NPE                |
| **순환 참조 감지**    | **기동 시점에 즉시 실패**            | 런타임까지 잠복            | 런타임까지 잠복                   |
| **테스트 용이성**     | `new`로 직접 생성 가능              | 세터 호출 필요             | 리플렉션 필요 (컨테이너 의존)     |
| **권장 여부**         | **권장 (공식 문서 기준)**            | 선택적 의존성에 한정       | 테스트 코드 외 비권장             |

생성자 주입을 기본으로 쓰는 이유는 결국 **"잘못된 상태의 객체가 존재할 수 없게 만든다"**로 요약된다. 필드 주입은 `@Autowired`만 붙이면 편하지만, 의존 객체 없이도 인스턴스가 생성되므로 테스트에서 NPE가 나기 쉽고, 순환 참조를 컨테이너가 조용히 허용해 설계 문제를 감춘다.

```java
// 필드 주입 (비권장): 의존성이 없는 상태로 생성 가능, 테스트 시 리플렉션 필요
@Service
public class PaymentService {
    @Autowired private PaymentGateway gateway;
}

// 생성자 주입 (권장): final로 불변, 롬복 @RequiredArgsConstructor로 생성자 생략 가능
@Service
@RequiredArgsConstructor
public class PaymentService {
    private final PaymentGateway gateway;
}
```

<br>

### 6. 순환 참조(Circular Reference)

A가 B를, B가 A를 필요로 하는 상태다. 주입 방식에 따라 컨테이너의 대응이 다르다.

```
생성자 주입:  A 생성 시도 → B 필요 → B 생성 시도 → A 필요 → A는 "생성 중" → BeanCurrentlyInCreationException
세터/필드 주입: A 인스턴스화(주입 전) → 미완성 A 참조를 임시 저장소에 등록 → B 생성 → B에 미완성 A 주입 → A에 B 주입 → 완료
```

- 세터·필드 주입은 "인스턴스화"와 "주입"이 분리돼 있어 **미완성 객체의 참조(early reference)**를 먼저 넘겨주는 방식으로 순환을 풀 수 있음
- 생성자 주입은 인스턴스화 자체에 의존 객체가 필요하므로 풀 수 없고, 기동 시점에 예외로 드러남
- **Spring Boot 2.6부터 순환 참조는 기본적으로 금지**되며(`spring.main.allow-circular-references=false`), 필드 주입이라도 기동에 실패함

> ⚠️ `@Lazy`를 한쪽에 붙이거나 `allow-circular-references=true`로 켜면 기동은 되지만, 두 클래스가 서로의 책임을 나눠 갖고 있다는 **설계 신호를 무시하는 것**이다. 공통 로직을 제3의 클래스로 추출하거나, 이벤트(`ApplicationEventPublisher`)로 한 방향 의존을 끊는 것이 정석이다.

<br>

### 7. 면접·실무 체크포인트

| **질문**                                     | **핵심 답변**                                                                                   |
| -------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| **IoC와 DI의 차이는?**                       | IoC는 제어 흐름을 프레임워크로 넘기는 **원칙**, DI는 의존 객체를 외부에서 넣어 주는 **구현 기법** |
| **BeanFactory와 ApplicationContext 차이는?** | ApplicationContext는 BeanFactory + 이벤트·국제화·환경·AOP 통합, 싱글톤 **미리 생성**            |
| **생성자 주입을 권장하는 이유는?**           | 불변성(`final`), 필수 의존성 강제, **순환 참조 조기 발견**, 컨테이너 없이 테스트 가능           |
| **순환 참조는 어떻게 해결하는가?**           | `@Lazy`는 임시방편, 책임 분리·이벤트로 **의존 방향을 한쪽으로** 정리하는 것이 정답              |
| **싱글톤 빈에 프로토타입 빈을 주입하면?**    | 한 번 주입된 인스턴스가 고정됨 → `ObjectProvider` 또는 **스코프 프록시**로 조회 시점을 늦춤     |
| **@PostConstruct는 왜 생성자 대신 쓰는가?**  | 생성자 시점엔 의존성 주입이 끝나지 않았기 때문. 프록시 생성은 그보다도 뒤 단계에서 일어남       |
