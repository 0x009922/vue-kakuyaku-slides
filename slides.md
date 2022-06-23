---
theme: seriph
class: text-center
highlighter: shiki
lineNumbers: false
title: Vue Async Tasks
---

# Vue Async Tasks

---
layout: section
---

# Что нужно делать с промисами?

---

# Общее для асинхронных операций

<br>

Действия:

- **Async data** - загрузить что-то и получить результат промиса
- **Async side-effect** - операция без получения результата
- Abort (иногда) - отменить операцию во время действия

Со стороны Vue

- Иметь реактивное состояние некой операции
  - pending
  - rejected + ошибка
  - fulfilled + результат
  - aborted

---

# Nice to have

- Автоматическая параметризованная загрузка
  - keep-alive данных при переключении между параметрами
    - "грязная" пометка всех данных или только частично
  - кэширование всех или *каких-то* данных напр. в `localStorage`
  - чтобы это можно было тестить, без синглтонов
  - time-to-live
  - Автоматическая перезагрузка когда:
    - network change
    - window focus
  - Prefetch by key
- Error Retry для любых операций
- Pending Delay для любых операций
  
---
layout: section
---

# `vue-promised`

---

# Загрузка данных и состояние промиса

```ts
const {
  data,
  error,
  isDelayElapsed,
  isPending,
  isRejected,
  isResolved,
} = usePromise(axios.get('/users'))
```

---

# Side-effect callback

```ts
const myAction = ref<null | Promise<void>>(null)

function doAction() {
  myAction.value = axios.post('/hey')
}

const PENDING_DELAY = 500
const { data } = usePromise(myAction, PENDING_DELAY)
```


---
clicks: 5
---

# Параметризованная загрузка, keep-alive

```ts {all|1|3-4|6-17|all}
const storage = reactive(new Map<number, unknown>())

const userId = ref(42)
const loadedUser = computed(() => storage.get(userId.value))

const fetchPromise = ref<null | Promise<void>>(null)
const { isPending } = usePromise(fetchPromise)

watch(
  userId,
  (id) => {
    fetchPromise.value = fetch(`/users/${id}`).then(async (x) => {
      storage.set(id, await x.json())
    })
  },
  { immediate: true },
)

return { loadedUser, isPending }
```

---

# `<Promised />`

```vue
<template>
  <Promised :promise="myAction">
    <template #default="data">{{ data }}</template>
    <template #pending>...</template>
  </Promised>

  <Promised :promise="myAction">
    <template #combined="{ data, error, isPending }">
      Данные: {{ data }} <br>
      Ошибка: {{ error }} <br>
      Загружается? {{ isPending }}
    </template>
  </Promised>
</template>
```

---

# Pros & Cons

<br>

Pros:

- Простая маленькая либа
- Pending Delay по умолчанию
- `<Promised />`

Cons:

- Проблемы с типами:
  - `data` - `Ref<null | undefined | T>`. null-проблема
  - `error` - `Ref<null | undefined | Error>`. null-проблема + `throw` можно делать с чем угодно
- Нет механизма прерывания операции

---
layout: section
---

# `useAsyncState()`

by `@vueuse/core`

---

# Просто загрузка

```ts
const { state, error, isLoading, isReady, execute } = useAsyncState(axios.get('/users'), null, {
  immediate: false,
  delay: 600,
  resetOnExecute: true,
  onError(e) {
    console.error(e)
  },
})

execute(500)

```

---
clicks: 1
---

# Параметризованная загрузка

<br>

Пример 1:

```ts
const params = reactive({ a: 0, b: 'foo' })

const { execute, state } = useAsyncState(async () => axios.get(`/users/${params.a}/${params.b}`), null)

watch(params, () => execute())
```

Пример 2:

```ts {all|7-8}
const { execute } = useAsyncState(async (a: number, b: string) => fetch(`/users/${a}/${b}`), null)

const params = reactive({ a: 0, b: 'foo' })

watch(params, ({ a, b }) => execute(300, a, b))

// OOPS! No type errors!
execute(500, 'foo', false)
```

---

# Callback

```ts
const { isLoading, error, execute } = useAsyncState(async (body: unknown) => {
  await axios.post('/users/new', body)
  return null
}, null)

function createUser(user: { name: string }) {
  execute(0, user)
}
```

Путает:

- useAsync**State**
- Два раза давать `null` или что-то другое

---

# Pros & Cons

<br>

Pros:

- отложенный запуск
- delay
- `state` - `Ref<T>`. Можно положить свой тип, и самому обработать случай пустого значения нормально.

Cons:

- `execute()` передаёт аргументы в функцию, но типов нет
- Некрасиво для side-effect'ов

---
layout: section
---

# `swrv`

---

# Примеры

```ts
const { data, error } = useSWRV('/api/user', fetcher)

return {
  data,
  error,
}
```

```ts
const endpoint = ref('/api/user/Geralt')
const { data, error, mutate } = useSWRV(endpoint.value, fetch)

return {
  endpoint,
  data,
  error,
}
```

---

# Prefetch


```ts
import { mutate } from 'swrv'

function prefetch() {
  mutate(
    '/api/data',
    fetch('/api/data').then((res) => res.json())
  )
  // the second parameter is a Promise
  // SWRV will use the result when it resolves
}
```

---

# Решения, которые мы заслужили

<div class="grid grid-cols-5 gap-4">

<div class="col-span-2">

```ts
const STATES = {
  VALIDATING: 'VALIDATING',
  PENDING: 'PENDING',
  SUCCESS: 'SUCCESS',
  ERROR: 'ERROR',
  STALE_IF_ERROR: 'STALE_IF_ERROR',
}
```

</div>

<div class="col-span-3">

```ts
export default function (data, error, isValidating) {
  const state = ref('idle')
  watchEffect(() => {
    if (data.value && isValidating.value) {
      state.value = STATES.VALIDATING
    } else if (data.value && error.value) {
      state.value = STATES.STALE_IF_ERROR
    } else if (data.value === undefined && !error.value) {
      state.value = STATES.PENDING
    } else if (data.value && !error.value) {
      state.value = STATES.SUCCESS
    } else if (data.value === undefined && error) {
      state.value = STATES.ERROR
    }
  })

  return {
    state,
    STATES,
  }
}
```

</div>
</div>

---

# продолжение...

```vue
<template>
  <div>
    <div v-if="[STATES.ERROR, STATES.STALE_IF_ERROR].includes(state)">
      {{ error }}
    </div>
    <div v-if="[STATES.PENDING].includes(state)">Loading...</div>
    <div v-if="[STATES.VALIDATING].includes(state)">
      <!-- serve stale content without "loading" -->
    </div>
    <div
      v-if="
        [STATES.SUCCESS, STATES.VALIDATING, STATES.STALE_IF_ERROR].includes(
          state
        )
      "
    >
      {{ data }}
    </div>
  </div>
</template>
```

---

# Pros & Cons

<br>

Pros

- Из коробки простая загрузка по ключам
- Cache
- Error Retry
- Requests Deduplication
- Prefetch

Cons

- Глобальное состояние
- Проблемы с типами
- Не использовать с колбэками
- Местами "прикольные" решения

---
layout: section
---

# `vswr`

Это не то же самое, что было только что!

--- 


# Pros & Cons

Мне лень давать примеры, так что...

Pros:

- Всё, что есть в `swrv`
- Можно создавать раздельные SWR инстансы
- Можно свой кэш добавить

Cons:

- Не юзабельно для сайд эффектов
- Опять null-проблема

---
layout: section
---

# И что ты предлагаешь?

---

# `Task<T>`

Базовый кирпичик для всего остального

Инкапсулирует

- асинхронную
- непараметризованную
- повторяемую
- возможно прерываемую

операцию.

---

# Инициализация таски

```ts
const task = useTask(async (onAbort) => {
  // do async stuff...
  await delay(40)

  // handle abort
  onAbort(() => {
    // ...
  })

  // return something (or nothing)
  return 20
})
```

---

# Состояние таски

```ts
watch(
  () => task.state,
  (state) => {
    if (state.kind === 'ok') {
      console.log(state.result)
    } else if (state.kind === 'err') {
      console.error(state.error)
    } else if (state.kind === 'pending') {
      console.log('pending...')
    }

    // also uninint & aborted
  },
)
```

---

# Запуск и прерывание

```ts
// just run
task.run()

// abort pending & run new task
task.run()

// run and wait for exactly this run
// result is ok, err or aborted
const result = await task.run()

// just abort
// auto call on scope dispose
task.abort()
```

---

# Утилитки вокруг таски

- `useLastTaskResult(task)`
- `useDelayedPending(task, delay: MaybeRef<number>): Ref<boolean>`
- `useDelayedPendingTask<T>(task: Task<T>, delay: MaybeRef<number>): Task<T>`
- `useStaleIfErrorState(task)`
- `useErrorRetry()`

---

# Условная загрузка

---

# Параметризованная загрузка

---

# 

---

# Out of Vue

Можно использовать `BareTask<T>`, который почти всё то же, но не хранит состояние:

```ts
const task = new BareTask(async () => 42)
const result = await task.run()

expect(result).toEqual({ kind: 'ok', result: 42 })
```

```ts
const ERR = new Error('got you')
const task = new BareTask(async () => {
  throw ERR
})
const result = await task.run()

expect(result).toEqual({ kind: 'err', error: ERR })
```

---

# Канва презентации

A -> Z

A: Что нужно от промисов

- Просто загрузить данные откуда-то
- Автоматически ретраить при ошибках
- Загружать данные параметризованно, или по условию, автоматически
- Делать асинхронные сайд эффекты, навешивать на них удобства
- Отменять операции
- Состояние промисов (+ pending delay)
- Дедупликация одинаковых запросов в промежуток времени
- **Type-strong**
- Можно мокать

Продвинутая загрузка:

- Stale-While-Revalidate
- keep-alive данных с разными ключами
- `localStorage` кэширование
- Time-To-Live ревалидация
- Ревалидация при фокусе / появлении сети
- Prefetch

B: Какие есть сейчас решения, в чём их минусы

Простое:

- `vue-promised` - плохие типы, особый путь с `Ref<Promise>`
- `@vueuse/core` -> `useAsyncState`. Это только про загрузку. Плохие типы на аргументы внутренней функции
- ...

SWR:

- `swrv` - всё плохо
- `vswr` - всё хорошо, хотя к типам можно придраться, может ещё к чему

C: Как новая либа решает различные задачи

- `Task<T>` - база. Объяснить `TaskState<T>`, почему type-strong
- `useTask()`, `.run()`, `.abort()`
- Приятные use-as-you-want pluggable утилитки вокруг таски - error retry, last result, whenever ok/err, delayed pending
- Особая утилитка - `useStaleIfErrorState`. Объяснить `Maybe<T>`, почему type-strong. Такой простой беспамятный SWR.
- Условная и параметризованная загрузка - `useScope`
- Сайд эффекты - разные примеры, с параметрами и без, `useDanglingScope`
  - Простой вызов чего-то без параметров
  - Вызов с параметрами, берущимися из состояния
  - Stateless запуск с `useDanglingScope`
  - Навешиваются все те же утилитки
- Stale-While-Revalidate - TODO. Со скоупами можно сделать, надо только продумать дизайн, чтобы и кэш был, и ttl, и prefetch, и прочее
- deduplication, revalidate-on-focus/network - TODO, сделать довольно просто
- Прочее TODO (?):
  - run dedup
  - `useRerun`
  - all-in-one `usePromise(prom: Promise<T>)`, комбинация `vue-promised` + `@vueuse/core` с нормальными типами. Только для одноразового промиса.
  - all-in-one `useFullyPackedTask()`, сочетающая в себе `useTask` + все утилитки 
- Bonus - `BareTask<T>`

D: Просьба дать фидбек, хотят ли использовать, хотят ли помочь

- Насколько такое видится полезным, насколько хочется использовать. Btw лично я буду в любом случае, избавляет от необходимости ставить разные любы и ломать голову, как их дружить.
- Как назвать либу?
  - `@vue-async-tasks/*`
  - `@vue-tasks/*`
  - `@vue-async/*`
  - `@vue-use-task/*`
  - `@vue-use-tasks/*`
  - `@vue-any-task/*`
  - `@vue-any-async/*`
  - `@vue-any-promise/*`
  - `@vue-futures/*`
  - `@vue-spawn-async/*`
  - `@vue-async-spawn/*`
  - `@sora-vue-promises/*`
  - `@sora-vue-async/*`
  - `@yava/*` (Yet Another Vue Async)
- Если есть мысли, пожелания - contribution welcome

---
