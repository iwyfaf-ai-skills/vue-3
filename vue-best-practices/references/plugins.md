---
title: Лучшие практики создания плагинов Vue
impact: MEDIUM
impactDescription: Неправильная структура плагина или стратегия ключей внедрения приводит к сбоям установки, конфликтам и небезопасным API
type: best-practice
tags: [vue3, plugins, provide-inject, typescript, dependency-injection]
---

# Лучшие практики создания плагинов Vue

**Impact: СРЕДНЕЕ** - плагины Vue должны соответствовать контракту `app.use()`, предоставлять явные
возможности и использовать безопасные с точки зрения конфликтов ключи внедрения. Это обеспечивает
предсказуемость и компонуемость плагинов в больших приложениях.

## Список задач

- Экспортируйте плагины как объект с методом `install()` или как функцию установки
- Используйте экземпляр `app` в `install()` для регистрации компонентов/директив/провайдеров
- Типизируйте API плагинов с помощью `Plugin` (и типов кортежей параметров при необходимости)
- Используйте символьные ключи (предпочтительно `InjectionKey<T>`) для `provide/inject` в плагинах
- Добавьте небольшую типизированную обёртку-композабл для требуемых внедрений, чтобы обеспечить
быстрое обнаружение ошибок

## Структурируйте плагины для `app.use()`

Плагин Vue должен быть либо:

- Объектом с методом `install(app, options?)`
- Функцией с той же сигнатурой

**ПЛОХО:**
```ts
const notAPlugin = {
  doSomething() {}
}

app.use(notAPlugin)
```

**ХОРОШО:**
```ts
import type { App } from 'vue'

interface PluginOptions {
  prefix?: string
  debug?: boolean
}

const myPlugin = {
  install(app: App, options: PluginOptions = {}) {
    const { prefix = 'my', debug = false } = options

    if (debug) {
      console.log('Установка myPlugin с префиксом:', prefix)
    }

    app.provide('myPlugin', { prefix })
  }
}

app.use(myPlugin, { prefix: 'custom', debug: true })
```

**ХОРОШО:**
```ts
import type { App } from 'vue'

function simplePlugin(app: App, options?: { message: string }) {
  app.config.globalProperties.$greet = () => options?.message ?? 'Привет!'
}

app.use(simplePlugin, { message: 'Добро пожаловать!' })
```

## Явно регистрируйте возможности в `install()`

Внутри `install()` настраивайте поведение через API приложения Vue:

- `app.component()` для глобальных компонентов
- `app.directive()` для глобальных директив
- `app.provide()` для инжектируемых сервисов и конфигурации
- `app.config.globalProperties` для дополнительных глобальных помощников (экономно)

**ПЛОХО:**
```ts
const uselessPlugin = {
  install(app, options) {
    const service = createService(options)
  }
}
```

**ХОРОШО:**
```ts
const usefulPlugin = {
  install(app, options) {
    const service = createService(options)
    app.provide(serviceKey, service)
  }
}
```

## Типизируйте контракты плагинов

Используйте тип `Plugin` от Vue, чтобы сигнатуры установки и параметры были типобезопасны.

```ts
import type { App, Plugin } from 'vue'

interface MyOptions {
  apiKey: string
}

const myPlugin: Plugin<[MyOptions]> = {
  install(app: App, options: MyOptions) {
    app.provide(apiKeyKey, options.apiKey)
  }
}
```

## Используйте символьные ключи внедрения в плагинах

Строковые ключи могут конфликтовать (`'http'`, `'config'`, `'i18n'`). Используйте символьные ключи
с `InjectionKey<T>`, чтобы внедрения были уникальными и типизированными.

**ПЛОХО:**
```ts
export default {
  install(app) {
    app.provide('http', axios)
    app.provide('config', appConfig)
  }
}
```

**ХОРОШО:**
```ts
import type { InjectionKey } from 'vue'
import type { AxiosInstance } from 'axios'

interface AppConfig {
  apiUrl: string
  timeout: number
}

export const httpKey: InjectionKey<AxiosInstance> = Symbol('http')
export const configKey: InjectionKey<AppConfig> = Symbol('appConfig')

export default {
  install(app) {
    app.provide(httpKey, axios)
    app.provide(configKey, { apiUrl: '/api', timeout: 5000 })
  }
}
```

## Предоставляйте помощники для обязательных внедрений

Оборачивайте обязательные внедрения в композаблы, которые выбрасывают понятные ошибки настройки.

```ts
import { inject } from 'vue'
import { authKey, type AuthService } from '@/injection-keys'

export function useAuth(): AuthService {
  const auth = inject(authKey)
  if (!auth) {
    throw new Error('Плагин Auth не установлен. Возможно, вы забыли app.use(authPlugin)?')
  }
  return auth
}
```
