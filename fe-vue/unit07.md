## 생명주기와 DOM 접근 시점

컴포넌트는 생성 → 마운트 → 갱신 → 언마운트 단계를 거치며, 각 단계에 개입할 수 있는 **생명주기 훅(lifecycle hook)**을 제공한다. "DOM은 언제부터 존재하는가"와 "상태를 바꾼 직후 DOM은 왜 아직 옛날 그대로인가"를 이해해야 `onMounted`와 `nextTick`을 정확한 시점에 쓸 수 있다.

<br>

### 1. 생명주기 흐름

```
setup() 실행 ─── (Options API의 beforeCreate·created 시점에 해당)
     │           반응형 상태 준비 완료, DOM 없음
     ▼
onBeforeMount ── 렌더 함수 실행 직전, DOM 없음
     │
     ▼  최초 렌더 → 실제 DOM 생성·삽입
onMounted ────── DOM 접근 가능 (템플릿 ref 채워짐)
     │
     │  ┌──── 상태 변경 ────┐
     ▼  ▼                  │
onBeforeUpdate ─ 재렌더 직전 (DOM은 아직 이전 상태)
     │
     ▼  diff → 실제 DOM 패치
onUpdated ────── 갱신된 DOM 접근 가능
     │
     ▼  부모가 제거 / v-if false / key 변경
onBeforeUnmount  DOM 아직 존재, 정리 작업 시작 가능
     │
     ▼  DOM 제거, 이펙트·감시자 해제
onUnmounted ──── 완전히 제거됨
```

| **훅 (Composition API)** | **Options API (Vue 3)** | **Vue 2 이름**    | **DOM 상태**         | **주 용도**                                  |
| ------------------------ | ----------------------- | ----------------- | -------------------- | -------------------------------------------- |
| `setup()`                | `beforeCreate`·`created` | 동일             | 없음                 | 상태·컴포저블 초기화, 데이터 요청 시작       |
| **`onBeforeMount`**      | `beforeMount`           | 동일              | 없음                 | 거의 사용하지 않음                           |
| **`onMounted`**          | `mounted`               | 동일              | **생성됨**           | DOM 측정, 외부 라이브러리 초기화, 이벤트 등록 |
| **`onBeforeUpdate`**     | `beforeUpdate`          | 동일              | 갱신 전              | 갱신 전 스크롤 위치 등 저장                  |
| **`onUpdated`**          | `updated`               | 동일              | **갱신됨**           | 갱신 후 DOM 의존 작업 (남용 주의)            |
| **`onBeforeUnmount`**    | `beforeUnmount`         | `beforeDestroy`   | 존재                 | 타이머·리스너·구독 해제                      |
| **`onUnmounted`**        | `unmounted`             | `destroyed`       | 제거됨               | 최종 정리                                    |

- Vue 3에서 `destroy` 계열 이름이 **`unmount` 계열로 변경**됨. 의미는 동일함
- `KeepAlive`로 캐시된 컴포넌트는 `onActivated`/`onDeactivated`가 추가로 호출되고, 에러 처리에는 `onErrorCaptured`가 있음

> 💡 `<script setup>` 안의 코드는 `setup()` 시점에 실행된다. 즉 **스크립트 최상위에서 `document.querySelector`나 템플릿 ref를 읽으면 항상 `null`**이다. DOM이 필요한 코드는 반드시 `onMounted` 안으로 옮겨야 한다.

<br>

### 2. 훅 등록 규칙

생명주기 훅은 **`setup` 실행 중에 동기적으로** 호출되어야 현재 컴포넌트 인스턴스에 등록된다. Vue는 "지금 setup 중인 컴포넌트"를 전역 변수로 추적하는데, `await`나 비동기 콜백 뒤에는 이 추적이 끝나 있기 때문이다.

```typescript
import { onMounted } from 'vue'

// 안티패턴: await 뒤에 훅 등록 → 경고와 함께 무시됨
const data = await fetchData()
onMounted(() => { /* 실행되지 않음 */ })

// 개선: 훅은 먼저 동기적으로 등록하고, 비동기 작업은 훅 안이나 별도 함수에서
onMounted(async () => {
  const data = await fetchData()
  // ...
})
```

- 같은 이유로 컴포저블(unit04 참고)도 `setup` 최상위에서 동기적으로 호출해야 내부의 훅이 등록됨
- 같은 훅을 여러 번 등록할 수 있으며, 등록한 순서대로 실행됨. 컴포저블마다 자기 정리 로직을 `onUnmounted`에 넣을 수 있는 이유임

<br>

### 3. 템플릿 ref — DOM 요소에 직접 접근하기

`ref` 속성으로 요소나 자식 컴포넌트에 대한 참조를 얻는다. 이 값은 **마운트 이후에만** 채워진다.

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const inputEl = ref<HTMLInputElement | null>(null)   // 템플릿의 ref="inputEl"과 이름 일치

console.log(inputEl.value)          // null — 아직 DOM 없음

onMounted(() => {
  inputEl.value?.focus()            // 마운트 후 → 실제 요소
})
</script>

<template>
  <input ref="inputEl" type="text" />
</template>
```

- Vue 3.5부터는 `useTemplateRef('inputEl')`로 변수명과 무관하게 참조를 얻을 수 있음 (버전에 따라 다를 수 있음)
- `v-for` 안의 `ref`는 **배열**로 채워지며, 순서가 소스 배열과 일치한다는 보장이 없음
- 자식 컴포넌트를 참조하면 `<script setup>` 컴포넌트는 기본적으로 **닫혀 있어** 내부에 접근할 수 없고, `defineExpose({ ... })`로 노출한 것만 보임

> ⚠️ `v-if`로 감싼 요소의 템플릿 ref는 조건이 참이 되어 **렌더링된 이후**에야 채워진다. `onMounted`에서 조건이 거짓이면 `null`이므로, 조건이 바뀐 직후 접근하려면 다음 절의 `nextTick`이 필요하다.

<br>

### 4. 상태 변경 직후 DOM이 옛날 그대로인 이유

Vue는 상태가 바뀔 때마다 즉시 DOM을 고치지 않는다. 같은 틱에서 일어난 변경을 **큐에 모아 두었다가** 현재 동기 코드가 끝난 뒤 **마이크로태스크**에서 한 번에 렌더링한다(unit01의 배칭). 따라서 상태를 바꾼 **바로 다음 줄**에서 DOM을 읽으면 아직 반영 전이다.

```
동기 코드                                   마이크로태스크
────────────────────────────────────────────┼──────────────────────
count.value++   → 갱신 큐에 등록            │
el.textContent  → 옛 값 (아직 렌더 안 됨)   │
await nextTick()  ──────────────────────────┼─▶ 렌더 실행 → DOM 갱신
el.textContent  → 새 값                     │       └─ nextTick 이후 코드 재개
```

```typescript
import { ref, nextTick } from 'vue'

const count = ref(0)
const el = ref<HTMLElement | null>(null)

async function increment() {
  count.value++
  console.log(el.value?.textContent)   // "0" — 아직 이전 DOM

  await nextTick()                     // 렌더링 완료를 기다림
  console.log(el.value?.textContent)   // "1" — 갱신된 DOM
}
```

- `nextTick()`은 **다음 DOM 갱신 사이클이 끝난 뒤** 해결되는 Promise를 반환함. 콜백 인자로 넘겨도 되지만 `await` 형태가 읽기 쉬움
- 배칭 덕분에 `count.value++`를 100번 해도 렌더는 1번이며, 이것이 Vue가 "상태를 자유롭게 바꿔도 성능이 유지되는" 이유임

<br>

### 5. nextTick이 필요한 대표 상황

| **상황**                                          | **왜 필요한가**                                                 | **대안**                                 |
| ------------------------------------------------- | --------------------------------------------------------------- | ---------------------------------------- |
| **`v-if`로 요소를 켠 직후 포커스·측정**            | 조건이 참이 되어도 DOM은 다음 틱에 생성됨                       | -                                        |
| **리스트에 항목 추가 후 맨 아래로 스크롤**         | 새 항목의 DOM이 아직 없어 `scrollHeight`가 옛 값               | `watch(list, ..., { flush: 'post' })`    |
| **데이터 변경 후 외부 라이브러리 갱신**            | 차트·에디터가 옛 DOM을 기준으로 다시 그림                       | `onUpdated` (모든 갱신에 반응하므로 주의) |
| **테스트에서 상태 변경 후 DOM 검증**               | 렌더 전에 단언하면 항상 실패                                    | 테스트 유틸의 `await nextTick()`         |

```vue
<script setup lang="ts">
import { ref, nextTick } from 'vue'

const editing = ref(false)
const titleInput = ref<HTMLInputElement | null>(null)

// 안티패턴: v-if를 켠 직후 접근 → titleInput.value는 아직 null
function startEditBad() {
  editing.value = true
  titleInput.value?.focus()          // 아무 일도 일어나지 않음
}

// 개선: 렌더링을 기다린 뒤 접근
async function startEdit() {
  editing.value = true
  await nextTick()
  titleInput.value?.focus()          // 정상 동작
}
</script>

<template>
  <input v-if="editing" ref="titleInput" />
  <button v-else @click="startEdit">편집</button>
</template>
```

> 💡 `watch`에 `flush: 'post'`를 주면 콜백이 **DOM 갱신 이후**에 실행되므로 `nextTick`을 매번 쓰지 않아도 된다. "특정 상태가 바뀔 때마다 DOM을 다뤄야 한다"면 이벤트 핸들러의 `nextTick`보다 `flush: 'post'` 감시자가 선언적이고 누락이 적다(unit03 참고).

<br>

### 6. 부모·자식 훅 실행 순서와 정리 책임

```
마운트:  부모 setup → 부모 onBeforeMount → 자식 setup → 자식 onBeforeMount
         → 자식 onMounted → 부모 onMounted            (자식이 먼저 마운트 완료)

언마운트: 부모 onBeforeUnmount → 자식 onBeforeUnmount
         → 자식 onUnmounted → 부모 onUnmounted        (자식이 먼저 제거 완료)
```

- 부모의 `onMounted`에서는 **모든 자식의 DOM이 이미 존재**하므로 자식 요소 측정이 가능함
- 자식이 `onMounted`에서 부모 DOM의 최종 레이아웃(크기 등)에 의존하면 부모가 아직 마운트 완료 전일 수 있어 주의해야 함

**정리(cleanup) 책임**

```typescript
import { onMounted, onBeforeUnmount } from 'vue'

let timer: ReturnType<typeof setInterval>
const onResize = () => { /* ... */ }

onMounted(() => {
  timer = setInterval(poll, 5000)
  window.addEventListener('resize', onResize)
})
onBeforeUnmount(() => {
  clearInterval(timer)                              // 등록한 것은 반드시 해제
  window.removeEventListener('resize', onResize)
})
```

- `watch`·`computed`처럼 Vue가 만든 이펙트는 언마운트 시 **자동 해제**되지만, `setInterval`·전역 이벤트 리스너·WebSocket·외부 라이브러리 인스턴스는 **개발자가 직접 해제**해야 함. 이를 빠뜨리면 컴포넌트가 사라진 뒤에도 콜백이 실행되는 메모리 누수가 생김
- SSR 환경에서는 `onMounted`·`onUnmounted`가 서버에서 실행되지 않으므로, `window`·`document` 접근은 이 훅 안에 두는 것이 안전함

> ⚠️ `onUpdated` 안에서 상태를 변경하면 다시 갱신이 일어나 **무한 루프**에 빠질 수 있다. 갱신 후 DOM 작업이 필요하면 `onUpdated`보다 원인이 되는 상태를 지정한 `watch(..., { flush: 'post' })`가 안전하다.

<br>

### 7. 면접·실무 체크포인트

- **`setup`과 `onMounted`의 차이**: `setup`은 DOM 생성 전(상태 초기화), `onMounted`는 DOM 생성 후(측정·외부 라이브러리·리스너)
- **템플릿 ref는 마운트 후에만 값이 있음**. `<script setup>` 최상위에서 읽으면 `null`
- **`nextTick`이 필요한 이유**: 상태 변경은 **배칭**되어 마이크로태스크에서 렌더되므로, 변경 직후의 DOM은 아직 옛 상태. 대표 상황은 `v-if` 직후 포커스, 추가 후 스크롤
- **`nextTick` 대안**: 상태 기반이면 `watch`의 `flush: 'post'`가 선언적
- **훅은 동기적으로 등록**: `await` 뒤나 콜백 안에서 등록하면 무시됨
- **Vue 2와의 이름 차이**: `beforeDestroy`/`destroyed` → `beforeUnmount`/`unmounted`, `beforeCreate`/`created` → `setup`
- **정리 책임**: 타이머·전역 리스너·구독은 `onBeforeUnmount`/`onUnmounted`에서 직접 해제
- 렌더링 자체의 최적화(`v-if`/`v-show`, `key`)는 **unit06**을 참고할 것
