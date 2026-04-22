---
title: Основные паттерны реактивности (ref, reactive, shallowRef, computed, watch)
impact: MEDIUM
impactDescription: Чёткий выбор реактивных конструкций делает состояние предсказуемым и сокращает лишние обновления в Vue 3 приложениях
type: efficiency
tags: [vue3, reactivity, ref, reactive, shallowRef, computed, watch, watchEffect, external-state, best-practice]
---

# Основные паттерны реактивности (ref, reactive, shallowRef, computed, watch)

**Impact: СРЕДНЕЕ** — Сначала выберите правильную реактивную примитиву, вычисляйте через `computed`,
и используйте наблюдатели только для сайд-эффектов.

Этот справочник охватывает ключевые решения по реактивности для локального состояния, внешних
данных, производных значений и эффектов.

## Список задач

- Правильно объявлять реактивное состояние
  - Выбирать корректный метод объявления для объектов/массивов/Map/Set
- Следовать лучшим практикам для `reactive`
  - Избегать деструктуризации `reactive()` напрямую
  - Передавать свойства из `reactive` в функции без потери реактивности
  - Корректно наблюдать за `reactive`
- Использовать `readonly` для защиты пропсов и сторов от мутаций
- Следовать лучшим практикам для `computed`
  - Предпочитать `computed` вместо наблюдателя, присваивающего значение в ref
  - Выносить фильтрацию/сортировку из шаблонов
  - Использовать `computed` для переиспользуемой логики классов/стилей
  - Держать геттеры computed чистыми (без сайд-эффектов) и помещать сайд-эффекты в наблюдатели
- Следовать лучшим практикам для наблюдателей
  - Использовать `immediate: true` вместо дублирующих начальных вызовов
  - Очищать асинхронные эффекты в наблюдателях

## Правильно объявлять реактивное состояние

### Выбирать корректный метод объявления для объектов/массивов/Map/Set

Используйте `ref()`, когда вы часто **заменяете значение целиком** (`state.value = newObj`) и при
этом нужна глубокая реактивность внутри. Обычно используется для:

- Часто переназначаемого состояния (полученный с сервера объект/список, сброс к значениям по
умолчанию, переключение пресетов).
- Возвращаемых значений композаблов, где обновления происходят в основном через присваивание `.value`.

Используйте `reactive()`, когда вы в основном **мутируете свойства**, а полная замена объекта редка.
Обычно используется для:

- Паттерна «единого объекта состояния» (сторы/формы): `state.count++`, `state.items.push(...)`,
`state.user.name = ...`.
- Ситуаций, когда хочется избежать `.value` и обновлять вложенные поля на месте.

```ts
import { reactive } from 'vue'

const state = reactive({
  count: 0,
  user: { name: 'Alice', age: 30 }
})

state.count++ // ✅ реактивно
state.user.age = 31 // ✅ реактивно
// ❌ избегайте замены ссылки на реактивный объект:
// state = reactive({ count: 1 })
```

Используйте `shallowRef()`, когда значение прозрачно / не должно быть проксировано
(экземпляры классов, объекты внешних библиотек, очень большие вложенные данные) и вы хотите, чтобы
обновления происходили только при замене `state.value` (без глубокого отслеживания). Обычно
используется для:

- Хранения внешних экземпляров/хендлов (SDK-клиенты, экземпляры классов) без проксирования их
внутренностей Vue.
- Больших данных, где обновление происходит через замену корневой ссылки (иммутабельный стиль).

```ts
import { shallowRef } from 'vue'

const user = shallowRef({ name: 'Alice', age: 30 })

user.value.age = 31 // ❌ не реактивно
user.value = { name: 'Bob', age: 25 } // ✅ вызывает обновление
```

Используйте `shallowReactive()`, когда вам нужны реактивными только свойства верхнего уровня;
вложенные объекты остаются сырыми. Обычно используется для:

- Объектов-контейнеров, где меняются только ключи верхнего уровня, а вложенные данные должны
оставаться неуправляемыми/непроксированными.
- Смешанных структур, где Vue отслеживает объект-обёртку, но не глубоко вложенные или внешние
объекты.

```ts
import { shallowReactive } from 'vue'

const state = shallowReactive({
  count: 0,
  user: { name: 'Alice', age: 30 }
})

state.count++ // ✅ реактивно
state.user.age = 31 // ❌ не реактивно
```

## Лучшие практики для `reactive`

### Избегать деструктуризации `reactive()` напрямую

**ПЛОХО:**

```ts
import { reactive } from 'vue'

const state = reactive({ count: 0 })
const { count } = state // ❌ разрыв реактивности
```

### Передавать свойства из reactive в функции без потери реактивности

**ПЛОХО:**

передача примитива или отдельного свойства ломает реактивность.

```ts
import { reactive } from 'vue'

const state = reactive({ user: { name: 'Alice', age: 30 } })

// ❌ userName — это просто строка, реактивность потеряна
const userName = state.user.name
someFunction(userName)

// ❌ даже передача вложенного объекта — это ссылка на прокси,
// но если функция ожидает примитив или заменяет поле, реактивность может сломаться
someOtherFunction(state.user.name)
```

**ХОРОШО:**

используйте `computed` или `toRef` / `toRefs`, чтобы сохранить связь.

```ts
import { reactive, computed, toRef, toRefs } from 'vue'

const state = reactive({ user: { name: 'Alice', age: 30 } })

// ✅ Вариант 1: computed сохраняет реактивность
const userName = computed(() => state.user.name)
someFunction(userName) // функция должна принимать ref или вызывать .value

// ✅ Вариант 2: toRef для одного поля
const userNameRef = toRef(state.user, 'name')
someFunction(userNameRef)

// ✅ Вариант 3: toRefs для всех полей объекта
const { name, age } = toRefs(state.user)
someFunction(name) // name — это ref
```

### Корректно наблюдать за reactive

**ПЛОХО:**

передача не-геттера в `watch()`.

```ts
import { reactive, watch } from 'vue'

const state = reactive({ count: 0 })

// ❌ watch ожидает геттер, ref, реактивный объект или массив из них
watch(state.count, () => { /* ... */ })
```

**ХОРОШО:**

сохраняйте реактивность с помощью `toRefs()` или используйте геттер.

```ts
import { reactive, toRefs, watch } from 'vue'

const state = reactive({ count: 0 })
const { count } = toRefs(state) // ✅ count — это ref

watch(count, () => { /* ... */ }) // ✅
watch(() => state.count, () => { /* ... */ }) // ✅
```
## Использовать readonly для защиты пропсов и сторов от мутаций

Когда вы передаёте реактивные данные наружу (в дочерние компоненты, из сторов, из композаблов),
важно защитить их от нежелательных изменений. `readonly` создаёт реактивную, но доступную только
для чтения обёртку.

### Защита сторов и публичного API

```ts
import { reactive, readonly } from 'vue'

// Внутреннее состояние (можно менять)
const internalState = reactive({
  user: { name: 'Alice', isAdmin: false },
  posts: []
})

// Публичное API — только для чтения
const publicState = readonly(internalState)

function setUserName(name: string) {
  internalState.user.name = name // ✅ разрешено внутри стора
}

// Наружу отдаём readonly
export const useUserStore = () => ({
  state: publicState,
  setUserName
})
```

### Защита пропсов в компонентах

Хотя пропсы сами по себе не должны мутироваться, `readonly` помогает явно обозначить намерение и
избежать случайных изменений.

```vue
<script setup lang="ts">
  import { readonly, computed } from 'vue'

  const props = defineProps<{
    user: { name: string; age: number }
  }>()

  // Явно запрещаем мутации
  const protectedUser = readonly(props.user)

  // Безопасное использование — только через computed
  const displayName = computed(() => protectedUser.name)

  // ❌ Так делать нельзя — будет предупреждение в консоли
  // protectedUser.name = 'Bob'
</script>

<template>
  <div>{{ displayName }}</div>
</template>
```

### `readonly` с composable-функциями

```ts
import { ref, readonly } from 'vue'

function useCounter() {
  const count = ref(0)
  
  function increment() {
    count.value++
  }
  
  // Отдаём только для чтения
  return {
    count: readonly(count),
    increment
  }
}

const { count, increment } = useCounter()
// count.value++ ❌ — ошибка, так как count доступен только для чтения
increment() // ✅ — единственный способ изменить состояние
```

## Лучшие практики для `computed`

### Предпочитать `computed` вместо `watcher`, присваивающего значение в ref

**ПЛОХО:**
```ts
import { ref, watchEffect } from 'vue'

const items = ref([{ price: 10 }, { price: 20 }])
const total = ref(0)

watchEffect(() => {
  total.value = items.value.reduce((sum, item) => sum + item.price, 0)
})
```

**ХОРОШО:**
```ts
import { ref, computed } from 'vue'

const items = ref([{ price: 10 }, { price: 20 }])
const total = computed(() =>
  items.value.reduce((sum, item) => sum + item.price, 0)
)
```

### Выносить фильтрацию/сортировку из шаблонов

**ПЛОХО:**
```vue
<template>
  <li v-for="item in items.filter(item => item.active)" :key="item.id">
    {{ item.name }}
  </li>

  <li v-for="item in getSortedItems()" :key="item.id">
    {{ item.name }}
  </li>
</template>

<script setup>
  import { ref } from 'vue'

  const items = ref([
    { id: 1, name: 'B', active: true },
    { id: 2, name: 'A', active: false }
  ])

  function getSortedItems() {
    return [...items.value].sort((a, b) => a.name.localeCompare(b.name))
  }
</script>
```

**ХОРОШО:**
```vue
<script setup>
  import { ref, computed } from 'vue'

  const items = ref([
    { id: 1, name: 'B', active: true },
    { id: 2, name: 'A', active: false }
  ])

  const visibleItems = computed(() =>
      items.value
          .filter(item => item.active)
          .sort((a, b) => a.name.localeCompare(b.name))
  )
</script>

<template>
  <li v-for="item in visibleItems" :key="item.id">
    {{ item.name }}
  </li>
</template>
```

### Использовать `computed` для переиспользуемой логики классов/стилей

**ПЛОХО:**
```vue
<template>
  <button :class="{ btn: true, 'btn-primary': type === 'primary' && !disabled, 'btn-disabled': disabled }">
    {{ label }}
  </button>
</template>
```

**ХОРОШО:**
```vue
<script setup>
  import { computed } from 'vue'

  const props = defineProps({
    type: { type: String, default: 'primary' },
    disabled: Boolean,
    label: String
  })

  const buttonClasses = computed(() => ({
    btn: true,
    [`btn-${props.type}`]: !props.disabled,
    'btn-disabled': props.disabled
  }))
</script>

<template>
  <button :class="buttonClasses">
    {{ label }}
  </button>
</template>
```

### Держать геттеры `computed` чистыми (без сайд-эффектов) и помещать сайд-эффекты в наблюдатели

Геттер `computed` должен только вычислять значение. Никаких мутаций, вызовов API, записи в
хранилище или генерации событий.
([Reference](https://vuejs.org/guide/essentials/computed.html#best-practices))

**ПЛОХО:**

сайд-эффекты внутри computed

```ts
const count = ref(0)

const doubled = computed(() => {
  // ❌ сайд-эффект
  if (count.value > 10) console.warn('Слишком большое значение!')
  return count.value * 2
})
```

**ХОРОШО:**

чистый computed + `watch()` для сайд-эффектов

```ts
const count = ref(0)
const doubled = computed(() => count.value * 2)

watch(count, (value) => {
  if (value > 10) console.warn('Too big!')
})
```

## Лучшие практики для наблюдателей

### Использовать `immediate: true` вместо дублирующих начальных вызовов

**ПЛОХО:**
```ts
import { ref, watch, onMounted } from 'vue'

const userId = ref(1)

function loadUser(id) {
  // ...
}

onMounted(() => loadUser(userId.value))
watch(userId, (id) => loadUser(id))
```

**ХОРОШО:**
```ts
import { ref, watch } from 'vue'

const userId = ref(1)

watch(
  userId,
  (id) => loadUser(id),
  { immediate: true }
)
```

### Очищать асинхронные эффекты в наблюдателях

При реакции на быстрые изменения (поисковые поля, фильтры) отменяйте предыдущий запрос.

**ХОРОШО:**

```ts
const query = ref('')
const results = ref<string[]>([])

watch(query, async (q, _prev, onCleanup) => {
  const controller = new AbortController()
  onCleanup(() => controller.abort())

  const res = await fetch(`/api/search?q=${encodeURIComponent(q)}`, {
    signal: controller.signal,
  })

  results.value = await res.json()
})
```
