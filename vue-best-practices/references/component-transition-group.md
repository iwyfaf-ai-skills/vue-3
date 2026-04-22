---
title: Лучшие практики использования компонента TransitionGroup
impact: MEDIUM
impactDescription: TransitionGroup анимирует элементы списка; отсутствие ключей или неправильное использование приводит к некорректной анимации списков
type: best-practice
tags: [vue3, transition-group, animation, lists, keys]
---

# Лучшие практики использования компонента TransitionGroup

**Impact: СРЕДНИЙ** - `<TransitionGroup>` анимирует списки элементов при их добавлении, удалении
и перемещении. Используйте его для списков `v-for` или динамических коллекций, где отдельные
элементы меняются со временем.

## Список задач

- Используйте `<TransitionGroup>` только для списков и повторяющихся элементов
- Указывайте уникальные стабильные ключи для каждого прямого потомка
- Используйте `tag`, если нужна семантическая обертка или обертка для управления
- Избегайте использования пропа `mode` (не поддерживается)
- Используйте JavaScript-хуки для создания эффекта каскадной анимации

## Используйте TransitionGroup для списков

`<TransitionGroup>` предназначен для элементов списка. Используйте `tag`, чтобы при необходимости
управлять элементом-оберткой.

**ПЛОХО:**
```vue
<template>
  <TransitionGroup name="fade">
    <ComponentA />
    <ComponentB />
  </TransitionGroup>
</template>
```

**ХОРОШО:**
```vue
<template>
  <TransitionGroup name="list" tag="ul">
    <li v-for="item in items" :key="item.id">
      {{ item.name }}
    </li>
  </TransitionGroup>
</template>
```

## Всегда указывайте стабильные ключи

Ключи обязательны. Без стабильных ключей Vue не может отслеживать позиции элементов, и анимация
ломается.

**ПЛОХО:**
```vue
<template>
  <TransitionGroup name="list" tag="ul">
    <li v-for="(item, index) in items" :key="index">
      {{ item.name }}
    </li>
  </TransitionGroup>
</template>
```

**ХОРОШО:**
```vue
<template>
  <TransitionGroup name="list" tag="ul">
    <li v-for="item in items" :key="item.id">
      {{ item.name }}
    </li>
  </TransitionGroup>
</template>
```

## Не используйте `mode` с TransitionGroup

`mode` предназначен только для `<Transition>`, так как он управляет сменой одного элемента.
Используйте `<Transition>`, если вам нужна последовательность "уход-появление".

**ПЛОХО:**
```vue
<template>
  <TransitionGroup name="list" tag="div" mode="out-in">
    <div v-for="item in items" :key="item.id">{{ item.name }}</div>
  </TransitionGroup>
</template>
```

**ХОРОШО:**
```vue
<template>
  <Transition name="fade" mode="out-in">
    <component :is="currentView" :key="currentView" />
  </Transition>
</template>
```

## Каскадная анимация списков с использованием data-атрибутов

Для создания каскадной анимации элементов списка передавайте индекс в JavaScript-хуки и
вычисляйте задержку для каждого элемента.

```vue
<template>
  <TransitionGroup
    tag="ul"
    :css="false"
    @before-enter="onBeforeEnter"
    @enter="onEnter"
  >
    <li v-for="(item, index) in items" :key="item.id" :data-index="index">
      {{ item.name }}
    </li>
  </TransitionGroup>
</template>

<script setup>
function onBeforeEnter(el) {
  el.style.opacity = 0
  el.style.transform = 'translateY(12px)'
}

function onEnter(el, done) {
  const delay = Number(el.dataset.index) * 80
  setTimeout(() => {
    el.style.transition = 'all 0.25s ease'
    el.style.opacity = 1
    el.style.transform = 'translateY(0)'
    setTimeout(done, 250)
  }, delay)
}
</script>
```
