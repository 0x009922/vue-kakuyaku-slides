---
theme: seriph
class: text-center
highlighter: shiki
lineNumbers: false
title: Vue Async Tasks
---

# vue-kakuyoku

kakuyoku - обещание

---


# `useTask()`

```ts
const task = useTask(async (onAbort) => {
  // do any async stuff
  await delay(750);

  // optional
  onAbort(() => {
    // do cleanup
  });

  return 42;
});

```

---

# Task usage


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

# What is a Task?

<br>

Task is an abstraction around

- async function (i.e. returns a `Promise<T>`)
- unparametrized (i.e. fires without any arguments)
- (optional) abortable & repeatable

---

# `Task<T>`

```ts
interface Task<T> {
  state: TaskState<T>;
  run: () => Promise<BareTaskRunResult<T>>;
  abort: () => void;
}

/** Simplified */
type BareTaskRunResult<T> =
  | { kind: "ok"; data: T }
  | { kind: "err"; error: unknown }
  | { kind: "aborted" };

/** Simplified */
type TaskState<T> =
  | { kind: "uninit" }
  | { kind: "pending" }
  | BareTaskRunResult<T>;
```

- Good TypeScript support for "discriminated" types

---

# Example with a raw Task

```vue
<script setup>
const task = useTask(async () => fetch("/users"));

task.run();

function rerun() {
  task.run();
}
</script>

<template>
  <button @click="rerun">Rerun</button>

  <div v-if="task.state.kind === 'ok'">
    {{ task.state.data }}
  </div>

  <div v-else-if="task.state.kind === 'pending'">Loading...</div>
</template>
```

---

# not that impressive...

---

# utilities around `Task<T>`

---

# Retry-on-error with `useErrorRetry()`

It is needed so much often...

```ts
const { retries, reset } = useErrorRetry(task, {
  count: 7, // default: 5
  interval: 3_000, // default: 5s
});
```

---

# Unwrap task state with `useStaleIfErrorState`

Always access last task result data

```ts
interface TaskStaleIfErrorState<T> {
  data: Maybe<T>;
  error: Maybe<unknown>;
  pending: boolean;
  fresh: boolean;
}

type Maybe<T> = null | { some: T };
```

- Access to last task result and error even if it is pending now
- Correct types with `Maybe<T>`

---

# Stale state usage example

```vue
<script setup>
const task = useTask(async () => fetch("/users"));
const state = useStaleIfErrorState(task);
task.run();

function reload() {
  task.run();
}
</script>

<template>
  <Users
    v-if="state.data"
    :data="state.data.some"
    :is-fresh="state.fresh"
  />
  <Error v-if="state.error" :err="state.error.some" />
  <Spinner v-if="state.pending" />

  <button :disabled="state.pending" @click="reload">Reload</button>
</template>
```

---

# Run side effect whenever task errors/succeeds

```ts
wheneverTaskErrors(task, (error) => {
  console.error(error);
});

wheneverTaskSucceeds(task, (users) => {
  store.$patch({ users });
});
```

---

# `wheneverTaskXXX`

These shorthands are simple

```ts
wheneverTaskErrors(task, (error) => {
  console.error(error);
});

// same as

watch(
  () => task.state,
  (state) => {
    if (state.kind === "err") {
      fn(state.error);
    }
  },
  {
    // important default options
    immediate: true,
    flush: "sync",
    ...options,
  }
);
```

---

# Debounce pending state

## Just a single Ref

```ts
const pending = useDelayedPending(
  task,
  // MaybeRef<number>
  // or 200 by default
  500
);
```

## Make a task wrapper

```ts
const delayedPendingTask = useDelayedPendingTask(task);

const state = useStaleIfErrorState(delayedPendingTask);
```

---

# Remember only last task result

```ts
// null, ok or error
const lastResult = useLastTaskResult(task);
```

---

# Flexible task setup with scopes

---
clicks: 3
---

# Conditional task **setup**

```ts {all|1|3-12|14-15}
const isVisible = ref(false);

const scope = useScope(isVisible, () => {
  const task = useTask(() => fetch("/stats"));
  const staleState = useStaleIfErrorState(task);
  useErrorRetry(task);
  task.run()

  const { run: retry } = task;

  return { retry, state: staleState };
});

const isPending = computed(() => scope.value?.setup.state.pending ?? false);
const data = computed(() => scope.value?.setup.state.data?.some ?? null);
```

---

# Usage of a conditional scope

```vue
<template>
  <div>
    Pending? {{ isPending }} <br />
    Data? {{ data }}

    <button v-if="scope" @click="scope.setup.retry()">Retry</button>
  </div>
</template>
```

---

# Keyed setup

```ts
const userId = ref(15);

const scope = useScope(userId, (staticUserId) => {
  const task = useTask(() => fetch(`/users/${staticUserId}`));

  // setup anything...

  return task;
});

watchEffect(() => {
  console.log("scope is always non-null, even in TS:", scope.setup.state);
});
```

---

# Keyed + Conditional

```ts
const isVisible = ref(false);
const userId = ref(42);

const scope = useScope(
  computed(() => isVisible.value && userId.value),
  (staticUserId) => {
    // ...
  }
);

// scope.value is not always exist, according to TS
// Type Safe!
```

---

# Bonus: nested scopes setup

```ts
const key = ref("foo");
const enableErrorRetry = ref(false);

const scope = useScope(key, (key) => {
  const task = useTask(() => fetch(`/baz/${key}`));

  useScope(enableErrorRetry, () => {
    useErrorRetry(task);
  });

  return useStaleIfErrorState(task);
});
```

---

# Dynamic callback dispatch with `useDanglingScope`

---

# submit a form without any scopes

```ts
// edit reactive bindings with form elements
const user = reactive({
  name: "Joe",
  age: 42,
});

// our main task
const submitTask = useTask(() => axios.post("/users", user));

const lastResult = useLastTaskResult(submitTask);
// can be shown in the template
const lastError = computed(() =>
  lastResult.value.kind === "err" ? lastResult.value : null
);

// task execution
async function submit() {
  const result = await submitTask.run();
  if (result.kind === "ok") {
    store.updateUsers();
  }
}
```

---

# stateless form submit

```ts
const lastSubmitTask = useDanglingScope<Task<void>>();

function submit(data: { name: string; age: number }) {
  lastSubmitTask.setup((dispose) => {
    const task = useTask(() => axios.post(`/users`, data));
    task.run();

    // just retry...
    useErrorRetry(task, { count: Infinity, interval: 1000 });

    // stop retries after 15s
    useTimeoutFn(() => dispose(), 15_000);
  });
}

function forgetLastAction() {
  lastSubmitTask.dispose();
}
```

---

# TODO

- utility: deduplicate tasks runs in a period of time (throttle)
- utility: transform `onAbort` hook into `AbortSignal`
- utility: full-packed `useAsyncData()`
  
  All the best from `usePromise`(`vue-promised`) + `useAsyncState` (`@vueuse/core`)

  May include all nice stuff like error retry, stale state etc

  - \+ `useFetch()` Fetch API wrapper around `useAsyncData`
  

- utilities: revalidate on focus / network
- utilities to keep state alive, external storage; SWR
- SSR compatibility?

---

# bonus: `BareTask<T>`

- Vue-free core behind `useTask`

```ts
const readFileTask = new BareTask(() => fs.readFile("vulpes.3a"));

let count = 0;
while (count++ < 10) {
  const result = await readFileTask.run();

  if (result.kind === "ok") {
    console.log(result.value);
    break;
  } else if (result.kind === "err") {
    console.error(result.error);
  } else {
    // aborted
    console.log("impossibru...");
  }
}
```

---

# Comparison with other libraries

---

# `vue-promised`

```ts
const { data, error, isPending, isResolved, isRejected } = usePromise(
  fetch("/users")
);

// or

const prom = ref(null);
const promState = usePromise(prom);

function runFetch() {
  prom.value = fetch("/users");
}
```

---

# `vue-promised`

- pending delay out of the box
- `data: Ref<T | null | undefined>` - not really strong
- `error: Ref<Error | null | undefined>` - again not strong + you can `throw` literally anything, not only `Error`
- no utils like scopes, error retry, abortation etc

---

# `@vueuse/core`'s `useAsyncState`

```ts
const { state, isLoading, execute } = useAsyncState(
  (id: string, page: number) => fetch(`/users/${id}/posts?page=${page}`),
  null, // init state
  {
    immediate: false,
    resetOnExecute: true,
  }
);

execute(
  // execution delay
  500,
  // untyped args...
  123,
  "foo",
  false
);
```

--- 

# `@vueuse/core`

- only suitable for state: naming, null returns
- bad `execute` typing makes arguments passing useless
- no scope utils

---

# comparison with `swrv`

- only for state
- bad types
- fetch-oriented
- global singleton with all its outcomes
- a lot of unnecessary non-tree-shakeable utilities
- weird design decisions

---

# comparison with `vswr`

- only for state
- null-data type
- its... fine?

---

# Техническое на этом всё!

---

# Зачем эта презентация?

- Получить фидбек - удобно это выглядит или нет, хочется ли затаскивать на проекты
- Возможно кто-то захочет помочь в разработке и поддержке либы

---

# Название либы

Было много вариантов:

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

---

# Но остановился на `vue-kakuyoku`

<br>

Kakuyoku - обещание (яп.)

Вполне оправдано тем, какой компанией это разрабатывается

---

# Спасибо за внимание!

---

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
