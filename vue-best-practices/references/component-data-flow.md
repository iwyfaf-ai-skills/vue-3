---
title: Лучшие практики передачи данных между компонентами
impact: HIGH
impactDescription: Четкий поток данных между компонентами предотвращает ошибки состояния, устаревший интерфейс и хрупкую связанность
type: best-practice
tags: [vue3, props, emits, v-model, provide-inject, data-flow, typescript]
---

# Лучшие практики передачи данных между компонентами

**Impact: ВЫСОКОЕ** — Компоненты Vue остаются надежными, когда поток данных явный: свойства идут
вниз, события — вверх, `v-model` обрабатывает двусторонние привязки, а `provide/inject` обеспечивает
зависимости через дерево компонентов. Размывание этих границ приводит к устаревшему состоянию,
скрытой связанности и трудноотлаживаемому интерфейсу.

Основной принцип потока данных во Vue.js — Свойства вниз / События вверх. Это наиболее
поддерживаемый подход по умолчанию, и однонаправленный поток хорошо масштабируется.

## Task List

- Относитесь к свойствам как к read-only входным данным
- Используйте props/emit для взаимодействия компонентов; оставьте `refs` для императивных действий
- Когда для императивных API требуются `refs`, типизируйте их с помощью шаблонных `refs`
- Генерируйте события вместо прямого изменения состояния родителя
- Используйте `defineModel` для `v-model` в современном Vue (3.4+)
- Осознанно обрабатывайте модификаторы `v-model` в дочерних компонентах
- Используйте символы для ключей `provide/inject`, чтобы избежать пробрасывания свойств
(более ~3 уровней)
- Храните мутации в провайдере или предоставляйте явные действия
- В проектах с TypeScript предпочитайте типизированные `defineProps`, `defineEmits` и `InjectionKey`

## Props: однонаправленный поток данных вниз

Свойства — это входные данные. Не изменяйте их в дочернем компоненте.

**ПЛОХО:**
```vue
<script setup>
const props = defineProps({ count: Number })

function increment() {
  props.count++
}
</script>
```

**ХОРОШО:**

Если состояние нужно изменить, сгенерируйте событие, используйте `v-model` или создайте
локальную копию.

## Предпочитайте props/emit ссылкам на компоненты

**ПЛОХО:**
```vue
<script setup>
import { ref } from 'vue'
import UserForm from './UserForm.vue'

const formRef = ref(null)

function submitForm() {
  if (formRef.value.isValid) {
    formRef.value.submit()
  }
}
</script>

<template>
  <UserForm ref="formRef" />
  <button @click="submitForm">Submit</button>
</template>
```

**ХОРОШО:**
```vue
<script setup>
import UserForm from './UserForm.vue'

function handleSubmit(formData) {
  api.submit(formData)
}
</script>

<template>
  <UserForm @submit="handleSubmit" />
</template>
```

## Типизируйте ссылки на компоненты, когда требуется императивный доступ

По умолчанию предпочитайте props/emits. Когда родителю необходимо вызвать открытый метод дочернего
компонента, явно типизируйте ссылку и предоставляйте из дочернего компонента только intended API с
помощью `defineExpose`.

**ПЛОХО:**
```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'
import DialogPanel from './DialogPanel.vue'

const panelRef = ref(null)

onMounted(() => {
  panelRef.value.open()
})
</script>

<template>
  <DialogPanel ref="panelRef" />
</template>
```

**ХОРОШО:**
```vue
<!-- DialogPanel.vue -->
<script setup lang="ts">
function open() {}

defineExpose({ open })
</script>
```

```vue
<!-- Parent.vue -->
<script setup lang="ts">
import { onMounted, useTemplateRef } from 'vue'
import DialogPanel from './DialogPanel.vue'

// Vue 3.5+ с useTemplateRef
const panelRef = useTemplateRef('panelRef')

// До Vue 3.5 с ручной типизацией и ref
// const panelRef = ref<InstanceType<typeof DialogPanel> | null>(null)

onMounted(() => {
  panelRef.value?.open()
})
</script>

<template>
  <DialogPanel ref="panelRef" />
</template>
```

## Emits: явные события вверх

События компонентов не всплывают. Если родителю нужно узнать о событии, передайте его явно.

**ПЛОХО:**
```vue
<!-- Родитель ожидает событие "saved" от внука, но оно не всплывет -->
<Child @saved="onSaved" />
```

**ХОРОШО:**
```vue
<!-- Child.vue -->
<script setup>
const emit = defineEmits(['saved'])

function onGrandchildSaved(payload) {
  emit('saved', payload)
}
</script>

<template>
  <Grandchild @saved="onGrandchildSaved" />
</template>
```

**Именование событий**: используйте kebab-case в шаблонах и camelCase в скрипте:
```vue
<script setup>
const emit = defineEmits(['updateUser'])
</script>

<template>
  <ProfileForm @update-user="emit('updateUser', $event)" />
</template>
```

## `v-model`: предсказуемые двусторонние привязки

По умолчанию используйте `defineModel` для привязок компонентов и генерируйте события при вводе
данных. Используйте паттерн `modelValue` + `update:modelValue`, только если у вас Vue < 3.4.

**ПЛОХО:**
```vue
<script setup>
const props = defineProps({ value: String })
</script>

<template>
  <input :value="props.value" @input="$emit('input', $event.target.value)" />
</template>
```

**ХОРОШО (Vue 3.4+):**
```vue
<script setup>
const model = defineModel({ type: String })
</script>

<template>
  <input v-model="model" />
</template>
```

**ХОРОШО (Vue < 3.4):**
```vue
<script setup>
const props = defineProps({ modelValue: String })
const emit = defineEmits(['update:modelValue'])
</script>

<template>
  <input
      :value="props.modelValue"
      @input="emit('update:modelValue', $event.target.value)"
  />
</template>
```

Если вам нужно немедленно получить обновленное значение после изменения, используйте значение
события input или `nextTick` в родителе.

## Provide/Inject: общий контекст без пробрасывания свойств

Используйте `provide/inject` для состояния, разделяемого разными ветками дерева, но храните мутации
централизованно в провайдере и предоставляйте явные действия.

**ПЛОХО:**
```vue
// Provider.vue
provide('theme', reactive({ dark: false }))

// Consumer.vue
const theme = inject('theme')
// Изменение общего состояния с любой глубины становится трудноотслеживаемым
theme.dark = true
```

**ХОРОШО:**
```vue
// Provider.vue
const theme = reactive({ dark: false })
const toggleTheme = () => { theme.dark = !theme.dark }

provide(themeKey, readonly(theme))
provide(themeActionsKey, { toggleTheme })

// Consumer.vue
const theme = inject(themeKey)
const { toggleTheme } = inject(themeActionsKey)
```

Используйте символы для ключей, чтобы избежать конфликтов в больших приложениях:

```ts
export const themeKey = Symbol('theme')
export const themeActionsKey = Symbol('theme-actions')
```

## Используйте TypeScript-контракты для публичных API компонентов

В проектах с TypeScript типизируйте границы компонентов напрямую с помощью `defineProps`,
`defineEmits` и `InjectionKey`, чтобы некорректные полезные нагрузки и несоответствующие инъекции
вызывали ошибки на этапе компиляции.

**ПЛОХО:**
```vue
<script setup lang="ts">
import { inject } from 'vue'

const props = defineProps({
  userId: String
})

const emit = defineEmits(['save'])
const settings = inject('settings')

// Форма полезной нагрузки не проверяется
emit('save', 123)

// Ключ является строкой и не типобезопасен
settings?.theme = 'dark'
</script>
```

**ХОРОШО:**
```vue
<script setup lang="ts">
import { inject, provide } from 'vue'
import type { InjectionKey } from 'vue'

interface Props {
  userId: string
}

interface Emits {
  save: [payload: { id: string; draft: boolean }]
}

interface Settings {
  theme: 'light' | 'dark'
}

const settingsKey: InjectionKey<Settings> = Symbol('settings')

const props = defineProps<Props>()
const emit = defineEmits<Emits>()

provide(settingsKey, { theme: 'light' })

const settings = inject(settingsKey)
if (settings) {
  emit('save', { id: props.userId, draft: false })
}
</script>
```
