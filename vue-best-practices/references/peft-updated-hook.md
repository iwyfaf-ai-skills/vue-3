---
title: Избегайте дорогостоящих операций в хуке updated
impact: MEDIUM
impactDescription: Тяжелые вычисления в хуке updated приводят к проблемам с производительностью и потенциальным бесконечным циклам
type: capability
tags: [vue3, vue2, lifecycle, updated, performance, optimization, reactivity]
---

# Избегайте дорогостоящих операций в хуке `updated`

**Impact: СРЕДНЯЯ** - Хук `updated` выполняется после каждого изменения реактивного состояния,
которое вызывает повторный рендер. Размещение здесь дорогостоящих операций, API-вызовов или мутаций
состояния может привести к серьезному падению производительности, бесконечным циклам и падению
частоты кадров ниже оптимального порога в 60 fps.

Используйте `updated`/`onUpdated` экономно — только для операций, которые необходимо выполнить
после обновления DOM и которые невозможно обработать с помощью наблюдателей (`watch`) или
вычисляемых свойств. В большинстве случаев для работы с реактивными данными предпочтительнее
использовать наблюдатели (`watch`/`watchEffect`), которые дают больше контроля над тем, что
запускает колбэк.

## Список задач

- Никогда не выполняйте API-вызовы внутри updated
- Никогда не мутируйте реактивное состояние внутри `updated` (это вызывает бесконечные циклы)
- Используйте условные проверки, чтобы убедиться, что обновления релевантны перед выполнением
действий
- Отдавайте предпочтение `watch` или `watchEffect` для реакции на конкретные изменения данных
- Используйте троттлинг/дебонсинг, если операции внутри `updated` дорогостоящие
- Оставьте `updated` для низкоуровневых задач синхронизации с DOM

**ПЛОХО:**
```javascript
// ПЛОХО: API-вызов в updated — срабатывает при каждом рендере
export default {
  data() {
    return { items: [], lastUpdate: null }
  },
  updated() {
    // Это выполняется после каждого изменения состояния!
    fetch('/api/sync', {
      method: 'POST',
      body: JSON.stringify(this.items)
    })
  }
}
```

```javascript
// ПЛОХО: Мутация состояния в updated — бесконечный цикл
export default {
  data() {
    return { renderCount: 0 }
  },
  updated() {
    // Это вызывает ещё одно обновление, которое снова триггерит updated!
    this.renderCount++ // Бесконечный цикл
  }
}
```

```javascript
// ПЛОХО: Тяжелые вычисления при каждом обновлении
export default {
  updated() {
    // Тяжелая операция выполняется при каждом нажатии клавиши, при каждом изменении состояния
    this.processedData = this.heavyComputation(this.rawData)
    this.analytics = this.calculateMetrics(this.allData)
  }
}
```

**ХОРОШО:**
```javascript
import debounce from 'lodash-es/debounce'

// ХОРОШО: Используйте наблюдатель для конкретных изменений данных
export default {
  data() {
    return { items: [] }
  },
  watch: {
    // Срабатывает только когда items действительно меняется
    items: {
      handler(newItems) {
        this.syncToServer(newItems)
      },
      deep: true
    }
  },
  methods: {
    syncToServer: debounce(function(items) {
      fetch('/api/sync', {
        method: 'POST',
        body: JSON.stringify(items)
      })
    }, 500)
  }
}
```

```vue
<!-- ХОРОШО: Composition API с целевыми наблюдателями -->
<script setup>
  import { ref, watch, onUpdated } from 'vue'
  import { useDebounceFn } from '@vueuse/core'

  const items = ref([])
  const scrollContainer = ref(null)

  // Следим за конкретными данными — а не за любыми обновлениями
  watch(items, (newItems) => {
    syncToServer(newItems)
  }, { deep: true })

  const syncToServer = useDebounceFn((items) => {
    fetch('/api/sync', { method: 'POST', body: JSON.stringify(items) })
  }, 500)

  // Используем onUpdated только для синхронизации с DOM
  onUpdated(() => {
    // Прокручиваем вниз, только если изменилась высота контента
    if (scrollContainer.value) {
      scrollContainer.value.scrollTop = scrollContainer.value.scrollHeight
    }
  })
</script>
```

```javascript
// ХОРОШО: Условная проверка внутри хука updated
export default {
  data() {
    return {
      content: '',
      lastSyncedContent: ''
    }
  },
  updated() {
    // Действуем только при выполнении конкретного условия
    if (this.content !== this.lastSyncedContent) {
      this.syncContent()
      this.lastSyncedContent = this.content
    }
  },
  methods: {
    syncContent: debounce(function() {
      // Логика синхронизации
    }, 300)
  }
}
```

## Допустимые случаи использования хука `updated`

```javascript
// ХОРОШО: Низкоуровневая синхронизация с DOM
export default {
  updated() {
    // Синхронизируем стороннюю библиотеку с DOM от Vue
    this.thirdPartyWidget.refresh()

    // Обновляем позицию прокрутки после изменения контента
    this.$nextTick(() => {
      this.maintainScrollPosition()
    })
  }
}
```

## Отдавайте предпочтение вычисляемым свойствам для производных данных

```javascript
// ПЛОХО: Вычисление производных данных внутри updated
export default {
  data() {
    return { numbers: [1, 2, 3, 4, 5] }
  },
  updated() {
    this.sum = this.numbers.reduce((a, b) => a + b, 0) // Вызывает ещё одно обновление!
  }
}

// ХОРОШО: Используйте вычисляемое свойство
export default {
  data() {
    return { numbers: [1, 2, 3, 4, 5] }
  },
  computed: {
    sum() {
      return this.numbers.reduce((a, b) => a + b, 0)
    }
  }
}
```
