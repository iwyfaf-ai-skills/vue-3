---
title: Рекомендации по использованию компонента Transition
impact: MEDIUM
impactDescription: Transition анимирует один элемент или компонент; неправильная структура или отсутствие ключей нарушают анимацию
type: best-practice
tags: [vue3, transition, animation, performance, keys]
---

# Transition Component Best Practices

**Impact: СРЕДНИЙ** - `<Transition>` анимирует появление и исчезновение одного элемента или
компонента. Он идеально подходит для переключения состояний интерфейса, смены представлений или
анимации одного компонента за раз.

## Список задач

- Оберните один элемент или компонент внутри `<Transition>`
- Укажите `key` при переключении между элементами одного типа
- Используйте `mode="out-in"`, когда нужна последовательная смена элементов
- Отдавайте предпочтение `transform` и `opacity` для плавной анимации

## Используйте Transition только для одного корневого элемента

`<Transition>` поддерживает только одного прямого потомка. Объединяйте несколько узлов внутри
одного элемента или компонента.

**ПЛОХО:**
```vue
<template>
  <Transition name="fade">
    <h3>Заголовок</h3>
    <p>Описание</p>
  </Transition>
</template>
```

**ХОРОШО:**
```vue
<template>
  <Transition name="fade">
    <div>
      <h3>Заголовок</h3>
      <p>Описание</p>
    </div>
  </Transition>
</template>
```

## Принудительно применяйте анимацию между элементами одного типа

Vue переиспользует тот же DOM-элемент, если тип тега не меняется. Добавьте `key`, чтобы Vue
воспринимал их как разные элементы и запускал анимацию появления/исчезновения.

**ПЛОХО:**
```vue
<template>
  <Transition name="fade">
    <p v-if="isActive">Активен</p>
    <p v-else>Неактивен</p>
  </Transition>
</template>
```

**ХОРОШО:**
```vue
<template>
  <Transition name="fade" mode="out-in">
    <p v-if="isActive" key="active">Активен</p>
    <p v-else key="inactive">Неактивен</p>
  </Transition>
</template>
```

## Используйте `mode` для предотвращения наложения при смене элементов

При переключении между компонентами или представлениями используйте `mode="out-in"`, чтобы оба
элемента не отображались одновременно.

**ПЛОХО:**
```vue
<template>
  <Transition name="fade">
    <component :is="currentView" />
  </Transition>
</template>
```

**ХОРОШО:**
```vue
<template>
  <Transition name="fade" mode="out-in">
    <component :is="currentView" :key="currentView" />
  </Transition>
</template>
```

## Анимируйте `transform` и `opacity` для лучшей производительности

Избегайте свойств, которые вызывают пересчёт макета (layout), таких как `height`, `margin` или
`top`. Используйте `transform` и `opacity` для плавных, дружественных к GPU анимаций.

**ПЛОХО:**
```css
.slide-enter-active,
.slide-leave-active {
  transition: height 0.3s ease;
}

.slide-enter-from,
.slide-leave-to {
  height: 0;
}
```

**ХОРОШО:**
```css
.slide-enter-active,
.slide-leave-active {
  transition: transform 0.3s ease, opacity 0.3s ease;
}

.slide-enter-from {
  transform: translateX(-12px);
  opacity: 0;
}

.slide-leave-to {
  transform: translateX(12px);
  opacity: 0;
}
```
