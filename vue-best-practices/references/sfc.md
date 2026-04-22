---
title: Структура однофайловых компонентов, стилизация и шаблоны шаблонов
impact: MEDIUM
impactDescription: Согласованная структура SFC и выбор стилизации улучшают поддерживаемость, работу инструментов и производительность рендеринга
type: best-practice
tags: [vue3, sfc, scoped-css, styles, build-tools, performance, template, v-html, v-for, computed, v-if, v-show]
---

# Структура однофайловых компонентов, стилизация и шаблоны шаблонов

**Impact: СРЕДНЕЕ** – Использование SFC с согласованной структурой и производительными стилями
облегчает поддержку компонентов и избегает издержек рендеринга.

## Список задач

- Используйте `.vue` SFC вместо отдельных файлов `.js`/`.ts` и `.css` для компонентов
- По умолчанию размещайте шаблон, скрипт и стили в одном SFC
- Используйте PascalCase для имён компонентов в шаблонах и именах файлов
- Отдавайте предпочтение стилям с ограниченной областью действия (scoped)
- Отдавайте предпочтение селекторам классов (не селекторам элементов) в scoped CSS для 
производительности
- олучайте доступ к ссылкам на DOM/компоненты с помощью `useTemplateRef()` в Vue 3.5+
- Используйте ключи в camelCase в привязках `:style` для согласованности и поддержки IDE
- Правильно используйте `v-for` и `v-if`
- Никогда не используйте `v-html` с непроверенным/пользовательским содержимым
- Выбирайте между `v-if` и `v-show` на основе частоты переключения и стоимости начального рендеринга

## Размещайте шаблон, скрипт и стили вместе

**ПЛОХО:**
```
components/
├── UserCard.vue
├── UserCard.js
└── UserCard.css
```

**ХОРОШО:**
```vue
<!-- components/UserCard.vue -->
<script setup>
import { computed } from 'vue'

const props = defineProps({
  user: { type: Object, required: true }
})

const displayName = computed(() =>
    `${props.user.firstName} ${props.user.lastName}`
)
</script>

<template>
  <div class="user-card">
    <h3 class="name">{{ displayName }}</h3>
  </div>
</template>

<style scoped>
.user-card {
  padding: 1rem;
}

.name {
  margin: 0;
}
</style>
```

## Используйте PascalCase для имён компонентов

**ПЛОХО:**
```vue
<script setup>
import userProfile from './user-profile.vue'
</script>

<template>
  <user-profile :user="currentUser" />
</template>
```

**ХОРОШО:**
```vue
<script setup>
import UserProfile from './UserProfile.vue'
</script>

<template>
  <UserProfile :user="currentUser" />
</template>
```

## Лучшие практики для блока `<style>` в SFC

### Отдавайте предпочтение стилям с ограниченной областью действия

- Используйте `<style scoped>` для стилей, относящихся к компоненту.
- Храните глобальный CSS в отдельном файле (например, `src/assets/main.css`) для сбросов, 
типографики, токенов и т.д.
- Используйте `:deep()` экономно (только в крайних случаях).

**ПЛОХО:**

```vue
<style>
/* ❌ влияет на всё */
button { border-radius: 999px; }
</style>
```

**ХОРОШО:**

```vue
<style scoped>
.button { border-radius: 999px; }
</style>
```

**ХОРОШО:**

```css
/* src/assets/main.css */
/* ✅ сбросы, токены, типографика, общие правила */
:root { --radius: 999px; }
```

### Используйте селекторы классов в scoped CSS

**ПЛОХО:**
```vue
<template>
  <article>
    <h1>{{ title }}</h1>
    <p>{{ subtitle }}</p>
  </article>
</template>

<style scoped>
article { max-width: 800px; }
h1 { font-size: 2rem; }
p { line-height: 1.6; }
</style>
```

**ХОРОШО:**
```vue
<template>
  <article class="article">
    <h1 class="article-title">{{ title }}</h1>
    <p class="article-subtitle">{{ subtitle }}</p>
  </article>
</template>

<style scoped>
.article { max-width: 800px; }
.article-title { font-size: 2rem; }
.article-subtitle { line-height: 1.6; }
</style>
```

## Получайте доступ к ссылкам на DOM/компоненты с помощью `useTemplateRef()`

Для Vue 3.5+: используйте `useTemplateRef()` для доступа к ссылкам в шаблоне.

```vue
<script setup lang="ts">
import { onMounted, useTemplateRef } from 'vue'

const inputRef = useTemplateRef<HTMLInputElement>('input')

onMounted(() => {
  inputRef.value?.focus()
})
</script>

<template>
  <input ref="input" />
</template>
```

## Используйте camelCase в привязках `:style`

**ПЛОХО:**
```vue
<template>
  <div :style="{ 'font-size': fontSize + 'px', 'background-color': bg }">
    Содержимое
  </div>
</template>
```

**ХОРОШО:**
```vue
<template>
  <div :style="{ fontSize: fontSize + 'px', backgroundColor: bg }">
    Содержимое
  </div>
</template>
```

## Правильно используйте `v-for` и `v-if`

### Всегда указывайте стабильный `:key`

- Отдавайте предпочтение примитивным ключам (`string` | `number`).
- Избегайте использования объектов в качестве ключей.

**ХОРОШО:**

```vue
<li v-for="item in items" :key="item.id">
  <input v-model="item.text" />
</li>
```

### Избегайте использования `v-if` и `v-for` на одном элементе

Это приводит к неясному намерению и лишней работе.
([Reference](https://vuejs.org/guide/essentials/list.html#v-for-with-v-if))

**Для фильтрации элементов**
**ПЛОХО:**

```vue
<li v-for="user in users" v-if="user.active" :key="user.id">
  {{ user.name }}
</li>
```

**ХОРОШО:**

```vue
<script setup lang="ts">
import { computed } from 'vue'

const activeUsers = computed(() => users.value.filter(u => u.active))
</script>

<template>
  <li v-for="user in activeUsers" :key="user.id">
    {{ user.name }}
  </li>
</template>
```

**Для условного показа/скрытия всего списка**
**ХОРОШО:**

```vue
<ul v-if="shouldShowUsers">
  <li v-for="user in users" :key="user.id">
    {{ user.name }}
  </li>
</ul>
```

## Никогда не рендерите непроверенный HTML с помощью `v-html`

**ПЛОХО:**
```vue
<template>
  <!-- ОПАСНО: непроверенные входные данные могут внедрять скрипты -->
  <article v-html="userProvidedContent"></article>
</template>
```

**ХОРОШО:**
```vue
<script setup>
import { computed } from 'vue'
import DOMPurify from 'dompurify'

const props = defineProps<{
  trustedHtml?: string
  plainText: string
}>()

const safeHtml = computed(() => DOMPurify.sanitize(props.trustedHtml ?? ''))
</script>

<template>
  <!-- Предпочтительно: экранированная интерполяция -->
  <p>{{ props.plainText }}</p>

  <!-- Только для доверенного/очищенного HTML -->
  <article v-html="safeHtml"></article>
</template>
```

## Выбирайте `v-if` или `v-show` в зависимости от поведения переключения

**ПЛОХО:**
```vue
<template>
  <!-- Частые переключения с v-if вызывают повторный монтинг/демонтинг -->
  <ComplexPanel v-if="isPanelOpen" />

  <!-- Редко показываемый контент с v-show платит стоимость начального рендера -->
  <AdminPanel v-show="isAdmin" />
</template>
```

**ХОРОШО:**
```vue
<template>
  <!-- Частые переключения: оставляем в DOM, переключаем display -->
  <ComplexPanel v-show="isPanelOpen" />

  <!-- Редкое условие: ленивый рендер только когда true -->
  <AdminPanel v-if="isAdmin" />
</template>
```
