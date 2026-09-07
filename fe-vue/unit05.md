## 컴포넌트 통신

컴포넌트는 독립적인 단위이므로 서로 데이터를 주고받을 **통로**가 필요하다. Vue는 관계에 따라 **props·emit**(부모-자식), **v-model**(양방향 축약), **provide·inject**(조상-후손), 전역 스토어(무관한 컴포넌트)를 제공하며, 각각의 **적정 범위**를 지키는 것이 유지보수 가능한 컴포넌트 트리의 조건이다.

<br>

### 1. 단방향 데이터 흐름

Vue 컴포넌트 통신의 대원칙은 **"데이터는 아래로, 이벤트는 위로"**다. 부모가 자식에게 props로 데이터를 내려 주고, 자식은 emit으로 "이런 일이 있었다"고 알릴 뿐 **부모의 데이터를 직접 바꾸지 않는다.**

```
        ┌──────────── 부모 ────────────┐
        │  state = ref(...)            │
        │      │ props ▼      ▲ emit   │
        │  ┌───┴───────────────┴───┐   │
        │  │         자식           │   │
        │  │  props 읽기만 / emit   │   │
        │  └────────────────────────┘   │
        └──────────────────────────────┘
   데이터: 위 → 아래 (읽기 전용)    이벤트: 아래 → 위 (변경 요청)
```

- 자식이 props를 직접 수정하면 **데이터의 출처가 두 곳**이 되어 어디서 바뀌었는지 추적할 수 없게 됨
- Vue는 props 수정 시 개발 모드에서 경고를 출력함

<br>

### 2. props와 emit

### 2-1. props — 부모에서 자식으로

`<script setup>`에서는 `defineProps`로 선언한다. TypeScript를 쓰면 **타입 기반 선언**이 가능하고, `withDefaults`로 기본값을 준다.

```vue
<!-- 자식: UserCard.vue -->
<script setup lang="ts">
interface Props {
  user: { id: number; name: string }
  size?: 'sm' | 'lg'
}
const props = withDefaults(defineProps<Props>(), { size: 'sm' })

// 안티패턴: props를 직접 수정 → 경고, 부모와 상태 불일치
// props.user.name = '변경'

// 개선: 초기값으로만 사용하고 로컬 상태로 복사
import { ref } from 'vue'
const localName = ref(props.user.name)
</script>
```

- `defineProps`·`defineEmits`는 컴파일러 매크로라 **import 없이** 사용하며, 반환값 `props`는 반응형 객체임
- `props.user`처럼 접근해야 반응성이 유지되고, `const { user } = props`로 구조 분해하면 끊김(unit02 참고). Vue 3.5부터는 `<script setup>` 안에서 `defineProps` 구조 분해가 반응성을 유지하도록 컴파일되지만, 버전에 따라 다를 수 있으므로 팀 컨벤션을 확인할 것

> ⚠️ 객체·배열 props는 **참조**로 전달되므로, 자식이 `props.user.name = ...`처럼 내부를 바꾸면 경고 없이 부모 데이터가 변한다. 런타임이 막아 주지 않으므로 규율로 지켜야 하며, 필요하면 `readonly()`로 감싸 내려 준다.

<br>

### 2-2. emit — 자식에서 부모로

```vue
<!-- 자식 -->
<script setup lang="ts">
const emit = defineEmits<{
  select: [id: number]                 // 이벤트명: [페이로드 타입]
  remove: [id: number, reason: string]
}>()
</script>

<template>
  <button @click="emit('select', 42)">선택</button>
</template>
```

```vue
<!-- 부모 -->
<UserCard :user="user" @select="onSelect" @remove="onRemove" />
```

- 이벤트 이름은 선언부 `defineEmits`에 명시해야 타입 검사와 자동 완성을 받을 수 있음. 선언하지 않은 이벤트는 `$attrs`로 흘러가 루트 요소에 바인딩되므로 의도치 않은 동작의 원인이 됨
- Vue 2의 `this.$emit`과 개념은 같지만, Vue 3는 `emits` 옵션(또는 `defineEmits`)으로 **선언을 요구**하는 방향으로 바뀜

<br>

### 3. v-model — 양방향 바인딩의 실체

`v-model`은 "props로 내려 주고 emit으로 받아 갱신하는" 패턴의 **문법 설탕**이다. 컴포넌트에 쓰면 다음과 같이 풀린다.

```vue
<!-- 이 두 줄은 동일하다 -->
<SearchInput v-model="keyword" />
<SearchInput :modelValue="keyword" @update:modelValue="keyword = $event" />
```

| **항목**            | **Vue 2**                          | **Vue 3**                                     |
| ------------------- | ---------------------------------- | --------------------------------------------- |
| **기본 prop 이름**  | `value`                            | **`modelValue`**                              |
| **기본 이벤트 이름**| `input`                            | **`update:modelValue`**                       |
| **여러 개 바인딩**  | `.sync` 수식어 (`:title.sync`)     | **`v-model:title`** (`.sync` 제거)            |
| **선언 편의 API**   | 없음 (`model` 옵션으로 이름 변경)  | **`defineModel()`** (3.4 이상 안정화)         |

```vue
<!-- 자식: SearchInput.vue — 수동 구현 -->
<script setup lang="ts">
defineProps<{ modelValue: string }>()
const emit = defineEmits<{ 'update:modelValue': [value: string] }>()
</script>
<template>
  <input :value="modelValue" @input="emit('update:modelValue', ($event.target as HTMLInputElement).value)" />
</template>
```

```vue
<!-- 자식: SearchInput.vue — defineModel (3.4 이상) -->
<script setup lang="ts">
const model = defineModel<string>({ default: '' })   // props + emit을 한 번에 선언
</script>
<template>
  <input v-model="model" />   <!-- model.value 변경 시 자동으로 update:modelValue 발생 -->
</template>
```

> 💡 `defineModel`이 반환하는 ref에 값을 대입해도 **자식이 부모 상태를 직접 바꾸는 것이 아니다.** 내부적으로 `update:modelValue`를 emit하고, 부모가 이를 받아 자기 상태를 갱신하는 단방향 흐름이 유지된다. 이 점을 설명할 수 있으면 "v-model은 단방향 원칙을 깨는가?"라는 질문에 답할 수 있다.

<br>

### 4. provide · inject — 조상에서 후손으로

깊이 중첩된 트리에서 중간 컴포넌트가 쓰지도 않는 props를 단지 아래로 넘기기 위해 받는 것을 **props 드릴링(prop drilling)**이라 한다. `provide`/`inject`는 조상이 값을 제공하면 **깊이에 관계없이** 후손이 꺼내 쓸 수 있게 해 이를 해결한다.

```typescript
// keys.ts — 타입 안전한 주입 키
import type { InjectionKey, Ref } from 'vue'
export const ThemeKey: InjectionKey<Ref<'light' | 'dark'>> = Symbol('theme')
```

```vue
<!-- 조상 -->
<script setup lang="ts">
import { ref, provide, readonly } from 'vue'
import { ThemeKey } from './keys'
const theme = ref<'light' | 'dark'>('light')
provide(ThemeKey, readonly(theme))     // 후손이 수정하지 못하도록 readonly로 제공
</script>
```

```vue
<!-- 깊은 후손 -->
<script setup lang="ts">
import { inject, ref } from 'vue'
import { ThemeKey } from './keys'
const theme = inject(ThemeKey)                      // Ref<'light' | 'dark'> | undefined
const themeSafe = inject(ThemeKey, ref('light'))    // 기본값 지정
</script>
```

**적정 범위 — 언제 써야 하는가**

- **적합**: 테마·로케일·현재 사용자처럼 **트리 전체가 공유하는 읽기 전용 컨텍스트**, 또는 `<Tabs>`/`<Tab>`처럼 **강하게 결합된 컴포넌트 묶음** 내부의 통신
- **부적합**: 일반적인 부모-자식 데이터 전달. `provide`는 **암묵적 의존성**을 만들어 컴포넌트를 단독으로 재사용·테스트하기 어렵게 함
- 후손이 값을 바꿔야 한다면 값 대신 **변경 함수를 함께 제공**해 변경 경로를 조상에 모음

> ⚠️ `provide`로 반응형 객체를 그대로 넘기면 어느 후손이든 수정할 수 있어 데이터 흐름을 추적하기 어렵다. `readonly()`로 감싸 제공하고, 변경은 함께 제공한 함수를 통해서만 하도록 제한하는 것이 안전하다.

<br>

### 5. 무관한 컴포넌트 간 통신

형제 컴포넌트나 트리상 멀리 떨어진 컴포넌트가 상태를 공유해야 하면 위 방법으로는 부족하다.

| **방법**                | **설명**                                              | **적정 범위**                                     |
| ----------------------- | ----------------------------------------------------- | ------------------------------------------------- |
| **상태 끌어올리기**     | 공통 부모로 상태를 올리고 props/emit으로 연결          | 형제 간 단순 공유                                 |
| **Pinia (전역 스토어)** | 앱 전역 상태를 모듈 단위로 관리. Vue 3 공식 권장       | 로그인 정보·장바구니 등 **앱 전역** 상태          |
| **공유 컴포저블**       | 모듈 스코프 `ref`를 컴포저블로 내보내 여러 곳에서 사용 | 소규모 전역 상태, 스토어가 과할 때                |
| **이벤트 버스 (mitt)**  | 발행-구독으로 임의 컴포넌트 간 이벤트 전달             | 최소화. Vue 3에서 `$on/$off`가 제거되어 외부 라이브러리 필요 |

- Vue 2의 Vuex는 Vue 3에서도 동작하지만, 공식 권장 스토어는 **Pinia**임 (mutation 개념 제거, TypeScript 친화적)
- 전역 스토어는 편리한 만큼 **모든 상태를 전역으로 올리는 유혹**이 있음. 두 컴포넌트만 쓰는 상태는 끌어올리기나 provide로 충분함

<br>

### 6. 통신 방법 선택 기준

```
데이터를 주고받을 두 컴포넌트의 관계는?
 ├─ 부모 ↔ 자식 (직접 연결) ─────────▶ props + emit  (양방향이면 v-model)
 ├─ 조상 ↔ 깊은 후손 (같은 서브트리) ─▶ provide + inject (읽기 전용 컨텍스트)
 ├─ 형제 (공통 부모 있음) ───────────▶ 상태 끌어올리기
 └─ 트리상 무관 / 앱 전역 ───────────▶ Pinia (또는 공유 컴포저블)
```

| **방법**            | **방향**        | **명시성** | **결합도** | **대표 용도**                     |
| ------------------- | --------------- | ---------- | ---------- | --------------------------------- |
| **props / emit**    | 부모 ↔ 자식     | **높음**   | 낮음       | 대부분의 컴포넌트 인터페이스      |
| **v-model**         | 부모 ↔ 자식     | 높음       | 낮음       | 입력 컴포넌트, 폼 요소 래핑       |
| **provide / inject**| 조상 → 후손     | 낮음       | 중간       | 테마·로케일, 복합 컴포넌트 내부   |
| **Pinia**           | 전역            | 중간       | 낮음       | 앱 전역 상태                      |

<br>

### 7. 면접·실무 체크포인트

- **단방향 데이터 흐름**: props는 읽기 전용, 변경은 emit으로 부모에게 요청. 객체 props의 내부 수정은 런타임이 막지 않으므로 규율로 지킴
- **v-model의 실체**: `modelValue` prop + `update:modelValue` 이벤트의 문법 설탕. Vue 2는 `value`/`input`, `.sync`는 Vue 3에서 `v-model:인자`로 통합
- **`defineModel`**(3.4 이상)은 선언을 줄여 줄 뿐 **단방향 흐름은 유지**됨
- **provide/inject 적정 범위**: 트리 전체가 공유하는 읽기 전용 컨텍스트와 복합 컴포넌트 내부. 일반 부모-자식 전달에 쓰면 의존성이 숨겨짐. `readonly` + 변경 함수 제공이 안전한 패턴
- **전역 상태**는 Pinia. 단, 두 컴포넌트만 쓰는 상태까지 전역으로 올리지 않음
- 통신에 쓰이는 `ref`·`reactive`의 반응성 규칙은 **unit02**, 컴포저블 설계는 **unit04**를 참고할 것
