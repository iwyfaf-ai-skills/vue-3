---
title: Рекомендации по асинхронным компонентам
impact: MEDIUM
impactDescription: Неудачная стратегия использования асинхронных компонентов может задерживать интерактивность в SSR-приложениях и вызывать мерцание индикаторов загрузки
type: best-practice
tags: [vue3, async-components, ssr, hydration, performance, ux]
---

# Рекомендации по асинхронным компонентам

**Impact: СРЕДНЕЕ** - Асинхронные компоненты должны снижать затраты на JavaScript без ухудшения
воспринимаемой производительности. Сосредоточьтесь на времени гидратации в SSR и стабильном UX при
загрузке.

## Список задач

- Используйте стратегии ленивой гидратации для некритичных поддеревьев компонентов в SSR
- Импортируйте только те хелперы гидратации, которые действительно используете
- Устанавливайте `loadingComponent` с задержкой, близкой к значению по умолчанию `200ms`, если
только реальные данные о UX не говорят об обратном
- Настраивайте `delay` и `timeout` вместе для предсказуемого поведения загрузки

## Используйте стратегии ленивой гидратации в SSR

В Vue 3.5+ асинхронные компоненты могут откладывать гидратацию до момента бездействия, появления в
области видимости, срабатывания медиавыражения или взаимодействия с пользователем.

**ПЛОХО:**
```vue
<script setup lang="ts">
import { defineAsyncComponent } from 'vue'

const AsyncComments = defineAsyncComponent({
  loader: () => import('./Comments.vue')
})
</script>
```

**ХОРОШО:**
```vue
<script setup lang="ts">
import {
  defineAsyncComponent,
  hydrateOnVisible,
  hydrateOnIdle
} from 'vue'

const AsyncComments = defineAsyncComponent({
  loader: () => import('./Comments.vue'),
  hydrate: hydrateOnVisible({ rootMargin: '100px' })
})

const AsyncFooter = defineAsyncComponent({
  loader: () => import('./Footer.vue'),
  hydrate: hydrateOnIdle(5000)
})
</script>
```

## Избегайте мерцания индикатора загрузки

Не показывайте интерфейс загрузки сразу для компонентов, которые обычно загружаются быстро.

**ПЛОХО:**
```vue
<script setup lang="ts">
import { defineAsyncComponent } from 'vue'
import LoadingSpinner from './LoadingSpinner.vue'

const AsyncDashboard = defineAsyncComponent({
  loader: () => import('./Dashboard.vue'),
  loadingComponent: LoadingSpinner,
  delay: 0
})
</script>
```

**ХОРОШО:**
```vue
<script setup lang="ts">
import { defineAsyncComponent } from 'vue'
import LoadingSpinner from './LoadingSpinner.vue'
import ErrorDisplay from './ErrorDisplay.vue'

const AsyncDashboard = defineAsyncComponent({
  loader: () => import('./Dashboard.vue'),
  loadingComponent: LoadingSpinner,
  errorComponent: ErrorDisplay,
  delay: 200,
  timeout: 30000
})
</script>
```

## Рекомендации по задержке

| Сценарий                           | Рекомендуемая задержка  |
|------------------------------------|--------------------|
| Небольшой компонент, быстрая сеть  | `200ms`            |
| Заведомо тяжелый компонент         | `100ms`            |
| Фоновый или некритичный интерфейс  | `300-500ms`        |
