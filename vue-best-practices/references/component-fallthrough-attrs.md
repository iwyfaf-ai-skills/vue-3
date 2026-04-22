---
title: Рекомендации по работе с сквозными атрибутами компонентов
impact: MEDIUM
impactDescription: Неправильный доступ к $attrs и неверные предположения о реактивности могут приводить к неопределённым значениям и наблюдателям, которые никогда не срабатывают
type: best-practice
tags: [vue3, attrs, fallthrough-attributes, composition-api, reactivity]
---

# Рекомендации по работе со сквозными атрибутами компонентов

**Impact: СРЕДНЯЯ** - Сквозные атрибуты не вызывают сложностей, если следовать соглашениям Vue:
имена с дефисом требуют скобочной нотации, ключи слушателей записываются в camelCase как `onX`, а
`useAttrs()` возвращает актуальное, но не реактивное значение.

## Task List

- Обращайтесь к именам атрибутов с дефисом через скобочную нотацию (например, `attrs['data-testid']`)
- Обращайтесь к слушателям событий через ключи в camelCase с префиксом `onX` (например, `attrs.onClick`)
- Не используйте `watch()` для значений, возвращённых `useAttrs()`; такие наблюдатели не
срабатывают при изменении атрибутов
- Используйте `onUpdated()` для побочных эффектов, зависящих от атрибутов
- Выносите часто используемые атрибуты в явные props, если требуется реактивность

## Правильный доступ к ключам атрибутов и слушателей

Имена атрибутов с дефисом сохраняют исходный регистр в JavaScript, поэтому точечная нотация не
работает для ключей, содержащих `-`.

**ПЛОХО:**
```vue
<script setup>
import { useAttrs } from 'vue'

const attrs = useAttrs()

console.log(attrs.data-testid)  // Синтаксическая ошибка
console.log(attrs.dataTestid)   // undefined для data-testid
console.log(attrs['on-click'])  // undefined
console.log(attrs['@click'])    // undefined
</script>
```

**ХОРОШО:**
```vue
<script setup>
import { useAttrs } from 'vue'

const attrs = useAttrs()

console.log(attrs['data-testid'])
console.log(attrs['aria-label'])
console.log(attrs['foo-bar'])

console.log(attrs.onClick)
console.log(attrs.onCustomEvent)
console.log(attrs.onMouseEnter)
</script>
```

### Таблица соответствия имён

| Использование в родителе  | Доступ в `attrs`               |
|---------------------------|--------------------------------|
| `class="foo"`             | `attrs.class`                  |
| `data-id="123"`           | `attrs['data-id']`             |
| `aria-label="..."`        | `attrs['aria-label']`          |
| `foo-bar="baz"`           | `attrs['foo-bar']`             |
| `@click="fn"`             | `attrs.onClick`                |
| `@custom-event="fn"`      | `attrs.onCustomEvent`          |
| `@update:modelValue="fn"` | `attrs['onUpdate:modelValue']` |

## `useAttrs()` не реактивен

`useAttrs()` всегда возвращает актуальные значения, но намеренно не поддерживает реактивность для
отслеживания через наблюдатели.

**ПЛОХО:**
```vue
<script setup>
import { watch, watchEffect, useAttrs } from 'vue'

const attrs = useAttrs()

watch(
    () => attrs.someAttr,
    (newValue) => {
      console.log('Изменилось:', newValue) // Никогда не срабатывает при изменении атрибутов
    }
)

watchEffect(() => {
  console.log(attrs.class) // Срабатывает при setup, но не при обновлении атрибутов
})
</script>
```

**ХОРОШО:**
```vue
<script setup>
import { onUpdated, useAttrs } from 'vue'

const attrs = useAttrs()

onUpdated(() => {
  console.log('Актуальные атрибуты:', attrs)
})
</script>
```

**ХОРОШО:**
```vue
<script setup>
import { watch } from 'vue'

const props = defineProps({
  someAttr: String
})

watch(
    () => props.someAttr,
    (newValue) => {
      console.log('Изменилось:', newValue)
    }
)
</script>
```

## Типовые сценарии

### Безопасная проверка наличия необязательных атрибутов

```vue
<script setup>
import { computed, useAttrs } from 'vue'

const attrs = useAttrs()

const hasTestId = computed(() => 'data-testid' in attrs)
const ariaLabel = computed(() => attrs['aria-label'] ?? 'Стандартная метка')
</script>
```

### Проброс слушателей после внутренней логики

```vue
<script setup>
import { useAttrs } from 'vue'

defineOptions({ inheritAttrs: false })

const attrs = useAttrs()

function handleClick(event) {
  console.log('Сначала сработала внутренняя обработка')
  attrs.onClick?.(event)
}
</script>

<template>
  <button @click="handleClick">
    <slot />
  </button>
</template>
```

## Замечания по TypeScript

`useAttrs()` типизирован как `Record<string, unknown>`, поэтому при необходимости приводите
отдельные ключи к нужному типу.

```vue
<script setup lang="ts">
import { useAttrs } from 'vue'

const attrs = useAttrs()

const testId = attrs['data-testid'] as string | undefined
const onClick = attrs.onClick as ((event: MouseEvent) => void) | undefined
</script>
```
