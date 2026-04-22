---
title: Использование v-once и v-memo для пропуска ненужных обновлений
impact: MEDIUM
impactDescription: v-once пропускает все будущие обновления для статического контента; v-memo условно мемоизирует поддеревья
type: efficiency
tags: [vue3, performance, v-once, v-memo, optimization, directives]
---

# Использование v-once и v-memo для пропуска ненужных обновлений

**Impact: СРЕДНЕЕ** - Vue пересчитывает шаблоны при каждом изменении реактивных данных.
Для контента, который никогда не меняется или меняется редко, директивы `v-once` и `v-memo`
сообщают Vue о необходимости пропускать обновления, уменьшая нагрузку на рендеринг.

Используйте `v-once` для действительно статического контента и `v-memo` для условно-статического
контента в списках.

## Список задач

- Примените `v-once` к элементам, которые используют данные во время выполнения, но никогда не
требуют обновления
- Примените `v-memo` к элементам списка, которые должны обновляться только при изменении
определённых условий
- Убедитесь, что мемоизированный контент не должен реагировать на другие изменения состояния
- Профилируйте с помощью Vue DevTools, чтобы подтвердить пропуск обновлений

## v-once: отрендерить один раз, никогда не обновлять

**ПЛОХО:**
```vue
<template>
  <!-- ПЛОХО: Пересчитывается при каждом ререндере родителя -->
  <div class="terms-content">
    <h1>Пользовательское соглашение</h1>
    <p>Версия: {{ termsVersion }}</p>
    <div v-html="termsContent"></div>
  </div>

  <!-- Этот контент НИКОГДА не меняется, но Vue проверяет его каждый рендер -->
  <footer>
    <p>© {{ copyrightYear }} {{ companyName }}</p>
  </footer>
</template>
```

**ХОРОШО:**
```vue
<template>
  <!-- ХОРОШО: Рендерится один раз, пропускается при всех будущих обновлениях -->
  <div class="terms-content" v-once>
    <h1>Пользовательское соглашение</h1>
    <p>Версия: {{ termsVersion }}</p>
    <div v-html="termsContent"></div>
  </div>

  <!-- v-once сообщает Vue, что этот контент никогда не нужно обновлять -->
  <footer v-once>
    <p>© {{ copyrightYear }} {{ companyName }}</p>
  </footer>
</template>

<script setup>
  // Эти значения устанавливаются один раз при создании компонента
  const termsVersion = '2.1'
  const termsContent = fetchedTermsHTML
  const copyrightYear = 2024
  const companyName = 'ООО "Рога и копыта"'
</script>
```

## v-memo: условная мемоизация для списков

**ПЛОХО:**
```vue
<template>
  <!-- ПЛОХО: Все элементы перерендериваются при изменении selectedId -->
  <div v-for="item in list" :key="item.id">
    <div :class="{ selected: item.id === selectedId }">
      <ExpensiveComponent :data="item" />
    </div>
  </div>
</template>
```

**ХОРОШО:**
```vue
<template>
  <!-- ХОРОШО: Элементы перерендериваются только когда меняется их состояние выделения -->
  <div
      v-for="item in list"
      :key="item.id"
      v-memo="[item.id === selectedId]"
  >
    <div :class="{ selected: item.id === selectedId }">
      <ExpensiveComponent :data="item" />
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const list = ref([/* много элементов */])
const selectedId = ref(null)

// Когда selectedId меняется:
// - Перерендеривается только предыдущий выделенный элемент (selected: true -> false)
// - Перерендеривается только новый выделенный элемент (selected: false -> true)
// - Все остальные элементы ПРОПУСКАЮТСЯ (значения v-memo не изменились)
</script>
```

## v-memo с несколькими зависимостями

```vue
<template>
  <!-- Перерендер только при изменении состояния выделения ИЛИ редактирования элемента -->
  <div
      v-for="item in items"
      :key="item.id"
      v-memo="[item.id === selectedId, item.id === editingId]"
  >
    <ItemCard
        :item="item"
        :selected="item.id === selectedId"
        :editing="item.id === editingId"
    />
  </div>
</template>

<script setup>
const selectedId = ref(null)
const editingId = ref(null)
const items = ref([/* ... */])
</script>
```

## v-memo с пустым массивом = v-once

```vue
<template>
  <!-- v-memo="[]" эквивалентно v-once -->
  <div v-for="item in staticList" :key="item.id" v-memo="[]">
    {{ item.name }}
  </div>
</template>
```

## Когда НЕ следует использовать эти директивы

```vue
<template>
  <!-- НЕ НАДО: Контент, который ДОЛЖЕН обновляться -->
  <div v-once>
    <span>Счётчик: {{ count }}</span>  <!-- count не будет обновляться! -->
  </div>

  <!-- НЕ НАДО: Когда дочерние компоненты имеют своё реактивное состояние -->
  <div v-memo="[selected]">
    <InputField v-model="item.name" />  <!-- v-model не будет работать корректно -->
  </div>

  <!-- НЕ НАДО: Когда выгода от мемоизации минимальна -->
  <span v-once>{{ simpleText }}</span>  <!-- Оверхед не оправдан -->
</template>
```

## Сравнение производительности

| Сценарий                                           | Без директивы             | С v-once/v-memo              |
|----------------------------------------------------|---------------------------|------------------------------|
| Статический заголовок, родитель рендерится 100 раз | Пересчитывается 100 раз   | Вычисляется 1 раз            |
| 1000 элементов, изменение выделения                | 1000 элементов перерендер | 2 элемента перерендер        |
| Сложный дочерний компонент                         | Полный перерендер         | Пропускается при мемоизации  |

## Отладка мемоизированных компонентов

```vue
<script setup>
import { onUpdated } from 'vue'

// Этот хук не сработает, если v-memo предотвратил обновление
onUpdated(() => {
  console.log('Компонент обновлён')
})
</script>
```
