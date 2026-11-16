### 15. Таймеры: не используем `setTimeout` и `setInterval` напрямую

#### Контекст

`setTimeout` и `setInterval` кажутся простым решением для любой задачи со словом «через»: «скрыть уведомление через 3 секунды», «обновлять данные каждые 5 секунд», «подождать, пока отрендерится». В React-проекте у такого кода есть типичные проблемы:

- **Забытая очистка.** Таймер продолжает жить после размонтирования компонента, срабатывает «не вовремя», дублируется в Strict Mode, держит в памяти замыкание.
- **Устаревшие замыкания.** Колбэк запоминает значения на момент создания таймера и работает с неактуальным состоянием.
- **Гадание о времени.** «Подождём 100 мс, и DOM точно появится». На быстром компьютере работает, на медленном нет. Число в таймере маскирует проблему последовательности действий.
- **Неточность.** Таймер срабатывает «не раньше чем через N мс», а не ровно через N. В фоновых вкладках браузеры замедляют таймеры. `setInterval` накапливает дрейф и не ждёт завершения асинхронной работы внутри колбэка, поэтому запросы накладываются друг на друга.
- **Непрозрачное намерение.** `setTimeout(fn, 300)` не говорит, зачем он здесь: это debounce, ожидание анимации или костыль?
- **Сложности с тестами.** Код с таймерами недетерминирован и требует `fake timers`.

#### Решение

**Не используем `setTimeout` и `setInterval` напрямую в компонентах и в коде фич.** Вместо них выбираем инструмент, который прямо выражает намерение (таблица ниже). Если таймер действительно нужен, используем общие хуки (`useTimeout`, `useInterval`, `useDebouncedValue`), которые гарантируют очистку и покрыты тестами.

##### Чем заменять

| Задача | Вместо таймера |
| --- | --- |
| Задержка при вводе (debounce) | `useDebouncedValue` или `debounce` из lodash (ADR 5) |
| Периодическая загрузка данных | RTK Query: `pollingInterval` (ADR 2) |
| Сфокусировать или измерить элемент после его появления | эффект, зависящий от появления, или callback ref (ADR 10, ADR 11) |
| Дождаться конца анимации | `onTransitionEnd`, `onAnimationEnd` |
| Обновление, привязанное к кадрам | `requestAnimationFrame` с `cancelAnimationFrame` |
| Таймаут сетевого запроса | `AbortSignal.timeout(ms)` |
| Повторные попытки с задержкой | механизм повторов (например, `retry` в RTK Query) |
| Автоскрытие уведомления | `useTimeout` |
| Обратный отсчёт, часы | вычислять от метки времени, `useInterval` только для перерисовки |
| Дождаться появления или размера элемента | `IntersectionObserver`, `ResizeObserver` |
| Не блокировать ввод тяжёлым рендером | `useDeferredValue`, `startTransition` |

##### Примеры

```tsx
// ❌ нет очистки: таймер срабатывает и после размонтирования компонента
useEffect(() => {
  setTimeout(() => setIsVisible(false), 3000);
}, []);

// ✅ через общий хук: очистка гарантирована
useTimeout(() => setIsVisible(false), 3000);
```

```tsx
// ❌ устаревшее замыкание: count всегда равен 0, поэтому счётчик останавливается на 1
useEffect(() => {
  const intervalId = setInterval(() => setCount(count + 1), 1000);

  return () => clearInterval(intervalId);
}, []);

// ✅ общий хук работает с актуальным колбэком, а состояние обновляем функцией
useInterval(() => setCount(prev => prev + 1), 1000);
```

```tsx
// ❌ «подождём 100 мс, пока элемент отрисуется»: гадание, которое зависит от скорости устройства
const handleOpen = () => {
  setIsOpen(true);
  setTimeout(() => inputRef.current?.focus(), 100);
};

// ✅ фокус ставим, когда элемент действительно появился
useEffect(() => {
  if (isOpen) inputRef.current?.focus();
}, [isOpen]);
```

```ts
// ❌ запрос может идти дольше 5 секунд: вызовы накладываются, нет отмены,
// опрос продолжается в неактивной вкладке
useEffect(() => {
  const intervalId = setInterval(() => loadStatus(), 5000);

  return () => clearInterval(intervalId);
}, []);

// ✅ RTK Query сам управляет опросом, отменой и состоянием загрузки
const {data} = useGetStatusQuery(undefined, {pollingInterval: 5000, skipPollingIfUnfocused: true});
```

```tsx
// ❌ накапливаем тики: интервал неточен и замедляется в фоновой вкладке
const [secondsLeft, setSecondsLeft] = useState(60);
useInterval(() => setSecondsLeft(prev => prev - 1), 1000);

// ✅ время считаем от метки конца; интервал нужен только для перерисовки
const [now, setNow] = useState(Date.now);
useInterval(() => setNow(Date.now()), 1000);
const secondsLeft = Math.max(0, Math.ceil((endsAt - now) / 1000));
```

```tsx
// ❌ длительность анимации продублирована в CSS и в JS: изменили одно, забыли другое
setTimeout(() => setIsMounted(false), 300);

// ✅ реагируем на реальное завершение анимации
<div className={styles.panel} onTransitionEnd={() => setIsMounted(false)} />
```

```ts
// ✅ таймаут запроса без таймера в нашем коде
const response = await fetch(url, {signal: AbortSignal.timeout(5000)});
```

```tsx
// ✅ debounce поиска и запрос по значению с задержкой
const [search, setSearch] = useState('');
const debouncedSearch = useDebouncedValue(search, 300);
const {data} = useSearchUsersQuery(debouncedSearch, {skip: !debouncedSearch});
```

##### Общие хуки

Таймеры живут в одном месте (`src/hooks`, см. ADR 1), и в каждом из них есть очистка. Тип идентификатора таймера пишем как `ReturnType<typeof setTimeout>`, чтобы не зависеть от окружения (в браузере число, в Node объект).

```ts
// useTimeout.ts
import {useEffect, useEffectEvent} from 'react';

export const useTimeout = (callback: () => void, delay: number | null) => {
  // всегда вызывает актуальный колбэк и не перезапускает таймер при его смене
  const onTimeout = useEffectEvent(callback);

  useEffect(() => {
    if (delay === null) return;

    const timerId = setTimeout(onTimeout, delay);

    return () => clearTimeout(timerId);
  }, [delay]);
};
```

```ts
// useInterval.ts
import {useEffect, useEffectEvent} from 'react';

export const useInterval = (callback: () => void, delay: number | null) => {
  const onTick = useEffectEvent(callback);

  useEffect(() => {
    if (delay === null) return;

    const intervalId = setInterval(onTick, delay);

    return () => clearInterval(intervalId);
  }, [delay]);
};
```

```ts
// useDebouncedValue.ts
import {useEffect, useState} from 'react';

export const useDebouncedValue = <T>(value: T, delay: number) => {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timerId = setTimeout(() => setDebouncedValue(value), delay);

    return () => clearTimeout(timerId);
  }, [value, delay]);

  return debouncedValue;
};
```

`useEffectEvent` доступен в React 19.2 и новее. В более ранних версиях актуальный колбэк хранят в `ref`, обновляя его в эффекте (ADR 10).

Хуки покрываем тестами с `fake timers` (ADR 14):

```ts
// tests/useDebouncedValue.test.ts
import {act, renderHook} from '@testing-library/react';
import {afterEach, beforeEach, expect, it, vi} from 'vitest';
import {useDebouncedValue} from '../useDebouncedValue';

beforeEach(() => vi.useFakeTimers());
afterEach(() => vi.useRealTimers());

it('обновляет значение только после задержки', () => {
  const {result, rerender} = renderHook(({value}) => useDebouncedValue(value, 300), {
    initialProps: {value: 'а'},
  });

  rerender({value: 'ан'});
  expect(result.current).toBe('а');

  act(() => vi.advanceTimersByTime(300));
  expect(result.current).toBe('ан');
});
```

##### Правила

1. **Прямые вызовы `setTimeout` и `setInterval` допустимы только внутри общих хуков и утилит,** которые инкапсулируют таймер.
2. **У любого таймера есть очистка** (`clearTimeout`, `clearInterval`, `cancelAnimationFrame`) при размонтировании или смене зависимостей.
3. **Таймер не используется для синхронизации.** Если код содержит «подождать 100 мс, пока что-то произойдёт», нужно подписаться на само событие или состояние.
4. **Время считаем по меткам, а не по количеству срабатываний.** Интервал нужен только для перерисовки.
5. **Периодическую работу с асинхронными запросами не пишем на `setInterval`:** используем RTK Query или планируем следующий вызов после завершения предыдущего.

##### Когда прямой таймер допустим

- **Внутри общих хуков и утилит** (`useTimeout`, `useInterval`, `useDebouncedValue`, `debounce`), для которых он и нужен.
- **Код вне React** (скрипты, серверная часть). Там по возможности используем `setTimeout` из `node:timers/promises`.
- **Тесты,** где время подменяется через `vi.useFakeTimers`.

#### Обоснование

**1. Очистка и актуальность заложены в абстракцию**

Хук `useTimeout` один раз правильно решает очистку и устаревшие замыкания, и их не нужно помнить в каждом компоненте.

**2. Инструмент называет намерение**

`pollingInterval` читается как «опрос», `useDebouncedValue` как «задержка ввода», `onTransitionEnd` как «после анимации». Голый `setTimeout(fn, 300)` приходится расшифровывать.

**3. События вместо гадания**

Подписка на реальное событие или состояние работает одинаково на любом устройстве. Таймер лишь предполагает, что за заданное время всё успеет произойти.

**4. Точность и поведение браузера**

Таймеры не гарантируют точного срабатывания и замедляются в неактивных вкладках. Расчёт от метки времени и готовые механизмы (RTK Query, `requestAnimationFrame`) это учитывают.

**5. Тестируемость**

Общие хуки с таймерами тестируются один раз. Остальной код использует их и не требует подмены времени.

#### Контроль

```js
// eslint.config.js (фрагмент)
{
  files: ['src/**/*.{ts,tsx}'],
  rules: {
    'no-restricted-globals': [
      'error',
      {name: 'setTimeout', message: 'Используйте useTimeout, useDebouncedValue или другой механизм (ADR 15).'},
      {name: 'setInterval', message: 'Используйте useInterval или RTK Query pollingInterval (ADR 15).'},
    ],
  },
},
{
  // общие хуки и утилиты, где таймеры разрешены
  files: ['src/hooks/**', 'src/lib/**', '**/tests/**'],
  rules: {'no-restricted-globals': 'off'},
},
```

Правило `no-restricted-globals` ловит прямые вызовы. Вызовы через `window.setTimeout` ловит правило `no-restricted-properties`, его при необходимости добавляем по аналогии.

#### Последствия

- Нет забытых таймеров, устаревших замыканий и накладывающихся запросов.
- Намерение кода читается по имени инструмента.
- Код не зависит от гадания о времени и стабильнее на медленных устройствах.
- Все таймеры собраны в нескольких проверенных хуках, их проще тестировать и менять.
- Для разработчика появляется шаг «подобрать подходящий инструмент» вместо привычного `setTimeout`. Таблица выше помогает его сделать.
- Общие хуки нужно написать и поддерживать, а для `useEffectEvent` требуется актуальная версия React.
