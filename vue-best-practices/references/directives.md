---
title: Рекомендации по использованию директив
impact: MEDIUM
impactDescription: Пользовательские директивы мощны, но их легко использовать неправильно; следование шаблонам предотвращает утечки, недопустимые варианты использования и неясные абстракции
type: best-practice
tags: [vue3, directives, custom-directives, composition, typescript]
---

# Directive Best Practices

**Impact: СРЕДНЕЕ** - Директивы предназначены для низкоуровневого доступа к DOM. Используйте их
экономно, обеспечивайте безопасность побочных эффектов и предпочитайте компоненты или композаблы,
когда требуется управляемое состояние или повторно используемое UI-поведение.

## Список задач

- Используйте директивы только тогда, когда вам нужен прямой доступ к DOM
- Не изменяйте аргументы директив или объекты привязки
- Очищайте таймеры, слушатели и наблюдатели в `unmounted`
- Регистрируйте директивы в `<script setup>` с префиксом `v-`
- В проектах TypeScript типизируйте значения директив и расширяйте типы шаблонных директив
- Для сложного поведения предпочитайте компоненты или композаблы

## Рассматривайте аргументы директив как доступные только для чтения

Привязки директив не являются реактивным хранилищем. Не записывайте в них.

```ts
const vFocus = {
  mounted(el, binding) {
    // binding.value доступен только для чтения
    el.focus()
  }
}
```

## Избегайте директив на компонентах

Директивы применяются к элементам DOM. При использовании на компонентах они прикрепляются к
корневому элементу и могут сломаться, если корневой элемент изменится.

**ПЛОХО:**
```vue
<MyInput v-focus />
```

**ХОРОШО:**
```vue
<!-- MyInput.vue -->
<script setup>
const vFocus = (el) => el.focus()
</script>

<template>
  <input v-focus />
</template>
```

## Очищайте побочные эффекты в `unmounted`

Любые таймеры, слушатели или наблюдатели должны быть удалены во избежание утечек.

```ts
const vResize = {
  mounted(el) {
    const observer = new ResizeObserver(() => {})
    observer.observe(el)
    el._observer = observer
  },
  unmounted(el) {
    el._observer?.disconnect()
  }
}
```

## Используйте сокращенную функцию для директив с одним хуком

Если вам нужны только `mounted`/`updated`, используйте форму функции.

```ts
const vAutofocus = (el) => el.focus()
```

## Используйте префикс `v-` и регистрацию в Script Setup

```vue
<script setup>
const vFocus = (el) => el.focus()
</script>

<template>
  <input v-focus />
</template>
```

## Типизируйте пользовательские директивы в проектах TypeScript

Используйте `Directive<Element, ValueType>`, чтобы `binding.value` был типизирован, и расширяйте
типы шаблонов Vue, чтобы директивы распознавались в SFC-шаблонах.

**ПЛОХО:**
```ts
// Нетипизированное значение директивы и отсутствие расширения типов шаблона
export const vHighlight = {
  mounted(el, binding) {
    el.style.backgroundColor = binding.value
  }
}
```

**ХОРОШО:**
```ts
import type { Directive } from 'vue'

type HighlightValue = string

export const vHighlight = {
  mounted(el, binding) {
    el.style.backgroundColor = binding.value
  }
} satisfies Directive<HTMLElement, HighlightValue>

declare module 'vue' {
  interface ComponentCustomProperties {
    vHighlight: typeof vHighlight
  }
}
```

## Обрабатывайте SSR с помощью `getSSRProps`

Хуки директив, такие как `mounted` и `updated`, не выполняются во время SSR. Если директива
устанавливает атрибуты/классы, влияющие на отображаемый HTML, предоставьте SSR-эквивалент через
`getSSRProps`, чтобы избежать несоответствий при гидратации.

**ПЛОХО:**
```ts
const vTooltip = {
  mounted(el, binding) {
    el.setAttribute('data-tooltip', binding.value)
    el.classList.add('has-tooltip')
  }
}
```

**ХОРОШО:**
```ts
const vTooltip = {
  mounted(el, binding) {
    el.setAttribute('data-tooltip', binding.value)
    el.classList.add('has-tooltip')
  },
  getSSRProps(binding) {
    return {
      'data-tooltip': binding.value,
      class: 'has-tooltip'
    }
  }
}
```

## Отдавайте предпочтение декларативным шаблонам, когда это возможно

Если подходит стандартный атрибут или привязка, используйте его вместо директивы.

## Выбирайте между директивами и компонентами

Используйте директиву для поведения на уровне DOM. Используйте компонент, когда поведение влияет на
структуру, состояние или рендеринг.