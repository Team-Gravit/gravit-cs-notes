## ref와 reactive

unit01에서 본 Proxy 기반 반응성을 실제로 사용하는 두 API가 **`ref`**와 **`reactive`**다. 둘은 감싸는 대상과 접근 방식이 다르며, 특히 `.value` **언박싱(unwrapping) 규칙**과 **구조 분해 시 반응성이 끊기는 이유**는 Vue 3 면접의 단골 질문이다.

<br>

### 1. 두 API가 존재하는 이유

Proxy는 **객체만** 감쌀 수 있다. `new Proxy(1, {})`는 에러다. 그래서 숫자·문자열·불리언 같은 원시값을 반응형으로 만들려면 객체로 한 번 감싸야 하고, 그 역할을 하는 것이 `ref`다.

```typescript
import { ref, reactive } from 'vue'

const count = ref(0)                        // { value: 0 } 형태의 반응형 객체
const user = reactive({ name: '김철수', age: 20 })  // 객체를 Proxy로 감쌈

count.value++            // ref는 .value로 접근
user.age++               // reactive는 일반 객체처럼 접근
```

- **`reactive(obj)`**: 객체를 Proxy로 감싸 반환함. 원시값은 감쌀 수 없음
- **`ref(value)`**: `{ value }` 형태의 **RefImpl 객체**를 만들고, `value` 속성에 getter/setter를 두어 추적·트리거함. 객체를 넣으면 내부적으로 `reactive()`로 변환해 보관함

```
ref(0)                          reactive({ age: 20 })
┌──────────────────────┐        ┌────────────────────────┐
│ RefImpl              │        │ Proxy                  │
│  get value() → track │        │  get(key) → track      │
│  set value() → trigger│       │  set(key) → trigger    │
│  _value: 0           │        │  target: { age: 20 }   │
└──────────────────────┘        └────────────────────────┘
      원시값도 OK                        객체만 OK
```

> 💡 `ref`가 `.value`를 요구하는 것은 불편함이 아니라 **필연**이다. 원시값은 참조가 아니라 값으로 복사되기 때문에, "이 값을 가리키는 상자"를 만들어 상자를 주고받아야 변경을 추적할 수 있다.

<br>

### 2. 언박싱(unwrapping) 규칙

`ref`는 상황에 따라 `.value` 없이 자동으로 풀려서 접근되는데, 이 규칙이 **일관되지 않아** 혼란을 준다. 정확히 언제 풀리는지 정리한다.

| **상황**                                   | **`.value` 필요?** | **예시**                                    |
| ------------------------------------------ | ------------------ | ------------------------------------------- |
| `<script>` 안에서 접근                     | **필요**           | `count.value++`                             |
| 템플릿의 **최상위** ref                    | 불필요 (자동 언박싱) | `{{ count }}`                               |
| 템플릿의 **중첩** ref (객체 속성)          | **필요**           | `{{ obj.count.value }}`                     |
| `reactive` 객체의 속성으로 담긴 ref        | 불필요 (자동 언박싱) | `state.count`                               |
| `reactive` **배열·Map** 안의 ref           | **필요**           | `list[0].value`                             |
| `watch`·`computed` 콜백 안에서 접근        | **필요**           | `watch(count, v => ...)`는 소스로만 전달    |

```typescript
import { ref, reactive } from 'vue'

const count = ref(0)
const state = reactive({ count })    // reactive 안에 ref를 넣음
state.count++                        // 자동 언박싱 → count.value도 1
console.log(count.value === state.count)  // true — 같은 ref를 공유

const list = reactive([ref(1)])
console.log(list[0].value)           // 배열 안에서는 언박싱되지 않음
```

> ⚠️ 템플릿에서 `{{ obj.count + 1 }}`처럼 **일반 객체(`reactive`가 아닌) 안의 ref**를 연산하면 `[object Object]1`이 된다. 최상위 ref만 자동 언박싱된다는 규칙을 기억하고, 필요하면 구조 분해로 최상위로 끌어올린다(`const { count } = obj`는 ref 자체를 꺼내므로 이 경우는 안전함).

<br>

### 3. 구조 분해 시 반응성이 끊기는 이유

가장 흔한 실수다. 반응성은 **"Proxy를 통해 속성에 접근하는 행위"**를 가로채서 동작하는데, 구조 분해는 그 접근을 **딱 한 번** 실행하고 결과를 **일반 변수에 복사**한다. 그 이후로는 Proxy를 거치지 않으므로 추적도 트리거도 일어나지 않는다.

```typescript
import { reactive } from 'vue'

const state = reactive({ count: 0 })

// 안티패턴: 구조 분해 → count는 그냥 숫자 0의 복사본
let { count } = state
count++                 // 일반 변수 증가. state.count는 여전히 0, 화면 갱신 없음

// 함수 인자로 넘길 때도 동일
function useCount(n: number) { /* n은 값 복사본 */ }
useCount(state.count)   // 이 시점의 0만 전달됨
```

```
state (Proxy) ──get('count')──▶ 0  ──복사──▶ count (일반 변수)
                    │                             │
             여기까지만 추적                이후 변경은 Vue가 모름
```

**해결: `toRefs` / `toRef`로 "연결된 상자"를 꺼낸다**

```typescript
import { reactive, toRefs, toRef } from 'vue'

const state = reactive({ count: 0, name: '홍길동' })

const { count, name } = toRefs(state)   // 각 속성을 ref로 변환 (원본과 양방향 연결)
count.value++                           // state.count도 1로 바뀜

const age = toRef(state, 'count')       // 특정 속성 하나만 연결
```

`toRefs`가 만든 ref는 내부적으로 `get value() { return state.count }`처럼 **원본 Proxy를 다시 읽는** 구조이므로 연결이 유지된다. 이것이 컴포저블(unit04 참고)에서 `reactive` 객체를 반환할 때 `toRefs`로 감싸는 이유다.

> 💡 같은 이유로 `reactive` 객체 **전체를 재할당**하는 것도 불가능하다. `state = reactive({...})`는 변수가 새 Proxy를 가리킬 뿐, 템플릿이 붙잡고 있던 옛 Proxy는 그대로다. 교체가 필요하면 `Object.assign(state, newObj)`을 쓰거나 처음부터 `ref`로 감싼다.

<br>

### 4. ref vs reactive 비교와 선택 기준

| **항목**              | **`ref`**                                   | **`reactive`**                               |
| --------------------- | ------------------------------------------- | -------------------------------------------- |
| **감쌀 수 있는 값**   | **모든 값** (원시값·객체·배열)              | 객체·배열·Map·Set만                          |
| **접근 방식**         | `.value` (템플릿 최상위는 자동 언박싱)      | 일반 객체처럼                                |
| **전체 재할당**       | **가능** (`x.value = newObj`)               | 불가능 (연결 끊김)                           |
| **구조 분해**         | ref 자체를 꺼내면 안전                      | 반응성 끊김 → `toRefs` 필요                  |
| **타입 추론**         | `Ref<T>`로 명확                             | 원본 타입 그대로                             |
| **적합한 용도**       | 단일 값, 교체되는 데이터(API 응답 등)       | 서로 묶여 다니는 필드 집합(폼 상태 등)       |

**선택 기준**

- 공식 문서와 대부분의 스타일 가이드는 **`ref`를 기본값**으로 권장함. 원시값·객체 모두 처리할 수 있고, 재할당·구조 분해에서 안전하며, 컴포저블의 반환값으로 일관된 형태를 유지할 수 있기 때문
- `reactive`는 **여러 필드를 한 덩어리로 다루는 로컬 상태**(폼 입력, 좌표 등)에서 `.value`를 줄여 가독성을 높이고 싶을 때 사용함
- 팀 내에서 둘을 섞어 쓸 때는 "언제 `.value`가 필요한가"를 매번 판단해야 하므로, **하나로 통일**하는 것이 실수를 줄임

<br>

### 5. 얕은 반응성 — shallowRef와 shallowReactive

기본 `ref`·`reactive`는 **깊은(deep)** 반응성이다. 중첩된 객체까지 모두 Proxy로 감싸므로 데이터가 크면 비용이 든다. 최상위 변경만 추적하면 충분할 때는 얕은 버전을 쓴다.

```typescript
import { shallowRef, triggerRef } from 'vue'

// 수만 개의 행을 가진 대용량 데이터 — 내부 필드는 추적할 필요 없음
const rows = shallowRef<Row[]>([])

rows.value = await fetchRows()   // .value 교체만 추적됨 → 렌더 1회
rows.value[0].name = '변경'       // 내부 변경은 추적 안 됨 (화면 갱신 없음)
triggerRef(rows)                 // 필요하다면 수동으로 트리거
```

- **`shallowRef`**: `.value` 교체만 추적. 대용량 리스트, 외부 라이브러리 인스턴스 보관에 적합
- **`shallowReactive`**: 최상위 속성 변경만 추적
- **`markRaw`**: 객체에 "절대 반응형으로 만들지 말 것"을 표시. 차트·지도 인스턴스처럼 내부 상태가 방대한 객체에 사용 (unit01 참고)

> ⚠️ 얕은 반응성은 성능 최적화 수단이지 기본값이 아니다. "왜 화면이 안 바뀌지?"의 원인이 `shallowRef` 때문인 경우가 종종 있으므로, 사용 시 변수명이나 주석으로 의도를 드러내는 것이 좋다.

<br>

### 6. Vue 2와의 차이

- Vue 2에는 `ref`·`reactive`가 없고, `data()`가 반환한 객체 전체가 자동으로 반응형이 됨. 원시값도 `data`의 속성으로만 존재하므로 언박싱 개념 자체가 없었음
- Vue 2.7에서 Composition API가 백포트되어 `ref`·`reactive`를 쓸 수 있지만, 내부 구현은 여전히 `Object.defineProperty`이므로 unit01의 감지 한계가 그대로 적용됨
- Vue 3의 `reactive`가 반환하는 것은 **원본과 다른 Proxy**이지만, Vue 2의 `data`는 원본 객체 자체를 변환하므로 `state === raw` 비교 결과가 다름

<br>

### 7. 면접·실무 체크포인트

| **질문**                                              | **핵심 답변**                                                                                         |
| ----------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| **`ref`는 왜 `.value`가 필요한가?**                   | Proxy는 객체만 감쌀 수 있어 원시값을 `{ value }` **상자**에 넣기 때문. 상자를 통해야 추적·트리거가 가능 |
| **템플릿에서 `.value`가 필요 없는 이유는?**           | 컴파일러가 **최상위 ref를 자동 언박싱**함. 중첩된 ref는 풀리지 않음                                   |
| **`reactive` 객체를 구조 분해하면 왜 반응성이 끊기나?** | 구조 분해는 Proxy를 **한 번만 거쳐 값을 복사**하므로 이후 접근이 추적되지 않음. `toRefs`로 해결       |
| **`ref`와 `reactive` 중 무엇을 기본으로 쓰나?**       | **`ref`**. 모든 값을 감쌀 수 있고 재할당·구조 분해에 안전함                                           |
| **`reactive`에 새 객체를 대입하면?**                  | 변수만 새 Proxy를 가리키고 기존 구독은 옛 Proxy에 남아 **연결이 끊김**                                |
| **대용량 데이터의 반응성 비용을 줄이려면?**           | `shallowRef`·`shallowReactive`·`markRaw`로 **깊은 변환을 회피**                                       |

- `ref`는 상자, `reactive`는 Proxy — 둘 다 "접근을 가로채는 통로"를 유지하는 것이 반응성의 조건이다
- 구조 분해·재할당·원본 직접 수정은 모두 **통로를 우회**하므로 반응성이 끊긴다
- 반응형 값을 소비하는 `computed`·`watch`의 동작은 **unit03**을 참고할 것
