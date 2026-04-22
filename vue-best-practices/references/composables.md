---
title: Шаблоны организации композаблов
impact: MEDIUM
impactDescription: Хорошо структурированные композаблы повышают поддерживаемость, повторяемость и производительность обновлений
type: best-practice
tags: [vue3, composables, composition-api, code-organization, api-design, readonly, utilities]
---

# Composable Organization Patterns

**Impact: СРЕДНИЙ** — Относитесь к композаблам как к переиспользуемым, сохраняющим состояние
строительным блокам и организуйте их код по функциональным областям. Это поддерживает большие
компоненты в удобном для сопровождения состоянии и предотвращает трудноотлаживаемые мутации и
проблемы с дизайном API.

## Список задач

- Комбинируйте сложное поведение из маленьких, узкоспециализированных композаблов
- Используйте объекты параметров для композаблов с множеством опциональных аргументов
- Возвращайте состояние только для чтения (readonly), когда обновления должны проходить через явные
действия
- Храните чистые утилитарные функции как простые утилиты, а не композаблы
- Организуйте код компонентов и композаблов по функциональным областям и выносите композаблы, когда
компоненты разрастаются

## Комбинируйте композаблы из маленьких примитивов

**ПЛОХО:**
```vue
<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'

const x = ref(0)
const y = ref(0)
const inside = ref(false)
const el = ref(null)

function onMove(e) {
  x.value = e.pageX
  y.value = e.pageY
  if (!el.value) return
  const r = el.value.getBoundingClientRect()
  inside.value = x.value >= r.left && x.value <= r.right &&
      y.value >= r.top && y.value <= r.bottom
}

onMounted(() => window.addEventListener('mousemove', onMove))
onUnmounted(() => window.removeEventListener('mousemove', onMove))
</script>
```

**ХОРОШО:**
```javascript
// composables/useEventListener.js
import { onMounted, onUnmounted, toValue } from 'vue'

export function useEventListener(target, event, callback) {
  onMounted(() => toValue(target).addEventListener(event, callback))
  onUnmounted(() => toValue(target).removeEventListener(event, callback))
}
```

```javascript
// composables/useMouse.js
import { ref } from 'vue'
import { useEventListener } from './useEventListener'

export function useMouse() {
  const x = ref(0)
  const y = ref(0)

  useEventListener(window, 'mousemove', (e) => {
    x.value = e.pageX
    y.value = e.pageY
  })

  return { x, y }
}
```

```javascript
// composables/useMouseInElement.js
import { computed } from 'vue'
import { useMouse } from './useMouse'

export function useMouseInElement(elementRef) {
  const { x, y } = useMouse()

  const isOutside = computed(() => {
    if (!elementRef.value) return true
    const rect = elementRef.value.getBoundingClientRect()
    return x.value < rect.left || x.value > rect.right ||
      y.value < rect.top || y.value > rect.bottom
  })

  return { x, y, isOutside }
}
```

## Используйте паттерн объекта параметров для аргументов композабла

**ПЛОХО:**
```javascript
export function useFetch(url, method, headers, timeout, retries, immediate) {
  // трудно читать и легко перепутать порядок
}

useFetch('/api/users', 'GET', null, 5000, 3, true)
```

**ХОРОШО:**
```javascript
export function useFetch(url, options = {}) {
  const {
    method = 'GET',
    headers = {},
    timeout = 30000,
    retries = 0,
    immediate = true
  } = options

  // реализация
  return { method, headers, timeout, retries, immediate }
}

useFetch('/api/users', {
  method: 'POST',
  timeout: 5000,
  retries: 3
})
```

```typescript
interface UseCounterOptions {
  initial?: number
  min?: number
  max?: number
  step?: number
}

export function useCounter(options: UseCounterOptions = {}) {
  const { initial = 0, min = -Infinity, max = Infinity, step = 1 } = options
  // реализация
}
```

## Возвращайте состояние только для чтения с явными действиями

**ПЛОХО:**
```javascript
export function useCart() {
  const items = ref([])
  const total = computed(() => items.value.reduce((sum, item) => sum + item.price, 0))
  return { items, total } // любой потребитель может мутировать напрямую
}

const { items } = useCart()
items.value.push({ id: 1, price: 10 })
```

**ХОРОШО:**
```javascript
import { ref, computed, readonly } from 'vue'

export function useCart() {
  const _items = ref([])

  const total = computed(() =>
    _items.value.reduce((sum, item) => sum + item.price * item.quantity, 0)
  )

  function addItem(product, quantity = 1) {
    const existing = _items.value.find(item => item.id === product.id)
    if (existing) {
      existing.quantity += quantity
      return
    }
    _items.value.push({ ...product, quantity })
  }

  function removeItem(productId) {
    _items.value = _items.value.filter(item => item.id !== productId)
  }

  return {
    items: readonly(_items),
    total,
    addItem,
    removeItem
  }
}
```

## Храните утилиты как утилиты

**ПЛОХО:**
```javascript
export function useFormatters() {
  const formatDate = (date) => new Intl.DateTimeFormat('en-US').format(date)
  const formatCurrency = (amount) =>
    new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(amount)
  return { formatDate, formatCurrency }
}

const { formatDate } = useFormatters()
```

**ХОРОШО:**
```javascript
// utils/formatters.js
export function formatDate(date) {
  return new Intl.DateTimeFormat('en-US').format(date)
}

export function formatCurrency(amount) {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD'
  }).format(amount)
}
```

```javascript
// composables/useInvoiceSummary.js
import { computed } from 'vue'
import { formatCurrency } from '@/utils/formatters'

export function useInvoiceSummary(invoiceRef) {
  const totalLabel = computed(() => formatCurrency(invoiceRef.value.total))
  return { totalLabel }
}
```

## Организуйте код компонентов и композаблов по функциональным областям

**ПЛОХО:**
```vue
<script setup>
import { ref, computed, watch, onMounted } from 'vue'

const searchQuery = ref('')
const items = ref([])
const selected = ref(null)
const showModal = ref(false)
const sortBy = ref('name')
const filter = ref('all')
const loading = ref(false)

const filtered = computed(() => items.value.filter(i => i.category === filter.value))
function openModal() { showModal.value = true }
const sorted = computed(() => [...filtered.value].sort(/* ... */))
watch(searchQuery, () => { /* ... */ })
onMounted(() => { /* ... */ })
</script>
```

**ХОРОШО:**
```vue
<script setup>
import { useItems } from '@/composables/useItems'
import { useSearch } from '@/composables/useSearch'
import { useSelectionModal } from '@/composables/useSelectionModal'

// Данные
const { items, loading, fetchItems } = useItems()

// Поиск/фильтрация/сортировка
const { query, visibleItems } = useSearch(items)

// Выбор + модальное окно
const { selectedItem, isModalOpen, selectItem, closeModal } = useSelectionModal()
</script>
```

```javascript
// composables/useItems.js
import { ref, onMounted } from 'vue'

export function useItems() {
  const items = ref([])
  const loading = ref(false)

  async function fetchItems() {
    loading.value = true
    try {
      items.value = await api.getItems()
    } finally {
      loading.value = false
    }
  }

  onMounted(fetchItems)
  return { items, loading, fetchItems }
}
```
