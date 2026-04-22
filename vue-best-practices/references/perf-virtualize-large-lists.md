---
title: Виртуализация больших списков для избежания перегрузки DOM
impact: HIGH
impactDescription: Рендеринг тысяч элементов списка создаёт избыточное количество DOM-узлов, что приводит к медленной отрисовке и высокому потреблению памяти
type: efficiency
tags: [vue3, performance, virtual-list, large-data, dom, optimization]
---

# Виртуализация больших списков для избежания перегрузки DOM

**Impact: ВЫСОКОЕ** - Рендеринг всех элементов большого списка (сотен или тысяч) создаёт огромное
количество DOM-узлов. Каждый узел потребляет память, замедляет начальный рендеринг и делает
обновления затратными. Виртуализация списка отображает только видимые элементы, что значительно
улучшает производительность.

Используйте библиотеки виртуализации при работе со списками, которые могут превышать 50–100
элементов, особенно если элементы имеют сложное содержимое.

## Список задач

- Определите списки, которые рендерят более 50–100 элементов
- Установите библиотеку виртуализации (`vue-virtual-scroller`, @tanstack/vue-virtual)
- Замените стандартный `v-for` на виртуализированный компонент
- Убедитесь, что элементы списка имеют одинаковую или оцениваемую высоту
- Тестируйте с реалистичными объёмами данных во время разработки

## Рекомендуемые библиотеки

| Библиотека                | Лучше всего подходит для                | Примечания                                        |
|---------------------------|-----------------------------------------|---------------------------------------------------|
| `vue-virtual-scroller`    | Общего использования, простой настройки | Самая популярная, хорошие настройки по умолчанию  |
| `@tanstack/vue-virtual`   | Сложных макетов, headless-подхода       | Независимая от фреймворка, гибкая                 |
| `vue-virtual-scroll-grid` | Сеточных макетов                        | 2D-виртуализация                                  |
| `vueuc/VVirtualList`      | Проектов на Naive UI                    | Часть экосистемы Naive UI                         |

**ПЛОХО:**
```vue
<template>
  <!-- ПЛОХО: рендерит ВСЕ 10 000 элементов немедленно -->
  <div class="user-list">
    <UserCard
        v-for="user in users"
        :key="user.id"
        :user="user"
    />
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import UserCard from './UserCard.vue'

const users = ref([])

onMounted(async () => {
  // Создаётся 10 000 DOM-узлов, браузер тормозит
  users.value = await fetchAllUsers()
})
</script>
```

**ХОРОШО:**
```vue
<template>
  <!-- ХОРОШО: единовременно рендерится только ~20 видимых элементов -->
  <RecycleScroller
      class="user-list"
      :items="users"
      :item-size="80"
      key-field="id"
      v-slot="{ item }"
  >
    <UserCard :user="item" />
  </RecycleScroller>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { RecycleScroller } from 'vue-virtual-scroller'
import 'vue-virtual-scroller/dist/vue-virtual-scroller.css'
import UserCard from './UserCard.vue'

const users = ref([])

onMounted(async () => {
  // 10 000 элементов в памяти, но только ~20 DOM-узлов
  users.value = await fetchAllUsers()
})
</script>

<style scoped>
.user-list {
  height: 600px; /* Контейнер должен иметь фиксированную высоту */
}
</style>
```

## Использование @tanstack/vue-virtual

```vue
<template>
  <div ref="parentRef" class="list-container">
    <div
        :style="{
        height: `${rowVirtualizer.getTotalSize()}px`,
        position: 'relative'
      }"
    >
      <div
          v-for="virtualRow in rowVirtualizer.getVirtualItems()"
          :key="virtualRow.key"
          :style="{
          position: 'absolute',
          top: 0,
          left: 0,
          width: '100%',
          height: `${virtualRow.size}px`,
          transform: `translateY(${virtualRow.start}px)`
        }"
      >
        <UserCard :user="users[virtualRow.index]" />
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import { useVirtualizer } from '@tanstack/vue-virtual'

const users = ref([/* 10 000 пользователей */])
const parentRef = ref(null)

const rowVirtualizer = useVirtualizer({
  count: users.value.length,
  getScrollElement: () => parentRef.value,
  estimateSize: () => 80,  // Оценочная высота строки
  overscan: 5  // Рендерить 5 дополнительных элементов сверху и снизу видимой области
})
</script>

<style scoped>
.list-container {
  height: 600px;
  overflow: auto;
}
</style>
```

## Динамическая высота с vue-virtual-scroller

```vue
<template>
  <!-- Для элементов переменной высоты используйте DynamicScroller -->
  <DynamicScroller
      :items="messages"
      :min-item-size="54"
      key-field="id"
  >
    <template #default="{ item, index, active }">
      <DynamicScrollerItem
          :item="item"
          :active="active"
          :data-index="index"
      >
        <ChatMessage :message="item" />
      </DynamicScrollerItem>
    </template>
  </DynamicScroller>
</template>

<script setup>
import { DynamicScroller, DynamicScrollerItem } from 'vue-virtual-scroller'
</script>
```

## Performance Comparison

| Подход                        | 100 элементов  | 1,000 элементов  | 10,000 элементов      |
|-------------------------------|----------------|------------------|-----------------------|
| Обычный `v-for`               | ~100 DOM узлов | ~1,000 DOM узлов | ~10,000 DOM узлов     |
| Виртуализированный            | ~20 DOM узлов  | ~20 DOM узлов    | ~20 DOM узлов         |
| Начальный рендеринг           | Быстро         | Медленно         | Очень медленно / краш |
| Виртуализированный рендеринг  | Быстро         | Быстро           | Быстро                |

## Когда НЕ нужна виртуализация

- Списки менее чем из 50 элементов с простым содержимым
- Списки, где все элементы должны быть одновременно доступны для скринридеров
- Макеты для печати, где всё содержимое должно быть отрендерено
- SEO-критичный контент, который должен присутствовать в исходном HTML
