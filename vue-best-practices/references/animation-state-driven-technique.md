---
title: Управляемые состоянием анимации с CSS-переходами и динамической привязкой стилей
impact: LOW
impactDescription: Сочетание реактивных привязок стилей Vue с CSS-переходами создаёт плавные интерактивные анимации
type: best-practice
tags: [vue3, animation, css, transition, style-binding, state, interactive]
---

# Управляемые состоянием анимации с CSS-переходами и динамической привязкой стилей

**Impact: НИЗКОЕ** - Для создания отзывчивых, интерактивных анимаций, которые реагируют на действия
пользователя или изменение состояния, комбинируйте динамическую привязку стилей Vue с
CSS-переходами. Это создаёт плавные анимации, которые интерполируют значения в реальном времени
в зависимости от состояния.

## Список задач

- Используйте `:style` для динамических свойств, которые часто меняются
- Добавьте CSS-свойство `transition` для плавной анимации между значениями
- Рассмотрите использование `transform` и `opacity` для анимаций с ускорением на GPU
- Для сложной интерполяции значений используйте наблюдатели (watchers) с библиотеками анимаций

## Базовый паттерн

```vue
<template>
  <div
      @mousemove="onMousemove"
      :style="{ backgroundColor: `hsl(${hue}, 80%, 50%)` }"
      class="interactive-area"
  >
    <p>Ведите мышью по этой области...</p>
    <p>Оттенок: {{ hue }}</p>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const hue = ref(0)

function onMousemove(e) {
  // Привязываем положение мыши по X к оттенку (0-360)
  const rect = e.currentTarget.getBoundingClientRect()
  hue.value = Math.round((e.clientX - rect.left) / rect.width * 360)
}
</script>

<style>
.interactive-area {
  transition: background-color 0.3s ease;
  height: 200px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}
</style>
```

## Типичные примеры использования

### Следование за положением мыши

```vue
<template>
  <div
      class="container"
      @mousemove="onMousemove"
  >
    <div
        class="follower"
        :style="{
        transform: `translate(${x}px, ${y}px)`
      }"
    />
  </div>
</template>

<script setup>
import { ref } from 'vue'

const x = ref(0)
const y = ref(0)

function onMousemove(e) {
  const rect = e.currentTarget.getBoundingClientRect()
  x.value = e.clientX - rect.left
  y.value = e.clientY - rect.top
}
</script>

<style>
.container {
  position: relative;
  height: 300px;
}

.follower {
  position: absolute;
  width: 20px;
  height: 20px;
  background: blue;
  border-radius: 50%;
  /* Плавное следование с переходом */
  transition: transform 0.1s ease-out;
  /* Предотвращаем срабатывание mousemove на самом элементе- follower */
  pointer-events: none;
}
</style>
```

### Анимация прогресса

```vue
<template>
  <div class="progress-container">
    <div
        class="progress-bar"
        :style="{ width: `${progress}%` }"
    />
  </div>
  <input
      type="range"
      v-model.number="progress"
      min="0"
      max="100"
  />
</template>

<script setup>
import { ref } from 'vue'

const progress = ref(0)
</script>

<style>
.progress-container {
  height: 20px;
  background: #e0e0e0;
  border-radius: 10px;
  overflow: hidden;
}

.progress-bar {
  height: 100%;
  background: linear-gradient(90deg, #4CAF50, #8BC34A);
  transition: width 0.3s ease;
}
</style>
```

### Анимация на основе скролла

```vue
<template>
  <div
      class="hero"
      :style="{
      opacity: heroOpacity,
      transform: `translateY(${scrollOffset}px)`
    }"
  >
    <h1>Листайте вниз</h1>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'

const scrollY = ref(0)

const heroOpacity = computed(() => {
  return Math.max(0, 1 - scrollY.value / 300)
})

const scrollOffset = computed(() => {
  return scrollY.value * 0.5  // Эффект параллакса
})

function handleScroll() {
  scrollY.value = window.scrollY
}

onMounted(() => {
  window.addEventListener('scroll', handleScroll, { passive: true })
})

onUnmounted(() => {
  window.removeEventListener('scroll', handleScroll)
})
</script>

<style>
.hero {
  height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  /* Примечание: для анимаций на основе скролла переход не нужен — изменения должны быть мгновенными */
}
</style>
```

### Смена цветовой темы с переходом

```vue
<template>
  <div
      class="app"
      :style="themeStyles"
  >
    <button @click="toggleTheme">Сменить тему</button>
    <p>Текущая тема: {{ isDark ? 'Тёмная' : 'Светлая' }}</p>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'

const isDark = ref(false)

const themeStyles = computed(() => ({
  '--bg-color': isDark.value ? '#1a1a1a' : '#ffffff',
  '--text-color': isDark.value ? '#ffffff' : '#1a1a1a',
  backgroundColor: 'var(--bg-color)',
  color: 'var(--text-color)'
}))

function toggleTheme() {
  isDark.value = !isDark.value
}
</script>

<style>
.app {
  min-height: 100vh;
  transition: background-color 0.5s ease, color 0.5s ease;
}
</style>
```

## Продвинутый уровень: числовая интерполяция с наблюдателями (watchers)

Для плавной анимации чисел (счётчики, статистика) используйте наблюдатели с библиотеками анимации:

```vue
<template>
  <div>
    <input v-model.number="targetNumber" type="number" />
    <p class="counter">{{ displayNumber.toFixed(0) }}</p>
  </div>
</template>

<script setup>
import { computed, ref, reactive, watch } from 'vue'
import gsap from 'gsap'

const targetNumber = ref(0)
const tweened = reactive({ value: 0 })

// Вычисляемое свойство для отображения
const displayNumber = computed(() => tweened.value)

watch(targetNumber, (newValue) => {
  gsap.to(tweened, {
    duration: 0.5,
    value: Number(newValue) || 0,
    ease: 'power2.out'
  })
})
</script>
```

## Рекомендации по производительности

```vue
<style>
/* ХОРОШО: свойства с аппаратным ускорением на GPU */
.element {
  transition: transform 0.3s ease, opacity 0.3s ease;
}

/* ИЗБЕГАЙТЕ: свойств, которые вызывают перерасчёт макета (layout) */
.element {
  transition: width 0.3s ease, height 0.3s ease, margin 0.3s ease;
}

/* При частых обновлениях рассмотрите использование will-change */
.frequently-animated {
  will-change: transform;
}
</style>
```
