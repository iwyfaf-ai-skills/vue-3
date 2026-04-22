---
title: Стратегия управления состоянием
impact: HIGH
impactDescription: Неправильный выбор хранилища может привести к утечкам запросов при SSR, хрупким потокам мутаций и проблемам масштабирования
type: best-practice
tags: [vue3, state-management, pinia, composables, ssr, vueuse]
---

# Стратегия управления состоянием

**Impact: ВЫСОКИЙ** - Используйте самое лёгкое решение для состояния, которое подходит для
архитектуры вашего приложения. SPA-приложения могут использовать легковесные глобальные композаблы,
в то время как SSR/Nuxt приложения должны по умолчанию использовать Pinia для безопасной изоляции
запросов и предсказуемого инструментария.

## Список задач

- Сначала храните состояние локально, переносите в общее/глобальное только при необходимости
- Используйте синглтон-композаблы только в не-SSR приложениях
- Предоставляйте глобальное состояние только для чтения и мутируйте через явные действия
- Отдавайте предпочтение Pinia для SSR/Nuxt, больших приложений и сложных сценариев отладки/работы
с плагинами
- Избегайте прямого экспорта мутируемого реактивного состояния на уровне модуля

## Выбирайте самый лёгкий подход к хранилищу

- **Функциональный композабл:** По умолчанию для переиспользуемой логики с локальным состоянием на
уровне фичи.
- **Синглтон-композабл или VueUse  `createGlobalState`:** Небольшие не-SSR приложения, которым
нужно общее состояние приложения.
- **Pinia:** SSR/Nuxt приложения, средние и крупные приложения, а также случаи, требующие DevTools,
плагинов или трассировки действий.

## Избегайте экспорта мутируемого состояния на уровне модуля

**ПЛОХО:**
```ts
// store/cart.ts
import { reactive } from 'vue'

export const cart = reactive({
  items: [] as Array<{ id: string; qty: number }>
})
```

**ХОРОШО:**
```ts
// composables/useCartStore.ts
import { reactive, readonly } from 'vue'

let _store: ReturnType<typeof createCartStore> | null = null

function createCartStore() {
  const state = reactive({
    items: [] as Array<{ id: string; qty: number }>
  })

  function addItem(id: string, qty = 1) {
    const existing = state.items.find((item) => item.id === id)
    if (existing) {
      existing.qty += qty
      return
    }
    state.items.push({ id, qty })
  }

  return {
    state: readonly(state),
    addItem
  }
}

export function useCartStore() {
  if (!_store) _store = createCartStore()
  return _store
}
```

## Не используйте синглтоны времени выполнения в SSR

Синглтоны на уровне модуля живут всё время выполнения приложения. В SSR это может привести к
утечкам состояния между запросами.

**ПЛОХО:**
```ts
// общий синглтон, переиспользуемый между запросами
const cartStore = useCartStore()

export function useServerCart() {
  return cartStore
}
```

**ХОРОШО:**

> Требуется зависимость `pinia`.

```ts
// stores/cart.ts
import { defineStore } from 'pinia'

export const useCartStore = defineStore('cart', {
  state: () => ({
    items: [] as Array<{ id: string; qty: number }>
  }),
  actions: {
    addItem(id: string, qty = 1) {
      const existing = this.items.find((item) => item.id === id)
      if (existing) {
        existing.qty += qty
        return
      }
      this.items.push({ id, qty })
    }
  }
})
```

## Используйте `createGlobalState` для небольшого глобального состояния в SPA

> Требуется зависимость `@vueuse/core`.

Если приложение не использует SSR и уже работает с VueUse, `createGlobalState` избавляет от
шаблонного кода для синглтона.

```ts
import { createGlobalState } from '@vueuse/core'
import { computed, ref } from 'vue'

export const useAuthState = createGlobalState(() => {
  const token = ref<string | null>(null)
  const isAuthenticated = computed(() => token.value !== null)

  function setToken(next: string | null) {
    token.value = next
  }

  return {
    token,
    isAuthenticated,
    setToken
  }
})
```
