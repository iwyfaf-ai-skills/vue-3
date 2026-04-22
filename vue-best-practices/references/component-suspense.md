---
title: Лучшие практики использования компонента Suspense
impact: MEDIUM
impactDescription: Suspense координирует асинхронные зависимости с резервным UI; неправильная настройка приводит к отсутствию состояний загрузки или запутанному UX
type: best-practice
tags: [vue3, suspense, async-components, async-setup, loading, fallback, router, transition, keepalive]
---

# Suspense Component Best Practices

**Impact: СРЕДНИЙ** - `<Suspense>` координирует асинхронные зависимости (асинхронные компоненты или
асинхронный setup) и отображает резервный интерфейс, пока они разрешаются. Неправильная настройка
приводит к отсутствию состояний загрузки, пустому рендеру или незаметным ошибкам UX.

## Список задач

- Оборачивайте содержимое слотов по умолчанию и fallback в один корневой узел
- Используйте `timeout`, когда вам нужно, чтобы резервный интерфейс появлялся при возвратах (reverts)
- Принудительно заменяйте корневой элемент с помощью `:key`, когда нужно, чтобы Suspense сработал
повторно
- Добавляйте `suspensible` к вложенным границам Suspense (Vue 3.3+)
- Используйте события `@pending`, `@resolve` и `@fallback` для программного отслеживания состояния
загрузки
- Вкладывайте `RouterView` -> `Transition` -> `KeepAlive` -> `Suspense` именно в таком порядке
- Используйте Suspense в продакшене осторожно, централизованно и с документацией

## Единый корневой элемент в слотах по умолчанию и fallback

Suspense отслеживает единственный непосредственный дочерний элемент в обоих слотах. Оборачивайте
несколько элементов в один элемент или компонент.

**ПЛОХО:**
```vue
<template>
  <Suspense>
    <AsyncHeader />
    <AsyncList />

    <template #fallback>
      <LoadingSpinner />
      <LoadingHint />
    </template>
  </Suspense>
</template>
```

**ХОРОШО:**
```vue
<template>
  <Suspense>
    <div>
      <AsyncHeader />
      <AsyncList />
    </div>

    <template #fallback>
      <div>
        <LoadingSpinner />
        <LoadingHint />
      </div>
    </template>
  </Suspense>
</template>
```

## Тайминг появления fallback при возвратах (reverts) через `timeout`

Когда Suspense уже разрешён и начинается новая асинхронная работа, предыдущее содержимое остаётся
видимым, пока не истечёт таймаут. Используйте `timeout="0"` для немедленного показа fallback
или небольшую задержку, чтобы избежать мерцания.

**ПЛОХО:**
```vue
<template>
  <Suspense>
    <component :is="currentView" :key="viewKey" />

    <template #fallback>
      Загрузка...
    </template>
  </Suspense>
</template>
```

**ХОРОШО:**
```vue
<template>
  <Suspense :timeout="200">
    <component :is="currentView" :key="viewKey" />

    <template #fallback>
      Загрузка...
    </template>
  </Suspense>
</template>
```

## Состояние ожидания (pending) повторно срабатывает только при замене корневого элемента

После разрешения Suspense переходит в состояние ожидания только тогда, когда меняется корневой
узел слота по умолчанию. Если асинхронная работа происходит глубже в дереве, резервный интерфейс
не появится.

**ПЛОХО:**
```vue
<template>
  <Suspense>
    <TabContainer>
      <AsyncDashboard v-if="tab === 'dashboard'" />
      <AsyncSettings v-else />
    </TabContainer>

    <template #fallback>
      Загрузка...
    </template>
  </Suspense>
</template>
```

**ХОРОШО:**
```vue
<template>
  <Suspense>
    <component :is="tabs[tab]" :key="tab" />

    <template #fallback>
      Загрузка...
    </template>
  </Suspense>
</template>
```

## Используйте `suspensible` для вложенных Suspense (Vue 3.3+)

Для вложенных границ Suspense требуется `suspensible` на внутренней границе, чтобы родительский
компонент мог координировать состояние загрузки. Без этого внутренний асинхронный контент может
рендерить пустые узлы до разрешения.

**ПЛОХО:**
```vue
<template>
  <Suspense>
    <LayoutShell>
      <Suspense>
        <AsyncWidget />
        <template #fallback>Загрузка виджета...</template>
      </Suspense>
    </LayoutShell>

    <template #fallback>Загрузка макета...</template>
  </Suspense>
</template>
```

**ХОРОШО:**
```vue
<template>
  <Suspense>
    <LayoutShell>
      <Suspense suspensible>
        <AsyncWidget />
        <template #fallback>Загрузка виджета...</template>
      </Suspense>
    </LayoutShell>

    <template #fallback>Загрузка макета...</template>
  </Suspense>
</template>
```

## Отслеживание загрузки с помощью событий Suspense

Используйте `@pending`, `@resolve` и `@fallback` для аналитики, глобальных индикаторов загрузки
или координации UI за пределами границы Suspense.

```vue
<script setup>
import { ref } from 'vue'

const isLoading = ref(false)

const onPending = () => {
  isLoading.value = true
}

const onResolve = () => {
  isLoading.value = false
}
</script>

<template>
  <LoadingBar v-if="isLoading" />

  <Suspense @pending="onPending" @resolve="onResolve">
    <AsyncPage />
    <template #fallback>
      <PageSkeleton />
    </template>
  </Suspense>
</template>
```

## Рекомендуемый порядок вложения с RouterView, Transition, KeepAlive

При комбинировании этих компонентов порядок вложения должен быть таким: `RouterView` -> `Transition`
-> `KeepAlive` -> `Suspense`, чтобы каждый враппер работал корректно.

**ПЛОХО:**
```vue
<template>
  <RouterView v-slot="{ Component }">
    <Suspense>
      <KeepAlive>
        <Transition mode="out-in">
          <component :is="Component" />
        </Transition>
      </KeepAlive>
    </Suspense>
  </RouterView>
</template>
```

**ХОРОШО:**
```vue
<template>
  <RouterView v-slot="{ Component }">
    <Transition mode="out-in">
      <KeepAlive>
        <Suspense>
          <component :is="Component" />
          <template #fallback>Загрузка...</template>
        </Suspense>
      </KeepAlive>
    </Transition>
  </RouterView>
</template>
```

## Используйте Suspense в продакшене осторожно

В продакшен-коде сводите количество границ Suspense к минимуму, документируйте, где они
используются, и всегда имейте резервную стратегию загрузки на случай, если вам потребуется заменить
или рефакторить эти компоненты.