---
title: Практики использования слотов компонентов
impact: MEDIUM
impactDescription: Плохой дизайн API слотов приводит к пустым DOM-обёрткам, слабой типобезопасности, ненадёжным значениям по умолчанию и излишней нагрузке на компоненты
type: best-practice
tags: [vue3, slots, components, typescript, composables]
---

# Component Slots Best Practices

**Impact: СРЕДНЯЯ** - Слоты — это ключевая часть API компонентов в Vue. Проектируйте их осознанно,
чтобы шаблоны оставались предсказуемыми, типизированными и производительными.

## Список задач

- Используйте сокращённый синтаксис для именованных слотов (`#` вместо `v-slot:`)
- Отрисовывайте элементы-обёртки для опциональных слотов только при наличии контента (проверки
через `$slots`)
- Типизируйте контракты слотов с помощью `defineSlots` в TypeScript-компонентах
- Предоставляйте запасной контент для опциональных слотов
- Отдавайте предпочтение композаблам перед renderless-компонентами для повторного использования
чистой логики

## Сокращённый синтаксис для именованных слотов

**ПЛОХО:**
```vue
<MyComponent>
  <template v-slot:header> ... </template>
</MyComponent>
```

**ХОРОШО:**
```vue
<MyComponent>
  <template #header> ... </template>
</MyComponent>
```

## Условный рендеринг обёрток опциональных слотов

Используйте проверки `$slots`, когда элементы-обёртки добавляют отступы, границы или ограничения по
вёрстке.

**ПЛОХО:**
```vue
<!-- Card.vue -->
<template>
  <article class="card">
    <header class="card-header">
      <slot name="header" />
    </header>

    <section class="card-body">
      <slot />
    </section>

    <footer class="card-footer">
      <slot name="footer" />
    </footer>
  </article>
</template>
```

**ХОРОШО:**
```vue
<!-- Card.vue -->
<template>
  <article class="card">
    <header v-if="$slots.header" class="card-header">
      <slot name="header" />
    </header>

    <section v-if="$slots.default" class="card-body">
      <slot />
    </section>

    <footer v-if="$slots.footer" class="card-footer">
      <slot name="footer" />
    </footer>
  </article>
</template>
```

## Типизация пропсов слотов с помощью defineSlots

В `<script setup lang="ts">` используйте `defineSlots`, чтобы потребители слотов получали
автодополнение и статические проверки.

**ПЛОХО:**
```vue
<!-- ProductList.vue -->
<script setup lang="ts">
interface Product {
  id: number
  name: string
}

defineProps<{ products: Product[] }>()
</script>

<template>
  <ul>
    <li v-for="(product, index) in products" :key="product.id">
      <slot :product="product" :index="index" />
    </li>
  </ul>
</template>
```

**ХОРОШО:**
```vue
<!-- ProductList.vue -->
<script setup lang="ts">
interface Product {
  id: number
  name: string
}

defineProps<{ products: Product[] }>()

defineSlots<{
  default(props: { product: Product; index: number }): any
  empty(): any
}>()
</script>

<template>
  <ul v-if="products.length">
    <li v-for="(product, index) in products" :key="product.id">
      <slot :product="product" :index="index" />
    </li>
  </ul>
  <slot v-else name="empty" />
</template>
```

## Предоставление запасного контента для слотов

Запасной контент делает компоненты устойчивыми к ситуации, когда родитель опускает опциональный
слот.

**ПЛОХО:**
```vue
<!-- SubmitButton.vue -->
<template>
  <button type="submit" class="btn-primary">
    <slot />
  </button>
</template>
```

**ХОРОШО:**
```vue
<!-- SubmitButton.vue -->
<template>
  <button type="submit" class="btn-primary">
    <slot>Отправить</slot>
  </button>
</template>
```

## Используйте композаблы вместо renderless-компонентов для чистой логики

Renderless-компоненты всё ещё полезны для композиции через слоты, но для переиспользования одной
лишь логики композаблы обычно оказываются чище.

**ПЛОХО:**
```vue
<!-- MouseTracker.vue -->
<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

const x = ref(0)
const y = ref(0)

function onMove(event: MouseEvent) {
  x.value = event.pageX
  y.value = event.pageY
}

onMounted(() => window.addEventListener('mousemove', onMove))
onUnmounted(() => window.removeEventListener('mousemove', onMove))
</script>

<template>
  <slot :x="x" :y="y" />
</template>
```

**ХОРОШО:**
```ts
// composables/useMouse.ts
import { ref, onMounted, onUnmounted } from 'vue'

export function useMouse() {
  const x = ref(0)
  const y = ref(0)

  function onMove(event: MouseEvent) {
    x.value = event.pageX
    y.value = event.pageY
  }

  onMounted(() => window.addEventListener('mousemove', onMove))
  onUnmounted(() => window.removeEventListener('mousemove', onMove))

  return { x, y }
}
```

```vue
<!-- MousePosition.vue -->
<script setup lang="ts">
import { useMouse } from '@/composables/useMouse'

const { x, y } = useMouse()
</script>

<template>
  <p>{{ x }}, {{ y }}</p>
</template>
```
