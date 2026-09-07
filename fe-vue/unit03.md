## computed와 watch

`computed`와 `watch`는 모두 반응형 상태의 변화에 반응하는 **이펙트**(unit01 참고)지만, 목적이 다르다. `computed`는 **값을 만들어 내고 캐싱**하며, `watch`는 **변화에 대응해 부수 효과(side effect)를 실행**한다. 이 차이와 `watch`·`watchEffect`의 선택 기준을 정확히 설명할 수 있어야 한다.

<br>

### 1. computed — 캐싱되는 파생 값

`computed`는 다른 반응형 상태로부터 **계산된 값(파생 상태)**을 만든다. 핵심은 **캐싱**이다. 의존하는 상태가 바뀌지 않았다면 몇 번을 읽어도 계산을 다시 하지 않는다.

```typescript
import { ref, computed } from 'vue'

const items = ref([
  { name: '노트북', price: 1_200_000, qty: 1 },
  { name: '마우스', price: 30_000, qty: 2 },
])

const total = computed(() =>
  items.value.reduce((sum, i) => sum + i.price * i.qty, 0)
)

console.log(total.value)   // 계산 실행 → 1,260,000
console.log(total.value)   // 캐시 반환, 계산 안 함
items.value[1].qty = 3     // 의존성 변경 → "더티" 표시만 함
console.log(total.value)   // 다음 접근 때 재계산 → 1,290,000
```

**동작 원리**

- `computed`는 내부에 **더티(dirty) 플래그**를 가진 지연 이펙트임
- 의존성이 바뀌면 즉시 재계산하지 않고 플래그만 세움 → **누군가 `.value`를 읽을 때** 비로소 재계산함(지연 평가)
- 계산 도중 읽은 상태가 자동으로 의존성이 되므로 별도로 선언할 필요가 없음

```
items 변경 ──▶ total: dirty = true (계산은 아직)
                     │
템플릿이 total 읽음 ──▶ dirty? → 재계산 → 캐시 갱신 → dirty = false
템플릿이 또 읽음  ──▶ dirty 아님 → 캐시 즉시 반환
```

> 💡 "메서드로 계산해도 되는데 왜 `computed`를 쓰나?"는 단골 질문이다. 템플릿의 메서드 호출은 **렌더링될 때마다 무조건 실행**되지만, `computed`는 **의존성이 바뀌었을 때만** 재계산된다. 리스트 필터링·정렬처럼 비용이 큰 계산일수록 차이가 커진다.

<br>

### 2. computed의 규칙과 함정

**순수 함수여야 한다**

`computed` 안에서는 **상태를 바꾸거나 비동기 요청을 하면 안 된다.** 언제 재계산될지(혹은 안 될지)는 Vue가 결정하므로, 부수 효과를 넣으면 실행 시점이 예측 불가능해진다.

```typescript
// 안티패턴: computed 안에서 상태 변경·비동기 호출
const result = computed(() => {
  loading.value = true          // 다른 상태 변경 → 무한 루프·예측 불가
  fetch('/api')                 // 비동기 부수 효과 → 언제 실행될지 모름
  return data.value
})

// 개선: 파생 값은 computed, 부수 효과는 watch로 분리 (3절 참고)
const result = computed(() => data.value?.filter(d => d.active) ?? [])
```

**쓰기 가능한 computed**

기본적으로 읽기 전용이지만, `get`·`set`을 함께 주면 대입도 가능하다. 양방향 바인딩용 파생 값(예: 전체 이름 ↔ 성·이름)에 쓴다.

```typescript
const first = ref('길동')
const last = ref('홍')

const fullName = computed({
  get: () => `${last.value}${first.value}`,
  set: (v: string) => { last.value = v[0]; first.value = v.slice(1) },
})
fullName.value = '김철수'   // last='김', first='철수'
```

> ⚠️ `computed`가 **반응형이 아닌 값**(일반 변수, `Date.now()`, `Math.random()`)만 읽으면 의존성이 없어 **처음 한 번 계산된 뒤 영원히 갱신되지 않는다.** "computed가 안 바뀐다"는 문제의 대부분은 반응형 값을 읽지 않아서다.

<br>

### 3. watch — 변화에 반응하는 부수 효과

`watch`는 특정 소스를 감시하다가 바뀌면 콜백을 실행한다. 값을 반환하지 않고, **API 호출·로컬 저장소 저장·라우팅** 같은 부수 효과를 담당한다.

```typescript
import { ref, watch } from 'vue'

const keyword = ref('')
const results = ref<string[]>([])

watch(keyword, async (newVal, oldVal) => {
  if (!newVal) { results.value = []; return }
  results.value = await searchApi(newVal)   // 부수 효과: 네트워크 요청
})
```

**감시 소스로 올 수 있는 것**

| **소스 형태**              | **예시**                            | **비고**                                        |
| -------------------------- | ----------------------------------- | ----------------------------------------------- |
| **ref**                    | `watch(count, cb)`                  | `.value` 없이 ref 자체를 전달                   |
| **getter 함수**            | `watch(() => state.count, cb)`      | `reactive`의 **특정 속성**을 볼 때 필수         |
| **reactive 객체**          | `watch(state, cb)`                  | **암묵적으로 deep**, 이전 값 = 새 값(같은 객체) |
| **배열(다중 소스)**        | `watch([a, b], ([a, b]) => ...)`    | 하나라도 바뀌면 실행                            |

> ⚠️ `watch(state.count, cb)`처럼 **reactive의 속성을 직접 넘기면** 숫자 값 하나가 전달될 뿐이라 감시가 되지 않는다(unit02의 구조 분해와 같은 원리). 반드시 `() => state.count` **getter**로 감싸야 한다.

<br>

### 4. watch의 주요 옵션

| **옵션**        | **기본값** | **설명**                                                                   |
| --------------- | ---------- | -------------------------------------------------------------------------- |
| **`immediate`** | `false`    | 등록 즉시 한 번 실행. 초기 데이터 로딩에 사용                              |
| **`deep`**      | `false`    | 중첩 속성 변경까지 감지. 객체를 순회하므로 **비용 큼**                     |
| **`flush`**     | `'pre'`    | 콜백 실행 시점. `'pre'`(렌더 전) / `'post'`(렌더 후, DOM 접근) / `'sync'`(즉시) |
| **`once`**      | `false`    | 한 번만 실행 후 자동 해제 (3.4 이상)                                       |

```typescript
watch(
  () => props.userId,
  async (id) => { user.value = await fetchUser(id) },
  { immediate: true }          // 마운트 직후에도 한 번 실행
)

watch(form, save, { deep: true })   // 폼의 모든 필드 변경을 감시 (남발 주의)
```

- `watch`는 함수를 반환하며, 이를 호출하면 감시가 **해제**됨. 컴포넌트 안에서 동기적으로 등록한 감시자는 언마운트 시 자동 해제되므로 보통 수동 해제는 불필요함
- 콜백 안에서 DOM을 읽어야 한다면 `flush: 'post'`를 주거나 `nextTick`을 기다려야 함(unit07 참고)

<br>

### 5. watchEffect — 의존성을 자동 추적하는 watch

`watchEffect`는 소스를 지정하지 않는다. 콜백을 **즉시 한 번 실행**하면서 그 안에서 읽은 반응형 값을 자동으로 의존성으로 등록하고, 그중 하나라도 바뀌면 다시 실행한다.

```typescript
import { ref, watchEffect } from 'vue'

const userId = ref(1)
const data = ref(null)

watchEffect(async (onCleanup) => {
  const controller = new AbortController()
  onCleanup(() => controller.abort())     // 다음 실행 전·해제 시 이전 요청 취소
  const res = await fetch(`/api/users/${userId.value}`, { signal: controller.signal })
  data.value = await res.json()
})
// userId만 읽었으므로 userId가 바뀔 때만 재실행
```

> 💡 `watchEffect`에서 `await` **뒤에** 읽은 반응형 값은 의존성으로 등록되지 않는다. 동기적으로 실행되는 첫 구간에서 읽은 값만 추적되기 때문이다. 비동기 로직에서는 필요한 값을 `await` 전에 읽어 두어야 한다.

<br>

### 6. watch vs watchEffect 선택 기준

| **항목**              | **`watch`**                                    | **`watchEffect`**                          |
| --------------------- | ---------------------------------------------- | ------------------------------------------ |
| **의존성 지정**       | **명시적** (소스 직접 지정)                    | 암묵적 (콜백에서 읽은 값 자동 추적)        |
| **최초 실행**         | 지연 (`immediate: true`로 변경 가능)           | **즉시 실행**                              |
| **이전 값 접근**      | **가능** (`(newVal, oldVal)`)                  | 불가능                                     |
| **트리거 조건 제어**  | 특정 값이 바뀔 때만 실행하도록 제한 가능       | 읽은 값 중 아무거나 바뀌면 실행            |
| **적합한 상황**       | "A가 바뀌면 B를 해라"처럼 **인과가 명확**할 때 | 여러 값을 조합해 동기화하는 **선언적** 로직 |

**선택 기준**

- **이전 값이 필요하거나, 특정 값의 변경에만 반응**해야 하면 `watch`
- 의존성이 여러 개고 "이 값들이 바뀌면 다시 계산·동기화"가 목적이면 `watchEffect`가 간결함
- 다만 `watchEffect`는 콜백이 커질수록 **무엇에 반응하는지 읽기 어려워지므로**, 팀 코드에서는 `watch`를 기본으로 두고 `watchEffect`는 짧은 동기화 로직에 한정하는 편이 관리하기 쉬움

<br>

### 7. computed vs watch — 무엇을 써야 하는가

| **기준**                         | **`computed`**                          | **`watch` / `watchEffect`**                  |
| -------------------------------- | --------------------------------------- | -------------------------------------------- |
| **목적**                         | 다른 상태에서 **값을 도출**             | 상태 변화에 **부수 효과 실행**               |
| **반환값**                       | 있음 (읽기 전용 ref)                    | 없음 (해제 함수만 반환)                      |
| **캐싱**                         | **있음**                                | 없음                                         |
| **비동기·상태 변경 허용**        | 불가                                    | **가능**                                     |
| **실행 시점**                    | 읽을 때 (지연)                          | 의존성 변경 직후 (flush 옵션 따름)           |

```typescript
// 안티패턴: 파생 값을 watch로 동기화 (중복 상태·초기값 누락 위험)
const fullName = ref('')
watch([first, last], () => { fullName.value = `${last.value}${first.value}` })

// 개선: 파생 값은 computed 하나로 끝낸다
const fullName = computed(() => `${last.value}${first.value}`)
```

> 💡 "이 값이 다른 상태로부터 **계산될 수 있는가**?"가 판단 기준이다. 계산될 수 있으면 `computed`, 계산이 아니라 **행동**(요청·저장·이동)이 필요하면 `watch`다. `watch`로 파생 상태를 만드는 것은 상태를 두 벌로 관리하는 셈이라 버그의 온상이 된다.

<br>

### 8. 면접·실무 체크포인트

- **`computed`와 메서드의 차이**: `computed`는 의존성 기반 **캐싱**, 메서드는 렌더마다 재실행
- **`computed`와 `watch`의 차이**: 전자는 **값 도출(순수)**, 후자는 **부수 효과**. 파생 값을 `watch`로 만들지 말 것
- **`watch`와 `watchEffect`의 차이**: 의존성 **명시 vs 자동 추적**, 이전 값 **접근 가능 vs 불가**, 최초 실행 **지연 vs 즉시**
- **`reactive` 속성을 감시할 때**: 값이 아닌 **getter 함수**를 넘겨야 함
- **`deep: true`**는 객체 전체를 순회하므로 큰 객체에는 비용이 크며, 가능하면 필요한 속성만 getter로 감시할 것
- **DOM에 접근하는 감시자**는 `flush: 'post'` 또는 `nextTick` 필요 (unit07 참고)
- **Vue 2와의 차이**: Options API의 `computed: {}`·`watch: {}` 블록으로 선언하며, 중첩 속성은 문자열 경로(`'user.name'`) 대신 Vue 3에서는 **getter 함수**로 감시함. `watchEffect`는 Vue 3(2.7 백포트 포함)에서 추가됨. Vue 3.4부터 `computed`는 결과가 이전과 같으면 하위 이펙트를 트리거하지 않음 (세부 동작은 버전에 따라 다를 수 있음)
