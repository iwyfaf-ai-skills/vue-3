---
title: Использование классовых анимаций для эффектов без появления/исчезновения
impact: LOW
impactDescription: Классовые анимации проще и производительнее для элементов, которые остаются в DOM
type: best-practice
tags: [vue3, animation, css, class-binding, state]
---

# Использование классовых анимаций для эффектов без появления/исчезновения

**Impact: НИЗКОЕ** - для анимации элементов, которые не появляются и не исчезают из DOM,
используйте CSS-анимацию на основе классов, запускаемую реактивным состоянием Vue. Это проще, чем
`<Transition>`, и лучше подходит для анимаций обратной связи, таких как встряхивание, пульсация или
эффект выделения.

## Список задач

- Используйте классовую анимацию для элементов, остающихся в DOM
- Используйте `<Transition>` только для анимации появления/исчезновения
- Комбинируйте CSS-анимацию с привязкой классов Vue (`:class`)
- Рассмотрите использование `setTimeout` для автоматического удаления классов анимации

**Когда использовать классовую анимацию:**

- Обратная связь с пользователем (встряхивание при ошибке, пульсация при успехе)
- Эффекты привлечения внимания (выделение изменений)
- Состояния при наведении/фокусе, требующие большего, чем CSS-переходы
- Любая анимация, где элемент остаётся смонтированным

**Когда использовать компонент Transition:**

- Элементы, появляющиеся/исчезающие из DOM (`v-if`/`v-show`)
- Анимация переходов между маршрутами
- Добавление/удаление элементов списка

## Базовый паттерн

```vue
<template>
  <div :class="{ shake: showError }">
    <button @click="submitForm">Отправить</button>
    <span v-if="showError">Эта функция отключена!</span>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const showError = ref(false)

function submitForm() {
  if (!isValid()) {
    // Запускаем анимацию встряхивания
    showError.value = true

    // Автоматически удаляем класс после завершения анимации
    setTimeout(() => {
      showError.value = false
    }, 820)  // Соответствует длительности анимации
  }
}
</script>

<style>
.shake {
  animation: shake 0.82s cubic-bezier(0.36, 0.07, 0.19, 0.97) both;
  transform: translate3d(0, 0, 0);  /* Включаем GPU-ускорение */
}

@keyframes shake {
  10%, 90% { transform: translate3d(-1px, 0, 0); }
  20%, 80% { transform: translate3d(2px, 0, 0); }
  30%, 50%, 70% { transform: translate3d(-4px, 0, 0); }
  40%, 60% { transform: translate3d(4px, 0, 0); }
}
</style>
```

## Распространённые паттерны анимации

### Пульсация при успехе

```vue
<template>
  <button
      @click="save"
      :class="{ pulse: saved }"
  >
    {{ saved ? 'Сохранено!' : 'Сохранить' }}
  </button>
</template>

<script setup>
  import { ref } from 'vue'

  const saved = ref(false)

  async function save() {
    await saveData()
    saved.value = true
    setTimeout(() => saved.value = false, 1000)
  }
</script>

<style>
.pulse {
  animation: pulse 0.5s ease-in-out;
}

@keyframes pulse {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.05); }
}
</style>
```

### Выделение при изменении

```vue
<template>
  <div
      :class="{ highlight: justUpdated }"
  >
    Значение: {{ value }}
  </div>
</template>

<script setup>
import { ref, watch } from 'vue'

const value = ref(0)
const justUpdated = ref(false)

watch(value, () => {
  justUpdated.value = true
  setTimeout(() => justUpdated.value = false, 1000)
})
</script>

<style>
.highlight {
  animation: highlight 1s ease-out;
}

@keyframes highlight {
  0% { background-color: yellow; }
  100% { background-color: transparent; }
}
</style>
```

### Подпрыгивание для привлечения внимания

```vue
<template>
  <div
      :class="{ bounce: needsAttention }"
      @animationend="needsAttention = false"
  >
    <BellIcon />
  </div>
</template>

<script setup>
import { ref } from 'vue'

const needsAttention = ref(false)

function notifyUser() {
  needsAttention.value = true
  // setTimeout не нужен — используем событие animationend
}
</script>

<style>
.bounce {
  animation: bounce 0.5s ease;
}

@keyframes bounce {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-10px); }
}
</style>
```

## Использование события animationend

Вместо `setTimeout` используйте событие `animationend` для более чистого кода:

```vue
<template>
  <div
      :class="{ animate: isAnimating }"
      @animationend="isAnimating = false"
  >
    Содержимое
  </div>
</template>

<script setup>
import { ref } from 'vue'

const isAnimating = ref(false)

function triggerAnimation() {
  isAnimating.value = true
  // Класс будет автоматически удалён по окончании анимации
}
</script>
```

## Композабл для переиспользуемых анимаций

```javascript
// composables/useAnimation.js
import { ref } from 'vue'

export function useAnimation(duration = 500) {
  const isAnimating = ref(false)

  function trigger() {
    isAnimating.value = true
    setTimeout(() => {
      isAnimating.value = false
    }, duration)
  }

  return {
    isAnimating,
    trigger
  }
}
```

```vue
<script setup>
import { useAnimation } from '@/composables/useAnimation'

const shake = useAnimation(820)
const pulse = useAnimation(500)
</script>

<template>
  <button
      :class="{ shake: shake.isAnimating.value }"
      @click="shake.trigger()"
  >
    Встряхни меня
  </button>

  <button
      :class="{ pulse: pulse.isAnimating.value }"
      @click="pulse.trigger()"
  >
    Пульсируй
  </button>
</template>
```
