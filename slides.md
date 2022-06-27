---
theme: shibainu
title: vue-kakuyaku
highlighter: shiki
lineNumbers: false
fonts:
  mono: 'JetBrains Mono'
  sans: 'Rubik'

layout: cover
---

# vue-kakuyaku

Решаем задачи с промисами

---
layout: default-2
---

# 確約 (kakuyaku) - обещание (яп.)

Перевод названия

`vue-kakuyaku` - похоже на `vue-promised`. Но функционалом&hellip; не слишком.

---
layout: default-3
---

# С чего начинается либа - `useTask()`

```ts
const task = useTask(async (onAbort) => {
  // делаем что-нибудь асинхронное
  await delay(750);

  // опционально настраиваем отмену операции
  onAbort(() => {
    // ...
  });

  return 42;
});
```

---
layout: default-6
---


# Использование `Task`

```ts
// запуск;
// если уже запущено - отмена и запуск снова
task.run()

// запустить и получить результат именно этого запуска
const result = await task.run()

// отмена текущей
// * срабатывает также onScopeDispose
task.abort()
```

---
layout: default-4
---

# `Task<T>`

Что такое ~~такса~~ таска?


`Task<T>` - это абстракция вокруг:

- асинхронной функции (т.е. возвращает `Promise<T>`)
- непараметризованной (т.е. без аргументов)
- (опционально) прерываемой & повторяемой

---
layout: default-3
---

# Декларация `Task<T>`

```ts
interface Task<T> {
  state: TaskState<T>;
  run: () => Promise<BareTaskRunResult<T>>;
  abort: () => void;
}

/** упрощённо */
type BareTaskRunResult<T> =
  | { kind: "ok"; data: T }
  | { kind: "err"; error: unknown }
  | { kind: "aborted" };

/** упрощённо */
type TaskState<T> =
  | { kind: "uninit" }
  | { kind: "pending" }
  | BareTaskRunResult<T>;
```

- Благодаря дискриминанту `kind` хорошо работает TypeScript

---

# Пример с "сырой" таской

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
layout: quote
---

# не особо впечатляет...

---
layout: section-3
---

# Утилитки вокруг таски

---
layout: default-4
---

# Error-retry

Это бывает нужно так часто... но при этом не всегда

```ts
const { retries, reset } = useErrorRetry(task, {
  count: 7, // default: 5
  interval: 3_000, // default: 5s
});
```

---
layout: default-2
---

# Stale-if-error state

Можно всегда иметь последний результат и ошибку таски

```ts
interface TaskStaleIfErrorState<T> {
  data: Maybe<T>;
  error: Maybe<unknown>;
  pending: boolean;
  fresh: boolean;
}

type Maybe<T> = null | { some: T };
```

- Даже если таска грузится *снова*, её прошлый результат доступен
- Нет конфузов с наличием data/error даже с точки зрения TS - спасибо `Maybe<T>`

---
layout: default
---

# `useStaleIfErrorState()`

Утилита в действии

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

# "Whenever task..."

Сайд-эффекты при ошибке/успехе

```ts
wheneverTaskErrors(task, (error) => {
  console.error(error);
});

wheneverTaskSucceeds(task, (users) => {
  store.$patch({ users });
});
```

---
layout: default-3
---

# А изнутри...

...утилитки сами по себе очень просты

```ts
wheneverTaskErrors(task, (error) => {
  console.error(error);
});

// то же что и

watch(
  () => task.state,
  (state) => {
    if (state.kind === "err") {
      fn(state.error);
    }
  },
  {
    // важные опции по умолчанию
    immediate: true,
    flush: "sync",
    ...options,
  }
);
```

---
layout: default-5
---

# Отложенный "pending"

- `useDelayedPending()` - в виде Ref самого по себе

  ```ts
  const pending = useDelayedPending(
    task,
    // MaybeRef<number>
    // по умолчанию, например, 200мс
    500
  );
  ```

- `useDelayedPendingTask()` - полная обёртка, в которой "pending" отложенное

  ```ts
  const delayedPendingTask = useDelayedPendingTask(task);

  const state = useStaleIfErrorState(delayedPendingTask);
  ```

---
layout: default-6
---

# Если нужен любой последний результат таски

```ts
// null, ok or error
const lastResult = useLastTaskResult(task);
```

---
layout: section
---

# Гибкий **setup** тасок с `useScope()`

---
clicks: 3
layout: default-3
---

# Опциональный **setup**

Настройка таски по реактивному условию

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
layout: default-4
---

# Использование

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
layout: default-6
---

# **Setup** по ключу

Когда настройка таски зависит от ID'шника, например

```ts
const userId = ref(15);

const scope = useScope(userId, (staticUserId) => {
  const task = useTask(() => fetch(`/users/${staticUserId}`));

  // любая логика...

  return task;
});

watchEffect(() => {
  console.log("scope is always non-null, even in TS:", scope.setup.state);
});
```

---
layout: default-2
---

# Ключ + условие

Можно совмещать и условие, и ключ - при этом корректный TypeScript

```ts
const isVisible = ref(false);
const userId = ref(42);

const scope = useScope(
  computed(() => isVisible.value && userId.value),
  (staticUserId) => {
    // ...
  }
);

// TS говорит, что `scope.value` не всегда существует
// Type Safe!
```

---

# Бонус: вложенная настройка скоупов

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
layout: section-2
---

# Cайд-эффекты

Асинхронные действия и "подвешенные скоупы"

---
layout: default
---

# Отправка формы

```ts
// редактируем данные формы тут
const user = reactive({
  name: "Joe",
  age: 42,
});

// здесь наша таска
const submitTask = useTask(() => axios.post("/users", user));

const lastResult = useLastTaskResult(submitTask);
// ошибку отправки можем показать в форме
const lastError = computed(() =>
  lastResult.value.kind === "err" ? lastResult.value : null
);

// запускаем таску
async function submit() {
  const result = await submitTask.run();
  if (result.kind === "ok") {
    // делаем что-нибудь в случае успеха, если нам это надо
    store.updateUsers();
  }
}
```

---

# Теперь без промежуточных состояний

```ts
const lastSubmitTask = useDanglingScope<Task<void>>();

function submit(data: { name: string; age: number }) {
  lastSubmitTask.setup((dispose) => {
    const task = useTask(() => axios.post(`/users`, data));
    task.run();

    // просто повторяем до бесконечности...
    useErrorRetry(task, { count: Infinity, interval: 1000 });

    // но вообще-то нет, после 15сек перестанем
    // или когда скоуп уничтожится
    useTimeoutFn(() => dispose(), 15_000);
  });
}

function forgetLastAction() {
  lastSubmitTask.dispose();
}
```

---

# TODO

- утилитка: дедупликация запусков за промежуток времени (троттлинг)
- утилитка: сделать из хука `onAbort()` сразу `AbortSignal`
- утилитка: swiss-knife `useAsyncData()`:
  
  - Сочетание всегда лучшего из `usePromise`(`vue-promised`) + `useAsyncState` (`@vueuse/core`)

  - Может включать в себя и весь "сахар" вроде error retry, stale state, deduplication etc

  - \+ `useFetch()` Fetch API обёртка вокруг `useAsyncData()`
  
- утилитка: revalidate on focus / network
- SWR: утилитки для сохранения состояния около-глобально, кэширование, TTL
- поддержка SSR?
- Протестить либу в боевых условиях! Станет понятно, что ещё нужно, что лишнее, что надо переработать.

<style>
ul {
  font-size: 0.9rem;
}

li p {
  margin: 8px;
}
</style>

---
layout: default-4
---

# Бонус: `BareTask<T>`

Vue-free ядро под капотом у `useTask()`. Можно использовать с любыми промисами.

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
layout: section-3
---

# Другие либы

---
layout: default-6
---

# `vue-promised`

```ts
const { data, error, isPending, isResolved, isRejected } = usePromise(
  fetch("/users")
);

// или

const prom = ref(null);
const promState = usePromise(prom);

function runFetch() {
  prom.value = fetch("/users");
}
```

---
layout: default-6
---

# `vue-promised`

- Отложенный pending из коробки
- `data: Ref<T | null | undefined>` - не type-safe
- `error: Ref<Error | null | undefined>` - снова не type-safe + можно выкинуть (`throw`) буквально что угодно, а не только `Error`
- Нет полезных утилиток со скоупами, error-retry, отменами etc

---
layout: default-4
---

# `useAsyncState()` из `@vueuse/core`

```ts
const { state, isLoading, execute } = useAsyncState(
  (id: string, page: number) => fetch(`/users/${id}/posts?page=${page}`),
  null, // начальное состояние
  {
    immediate: false,
    resetOnExecute: true,
  }
);

execute(
  // отложенный запуск
  500,
  // нетипизированные аргументы...
  123,
  "foo",
  false
);
```

---
layout: default-4
---

# `@vueuse/core`

- Изабельно только для **загрузки состояний**: useAsync**State**, бубны в случае безразличия к результату
- Отсутствие типизации аргументов в `execute()` делает это бесполезным в принципе
- Из коробки нет утилиток для повторения, скоупов etc

---
layout: default-2
---

# `swrv`

- только для загрузки + fetch only!
- проблемы с типами
- глобальный синглтоно, со всеми вытекающими
- не такие уж важные фичи, которые нельзя tree-shake'нуть
- другие странные решения

---
layout: default-5
---

# `vswr`

Как `swrv`, но не такое странное

- только для загрузки, но не только fetch
- небольшие проблемы с типами
- в остальном... вроде ок?

---
layout: quote
---

# Техническое на этом всё!

---
layout: right
---

# Зачем эта презентация?

<br>

Получить фидбек - удобно это выглядит или нет, хочется ли затаскивать на проекты.

Возможно кто-то захочет помочь в разработке и поддержке либы.

---
layout: default-2
---

# Другие варианты названия либы?

Разные примеры

Было много вариантов:

> vue-async-tasks, vue-tasks, vue-async, vue-use-task, vue-use-tasks, vue-any-task, vue-any-async, vue-any-promise, vue-futures, vue-spawn-async, vue-async-spawn, sora-vue-promises, sora-vue-async, yava (Yet Another Vue Async), vulpes (vue loves promises)...

---
layout: section
---

# Спасибо за внимание!
