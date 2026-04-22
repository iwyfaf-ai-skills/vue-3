---
title: Лучшие практики использования компонента Teleport
impact: MEDIUM
impactDescription: Teleport рендерит содержимое вне DOM-позиции компонента, что важно для оверлеев, но влияет на стилизацию и макет
type: best-practice
tags: [vue3, teleport, modal, overlay, positioning, responsive]
---

# Teleport Component Best Practices

**Impact: СРЕДНЯЯ** - `<Teleport>` рендерит часть шаблона компонента в другом месте DOM, сохраняя
при этом иерархию компонентов Vue. Используйте его для оверлеев (модальные окна, тосты, тултипы)
или любого интерфейса, который должен избегать ограничений контекста наложения (stacking context),
переполнения (overflow) или фиксированного позиционирования.

## Список задач

- Телепортируйте оверлеи в `body` или специальный контейнер вне корня приложения
- Используйте общую цель для похожих элементов интерфейса (`#modals`, `#notifications`)
и управляйте наслоением с помощью порядка или `z-index`
- Используйте `:disabled` для адаптивных макетов, которые должны рендериться внутри потока на
маленьких экранах
- Помните, что пропсы, события (emits) и `provide/inject` продолжают работать через телепорт
- Избегайте зависимости от родительских контекстов наложения (stacking contexts) или трансформаций
для телепортированного интерфейса

## Телепортируйте оверлеи из трансформированных контейнеров

Если у предка есть `transform`, `filter` или `perspective`, оверлеи с фиксированным
позиционированием могут вести себя так, как будто они позиционируются локально. Teleport позволяет
избежать этого контекста.

**ПЛОХО:**
```vue
<template>
  <div class="animated-container">
    <button @click="open = true">Открыть</button>

    <!-- Неработающее окно: фиксированное позиционирование ограничено трансформированным родителем -->
    <div v-if="open" class="modal">Модальное окно</div>
  </div>
</template>

<style>
.animated-container {
  transform: translateZ(0);
}

.modal {
  position: fixed;
  inset: 0;
  z-index: 9999;
}
</style>
```

**ХОРОШО:**
```vue
<template>
  <div class="animated-container">
    <button @click="open = true">Открыть</button>

    <Teleport to="body">
      <div v-if="open" class="modal">Модальное окно</div>
    </Teleport>
  </div>
</template>
```

## Адаптивные макеты с помощью `disabled`

Используйте `:disabled`, чтобы на мобильных устройствах рендерить внутри потока, а на больших
экранах — через телепорт:

```vue
<script setup>
import { useMediaQuery } from '@vueuse/core'

const isMobile = useMediaQuery('(max-width: 768px)')
</script>

<template>
  <Teleport to="body" :disabled="isMobile">
    <nav class="sidebar">Навигация</nav>
  </Teleport>
</template>
```

## Логическая иерархия сохраняется

Teleport меняет положение в DOM, но не дерево компонентов Vue. Пропсы, события (emits), слоты и
`provide/inject` продолжают работать:

```vue
<template>
  <Teleport to="body">
    <ChildPanel :message="message" @close="open = false" />
  </Teleport>
</template>
```

## Несколько телепортов в одну цель

Телепорты в одну цель добавляются в порядке объявления:

```vue
<template>
  <Teleport to="#notifications">
    <div>Первый</div>
  </Teleport>

  <Teleport to="#notifications">
    <div>Второй</div>
  </Teleport>
</template>
```

Используйте общий контейнер, чтобы сохранить предсказуемость наложения, и применяйте `z-index`
только тогда, когда вам нужно явное управление слоями.
