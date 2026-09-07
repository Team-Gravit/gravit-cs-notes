## 반응성 시스템 원리

Vue의 **반응성(Reactivity)**은 상태가 바뀌면 그 상태를 사용하는 화면·계산값이 자동으로 갱신되는 메커니즘이다. Vue 2는 `Object.defineProperty`, Vue 3는 `Proxy`로 이를 구현하는데, 두 방식의 차이가 곧 "어떤 변경을 감지할 수 있는가"의 차이로 이어지므로 원리를 정확히 이해해야 한다.

<br>

### 1. 반응성이란 무엇인가

일반 JavaScript에서는 변수를 바꿔도 그 변수로 계산한 값은 자동으로 다시 계산되지 않는다.

```typescript
let a = 1
let b = 2
let sum = a + b   // 3
a = 10
console.log(sum)  // 여전히 3 — a의 변경을 sum이 모른다
```

Vue는 이 문제를 **"누가 이 값을 읽었는지 기록하고(추적), 값이 바뀌면 기록된 쪽을 다시 실행한다(트리거)"**는 두 단계로 해결한다. 이 기록·재실행의 단위를 **이펙트(effect)**라고 부르며, 컴포넌트 렌더 함수·`computed`·`watch`가 모두 이펙트다.

```
          읽기(get)                        쓰기(set)
상태 ──────────────▶ 의존성 기록(track)    상태 변경 ──▶ 등록된 이펙트 재실행(trigger)
  │                                                            │
  └── 렌더 함수 / computed / watch 가 상태를 읽음 ◀────────────┘  (다시 읽으며 재추적)
```

> 💡 핵심은 "값을 읽는 순간을 가로챌 수 있어야 한다"는 점이다. 읽기를 가로채지 못하면 누가 의존하는지 알 수 없고, 쓰기를 가로채지 못하면 언제 갱신할지 알 수 없다. 두 버전의 차이는 **이 가로채기를 어떤 언어 기능으로 하느냐**에서 갈린다.

<br>

### 2. Vue 2 — Object.defineProperty 기반

Vue 2는 `data()`가 반환한 객체의 **모든 속성을 순회하며** 각 속성을 getter/setter로 바꿔 끼운다. getter에서 의존성을 수집하고, setter에서 갱신을 알린다.

```javascript
// Vue 2 반응성의 핵심을 단순화한 코드
function defineReactive(obj, key) {
  let value = obj[key]
  const dep = new Dep()           // 이 속성을 구독하는 이펙트 목록
  Object.defineProperty(obj, key, {
    get() {
      dep.depend()                // 현재 실행 중인 이펙트를 구독자로 등록
      return value
    },
    set(newValue) {
      if (newValue === value) return
      value = newValue
      dep.notify()                // 구독자 전부 재실행
    },
  })
}
```

이 방식은 **속성 단위**로 동작하므로, 초기화 시점에 존재하지 않던 속성은 getter/setter가 붙지 않는다. 그래서 다음과 같은 **감지 한계**가 생긴다.

| **변경 유형**                    | **Vue 2 감지 여부** | **우회 방법**                        |
| -------------------------------- | ------------------- | ------------------------------------ |
| 기존 속성 값 변경                | **감지**            | -                                    |
| **새 속성 추가** `obj.newKey = 1` | 감지 불가           | `Vue.set(obj, 'newKey', 1)`          |
| **속성 삭제** `delete obj.key`   | 감지 불가           | `Vue.delete(obj, 'key')`             |
| **배열 인덱스 대입** `arr[0] = x` | 감지 불가           | `Vue.set(arr, 0, x)` 또는 `splice`   |
| **배열 길이 변경** `arr.length = 0` | 감지 불가        | `arr.splice(0)`                      |
| `Map`·`Set` 내부 변경            | 감지 불가           | 일반 객체·배열로 대체                |

배열 메서드(`push`, `pop`, `splice` 등)는 Vue 2가 **프로토타입을 덮어써서** 감지하도록 별도 처리했다. 즉 "메서드는 되고 인덱스 대입은 안 되는" 비대칭이 있었고, 이것이 Vue 2 개발자를 가장 많이 괴롭힌 함정이었다.

> ⚠️ Vue 2에서 `this.user.age = 20`처럼 **`data`에 선언하지 않은 속성**을 추가하면 화면이 갱신되지 않는다. 이를 몰라서 "값은 바뀌는데 화면이 안 바뀐다"는 질문이 매우 흔했다. 해결책은 `data`에 미리 선언하거나 `this.$set`을 쓰는 것이다.

<br>

### 3. Vue 3 — Proxy 기반

Vue 3는 객체를 하나씩 변환하는 대신, 객체 전체를 감싸는 **Proxy**를 만든다. Proxy는 속성 읽기·쓰기·추가·삭제·`in` 연산·순회까지 모든 동작을 **트랩(trap)**으로 가로챌 수 있다.

```typescript
// Vue 3 reactive()의 핵심을 단순화한 코드
function reactive<T extends object>(target: T): T {
  return new Proxy(target, {
    get(obj, key, receiver) {
      track(obj, key)                              // 의존성 기록
      const result = Reflect.get(obj, key, receiver)
      return typeof result === 'object' && result !== null
        ? reactive(result)                         // 중첩 객체는 접근 시점에 지연 변환
        : result
    },
    set(obj, key, value, receiver) {
      const oldValue = (obj as any)[key]
      const ok = Reflect.set(obj, key, value, receiver)
      if (oldValue !== value) trigger(obj, key)    // 새 속성 추가도 여기서 잡힘
      return ok
    },
    deleteProperty(obj, key) {
      const ok = Reflect.deleteProperty(obj, key)
      trigger(obj, key)                            // 삭제도 감지
      return ok
    },
  })
}
```

**Proxy 방식이 얻는 이점**

- **새 속성 추가·삭제**를 별도 API 없이 감지함 → `Vue.set`/`Vue.delete`가 사라짐
- **배열 인덱스 대입·`length` 변경**도 일반 속성 쓰기와 같은 경로로 감지함
- `Map`·`Set`·`WeakMap`·`WeakSet`도 전용 핸들러로 지원함
- 중첩 객체를 **접근하는 순간에만** Proxy로 감싸는 지연(lazy) 변환이라 초기화 비용이 줄어듦 (Vue 2는 초기화 시 전체 트리를 재귀 순회)

<br>

### 4. 두 방식의 비교

| **항목**                | **Vue 2 (`Object.defineProperty`)**       | **Vue 3 (`Proxy`)**                          |
| ----------------------- | ----------------------------------------- | -------------------------------------------- |
| **감지 단위**           | 개별 속성                                 | **객체 전체**                                |
| **속성 추가·삭제**      | 감지 불가 (`Vue.set`/`Vue.delete` 필요)   | **감지**                                     |
| **배열 인덱스·length**  | 감지 불가 (메서드만 패치)                 | **감지**                                     |
| **Map·Set**             | 미지원                                    | **지원**                                     |
| **초기화 비용**         | 전체 트리 재귀 변환 (즉시)                | 접근 시점 지연 변환                          |
| **원본 객체와의 관계**  | 원본 자체를 변경 (동일 객체)              | **원본과 다른 Proxy 객체**를 반환            |
| **브라우저 지원**       | IE9+                                      | **IE 미지원** (Proxy는 폴리필 불가)          |
| **원시값 반응성**       | `data`의 속성으로만 가능                  | `ref`로 감쌈 (unit02 참고)                   |

<br>

### 5. Proxy 방식의 함정

Proxy가 만능은 아니다. "원본과 다른 객체를 반환한다"는 특성에서 새로운 종류의 실수가 생긴다.

<br>

### 5-1. 원본과 Proxy는 다른 객체다

```typescript
import { reactive } from 'vue'

const raw = { count: 0 }
const state = reactive(raw)

console.log(state === raw)   // false — 서로 다른 객체
raw.count++                  // 원본을 직접 바꾸면 추적되지 않음 → 화면 갱신 없음
state.count++                // Proxy를 통해 바꿔야 감지됨
```

원본 참조를 어딘가 들고 있다가 그쪽을 수정하면 반응성이 끊긴다. 항상 `reactive()`가 **반환한 객체**를 사용해야 한다.

<br>

### 5-2. 구조 분해·재할당, 그리고 의도적으로 제외하는 값

Proxy는 "객체를 통한 접근"을 가로채므로, 속성값을 **밖으로 꺼내 원시값으로 만든 순간** 추적 대상에서 벗어난다. 상세한 원리와 `toRefs`를 이용한 해결책은 unit02에서 다룬다.

**반응형으로 감싸지 않는 값**

- `markRaw()`로 표시한 객체나 클래스 인스턴스처럼 **Proxy로 감싸면 안 되는 대형 객체**(차트·지도 라이브러리 인스턴스 등)는 의도적으로 반응성에서 제외해야 성능이 유지됨
- Vue는 `Object.freeze`로 동결된(확장 불가) 객체를 반응형으로 만들지 않고 그대로 반환함. 변하지 않는 대형 상수 데이터를 의도적으로 추적에서 제외할 때 활용함

> ⚠️ 반응형 객체를 `JSON.stringify`, `structuredClone`, 서드파티 라이브러리에 그대로 넘기면 Proxy 때문에 예상치 못한 동작이나 성능 저하가 생길 수 있다. 이런 경우 `toRaw()`로 원본을 꺼내 전달한다.

<br>

### 6. 갱신은 언제 일어나는가 — 배칭

Vue는 상태가 바뀔 때마다 즉시 렌더링하지 않는다. 같은 틱(tick) 안에서 일어난 변경을 **큐에 모아 한 번만** 렌더링한다(배칭). 따라서 상태를 여러 번 바꿔도 DOM 갱신은 한 번이며, 갱신된 DOM에 접근하려면 `nextTick`을 기다려야 한다(unit07 참고).

```
동기 코드 실행 ──▶ count++ ──▶ count++ ──▶ list.push() ──▶ (동기 종료)
                    │            │             │
                    └────────────┴─────────────┴──▶ 갱신 큐에 1회 등록
                                                        │
                                              마이크로태스크에서 렌더 1회
```

> 💡 Vue 3는 `computed`가 재계산된 뒤 **값이 실제로 바뀐 경우에만** 하위 이펙트를 트리거하도록 개선되었다(3.4 기준). 세부 최적화는 버전에 따라 다를 수 있으므로, 면접에서는 "추적·트리거의 구조"와 "Proxy의 감지 범위"를 중심으로 설명하는 것이 안전하다.

<br>

### 7. 면접·실무 체크포인트

| **질문**                                                | **핵심 답변**                                                                                          |
| ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| **Vue의 반응성은 어떻게 동작하는가?**                   | 읽기 시 의존성을 **추적(track)**하고, 쓰기 시 등록된 이펙트를 **트리거(trigger)**해 재실행함           |
| **Vue 2와 Vue 3의 반응성 차이는?**                      | Vue 2는 `Object.defineProperty`로 **속성 단위** 변환, Vue 3는 `Proxy`로 **객체 전체**를 가로챔         |
| **Vue 2에서 배열 인덱스 대입이 감지되지 않는 이유는?**  | `defineProperty`는 인덱스마다 getter/setter를 붙이지 않았고, 메서드만 프로토타입 패치로 처리했기 때문  |
| **Vue 3에서 반응성이 끊기는 대표 상황은?**              | **원본 객체 직접 수정**, **구조 분해·재할당**, `markRaw`·동결 객체 사용                                |
| **Vue 3가 IE를 지원하지 않는 이유는?**                  | `Proxy`는 **폴리필이 불가능**한 언어 기능이기 때문                                                     |
| **왜 Proxy로 바꿨는가?**                                | 감지 한계 제거(추가·삭제·인덱스·Map/Set), 지연 변환으로 초기화 비용 감소, 코드 단순화                  |

- 반응성의 본질은 **추적과 트리거**이며, 그 구현 수단이 `defineProperty`에서 `Proxy`로 바뀌었다
- Vue 3에서는 `Vue.set`이 필요 없지만, 대신 **원본 ≠ Proxy**라는 새로운 주의점이 생겼다
- `ref`·`reactive`의 세부 규칙은 **unit02**, 이펙트의 대표 사례인 `computed`·`watch`는 **unit03**을 참고할 것
