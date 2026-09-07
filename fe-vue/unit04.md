## Composition API와 Options API

Vue 컴포넌트를 작성하는 두 가지 스타일이다. **Options API**는 `data`·`methods`·`computed` 같은 **옵션 블록**으로 코드를 나누고, **Composition API**는 `setup` 안에서 **함수 조합**으로 로직을 구성한다. 두 방식의 차이는 문법이 아니라 **로직을 어떻게 재사용하고 어디에 모으느냐**의 관점에서 이해해야 한다.

<br>

### 1. 두 스타일의 형태

같은 카운터 컴포넌트를 두 방식으로 작성하면 다음과 같다.

```typescript
// Options API (Vue 2 · Vue 3 모두 지원)
export default {
  data() {
    return { count: 0 }
  },
  computed: {
    double() { return this.count * 2 },
  },
  methods: {
    increment() { this.count++ },
  },
  mounted() {
    console.log('마운트됨', this.count)
  },
}
```

```vue
<!-- Composition API + <script setup> (Vue 3 권장 방식) -->
<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'

const count = ref(0)
const double = computed(() => count.value * 2)
const increment = () => { count.value++ }

onMounted(() => console.log('마운트됨', count.value))
</script>

<template>
  <button @click="increment">{{ count }} / {{ double }}</button>
</template>
```

- Options API는 `this`를 통해 상태·메서드에 접근하며, Vue가 각 옵션을 인스턴스에 합쳐 줌
- `<script setup>`은 `setup()` 함수의 **컴파일 타임 문법 설탕**임. 최상위에 선언한 변수·함수는 자동으로 템플릿에 노출되며, `return`이 필요 없음
- Composition API는 **Vue 3**의 기본 스타일이며, **Vue 2.7**에 백포트되었고 2.6 이하에서는 `@vue/composition-api` 플러그인이 필요함

<br>

### 2. 관심사가 흩어지는 문제 — Options API의 한계

Options API는 코드를 **"종류"**(데이터인가, 메서드인가)로 나눈다. 컴포넌트가 커지면 하나의 기능(예: 검색)에 관련된 코드가 `data`·`computed`·`watch`·`methods`·`mounted`에 **흩어져** 버린다.

```
Options API                         Composition API
┌─────────────┐                     ┌────────────────────┐
│ data        │ ← 검색·페이징·모달   │ useSearch()        │ ← 검색 관련 전부
├─────────────┤                     ├────────────────────┤
│ computed    │ ← 검색·페이징        │ usePagination()    │ ← 페이징 관련 전부
├─────────────┤                     ├────────────────────┤
│ watch       │ ← 검색·모달          │ useModal()         │ ← 모달 관련 전부
├─────────────┤                     └────────────────────┘
│ methods     │ ← 검색·페이징·모달       기능 단위로 응집
├─────────────┤
│ mounted     │ ← 검색·모달
└─────────────┘
 종류 단위로 분산
```

- 기능 하나를 수정하려면 파일을 위아래로 오가며 여러 블록을 동시에 봐야 함
- 기능 하나를 **다른 컴포넌트로 옮기기**가 어려움 → 재사용 수단으로 mixin이 등장했고, 그것이 다음 문제를 낳음

> 💡 작은 컴포넌트에서는 Options API의 "정해진 자리에 정해진 코드"가 오히려 읽기 쉽다. Composition API의 이점은 **컴포넌트가 커지고 기능이 여러 개 섞일 때** 드러난다. "항상 Composition API가 낫다"보다 "규모와 재사용 요구에 따라 이점이 커진다"고 답하는 것이 정확하다.

<br>

### 3. mixin의 문제점

**mixin**은 Vue 2에서 옵션 블록을 다른 컴포넌트에 **병합**해 로직을 재사용하는 방식이다. 동작은 하지만 구조적 문제가 세 가지 있다.

```javascript
// Vue 2 mixin 예시
const paginationMixin = {
  data() { return { page: 1, size: 20 } },
  computed: { offset() { return (this.page - 1) * this.size } },
  methods: { next() { this.page++ } },
}

export default {
  mixins: [paginationMixin, searchMixin, modalMixin],
  mounted() {
    this.next()     // 어느 mixin에서 온 메서드인가? 파일을 열어봐야 안다
  },
}
```

| **문제**                    | **설명**                                                                                         |
| --------------------------- | ------------------------------------------------------------------------------------------------ |
| **출처 불명확**             | `this.page`가 어느 mixin에서 왔는지 컴포넌트 코드만 봐서는 알 수 없음. mixin이 많을수록 추적 비용 증가 |
| **이름 충돌**               | 두 mixin이 같은 이름의 `data`·`method`를 정의하면 **나중 것이 조용히 덮어씀**. 컴파일 오류 없음  |
| **암묵적 결합**             | mixin이 `this.userId`처럼 **컴포넌트에 있을 것이라 가정한 속성**을 쓰면, 그 의존성이 코드에 드러나지 않음 |
| **타입 추론 불가**          | 병합된 `this`의 타입을 TypeScript가 정확히 추론하기 어려움                                       |

> ⚠️ Vue 3에서도 `mixins` 옵션은 남아 있지만 **하위 호환용**이며, 공식 문서는 신규 코드에서 mixin 대신 컴포저블을 쓸 것을 권장한다. 면접에서 "mixin의 단점"을 물으면 위 표의 **출처 불명확·이름 충돌·암묵적 결합** 세 가지를 예시와 함께 설명하면 된다.

<br>

### 4. 컴포저블 — Composition API의 로직 재사용

**컴포저블(Composable)**은 반응형 상태와 로직을 캡슐화한 **일반 함수**다. 관례상 `use`로 시작하며, 명시적으로 인자를 받고 명시적으로 값을 반환하므로 mixin의 문제가 모두 해결된다.

```typescript
// composables/usePagination.ts
import { ref, computed } from 'vue'

export function usePagination(size = 20) {
  const page = ref(1)
  const offset = computed(() => (page.value - 1) * size)
  const next = () => { page.value++ }
  const prev = () => { if (page.value > 1) page.value-- }
  return { page, offset, next, prev }    // 무엇을 제공하는지 시그니처로 드러남
}
```

```vue
<script setup lang="ts">
import { usePagination } from '@/composables/usePagination'
import { useSearch } from '@/composables/useSearch'

const { page, offset, next } = usePagination(10)       // 출처가 import로 명확
const { keyword, results } = useSearch({ offset })     // 의존성을 인자로 명시
</script>
```

**mixin 대비 이점**

- **출처가 명확함**: `import` 문과 구조 분해 이름으로 어디서 왔는지 바로 보임
- **이름 충돌이 없음**: 반환값을 구조 분해할 때 `const { page: searchPage } = ...`처럼 이름을 바꿀 수 있음
- **의존성이 명시적임**: 필요한 값을 **인자**로 받으므로 암묵적 결합이 사라짐
- **타입이 완전히 추론됨**: 일반 함수의 반환 타입이므로 TypeScript와 자연스럽게 결합함
- **여러 번 호출 가능**: 같은 컴포넌트에서 `usePagination()`을 두 번 호출해 독립된 상태 두 벌을 만들 수 있음 (mixin은 불가)

> 💡 컴포저블이 `reactive` 객체를 반환하면 호출부에서 구조 분해할 때 반응성이 끊긴다(unit02 참고). 그래서 컴포저블은 **`ref`들을 담은 일반 객체**를 반환하거나, `reactive`를 쓸 경우 `toRefs()`로 감싸 반환하는 것이 관례다.

<br>

### 5. 컴포저블 작성 규칙

- 이름은 `useXxx` 형식으로 짓고, 파일은 `composables/` 디렉터리에 둠
- **동기적으로 호출**해야 함. `onMounted`·`watch` 같은 훅 등록은 컴포넌트 `setup` 실행 컨텍스트 안에서만 유효하므로, `await` 뒤나 조건문·비동기 콜백 안에서 컴포저블을 호출하면 훅이 등록되지 않음
- 입력이 반응형일 수 있으면 `ref`·getter 모두 받도록 `toValue()`(3.3 이상)로 정규화하면 활용도가 높아짐
- 컴포저블이 등록한 이벤트 리스너·타이머는 `onUnmounted`에서 정리해 메모리 누수를 막음

```typescript
import { ref, onMounted, onUnmounted } from 'vue'

export function useWindowWidth() {
  const width = ref(window.innerWidth)
  const update = () => { width.value = window.innerWidth }
  onMounted(() => window.addEventListener('resize', update))
  onUnmounted(() => window.removeEventListener('resize', update))   // 정리 책임을 컴포저블이 가짐
  return { width }
}
```

<br>

### 6. 두 API 비교와 선택 기준

| **항목**              | **Options API**                            | **Composition API**                                |
| --------------------- | ------------------------------------------ | -------------------------------------------------- |
| **코드 구성 단위**    | 옵션 종류 (`data`·`methods` …)             | **기능(관심사)**                                   |
| **로직 재사용**       | mixin (출처 불명확·충돌 위험)              | **컴포저블** (명시적 import·반환)                  |
| **`this` 사용**       | 필수                                       | 없음 (클로저 변수)                                 |
| **TypeScript 지원**   | 제한적 (`defineComponent` 필요)            | **자연스러움**                                     |
| **학습 곡선**         | 낮음 (자리가 정해져 있음)                  | 반응성 원리(`ref`·`.value`) 이해 필요              |
| **번들 크기**         | 옵션 병합 코드 포함                        | 트리 셰이킹 유리, 미사용 API 제거                  |
| **주 사용 버전**      | Vue 2, Vue 3 (호환)                        | **Vue 3 기본**, Vue 2.7 백포트                     |

**선택 기준**

- 신규 Vue 3 프로젝트라면 **Composition API + `<script setup>` + TypeScript**가 사실상 표준임
- Vue 2 레거시를 유지보수하거나 팀 전체가 Options API에 익숙하고 컴포넌트가 작다면 Options API도 충분함
- 한 프로젝트에서 두 스타일을 섞으면 코드 리뷰와 온보딩 비용이 커지므로 **하나로 통일**함. 점진적 이전 시에는 새 컴포넌트부터 Composition API로 작성함

<br>

### 7. 면접·실무 체크포인트

| **질문**                                                   | **핵심 답변**                                                                                   |
| ---------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| **Composition API를 도입한 이유는?**                       | 큰 컴포넌트에서 **관심사가 옵션 블록에 흩어지는 문제**와 **mixin의 재사용 한계**를 해결하기 위해 |
| **mixin의 단점은?**                                        | **출처 불명확·이름 충돌(조용히 덮어씀)·암묵적 결합**, 타입 추론 불가                            |
| **컴포저블이 mixin보다 나은 점은?**                        | `import`로 출처 명확, 구조 분해로 이름 변경 가능, **인자로 의존성 명시**, 여러 번 호출 가능     |
| **`<script setup>`은 무엇인가?**                           | `setup()`의 컴파일 타임 문법 설탕. 최상위 바인딩이 자동 노출되고 `return` 불필요                |
| **컴포저블을 `await` 뒤에 호출하면?**                      | setup 컨텍스트가 끝나 **생명주기 훅이 등록되지 않음**. 동기적으로 호출해야 함                   |
| **Options API를 써도 되는가?**                             | 가능함. 작은 컴포넌트·레거시 유지보수에는 적합하지만 신규 프로젝트는 Composition API 권장       |

- Composition API의 핵심 가치는 문법이 아니라 **기능 단위 응집**과 **명시적 재사용**이다
- 컴포저블 작성 시 `ref` 반환·동기 호출·정리 책임을 지킨다
- 컴포넌트 간 데이터 전달은 **unit05**, 생명주기 훅의 세부 동작은 **unit07**을 참고할 것
