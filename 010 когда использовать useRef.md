### 10. Когда использовать `useRef`, а когда нет

#### Контекст

`useRef` часто воспринимают как «обычную переменную, которая не сбрасывается между рендерами». Из-за этого его используют не по назначению: хранят в нём то, что должно отображаться на экране, читают и пишут `ref.current` прямо в теле компонента, обходят с его помощью зависимости эффектов или создают через него тяжёлые объекты. В результате интерфейс не обновляется, значения рассинхронизируются, а React Compiler не может оптимизировать такие компоненты.

Главное отличие `ref` от состояния: **изменение `ref.current` не вызывает перерисовку**, а значение при этом сохраняется между рендерами.

#### Решение

`useRef` используем в двух случаях:

1. для доступа к **DOM-элементу**;
2. для хранения значения, которое должно пережить рендер, но **не влияет на то, что отображается на экране**: идентификатор таймера, экземпляр стороннего объекта, служебный флаг для императивной логики.

Во всех остальных случаях `useRef` не нужен. Выбираем инструмент по задаче:

| Что нужно | Инструмент |
| --- | --- |
| Показать значение в интерфейсе | `useState` или `useReducer` |
| Значение, которое меняется без перерисовки (id таймера, экземпляр библиотеки, служебный флаг) | `useRef` |
| Получить DOM-элемент для обработчика или эффекта (фокус, прокрутка, измерение) | `useRef` |
| Реагировать на появление или смену DOM-элемента (измерить размер, подключить observer) | callback ref + `useState` |
| Значение, которое можно вычислить из props или состояния | вычисляем прямо в рендере, без `ref` и без эффекта |
| Создать объект один раз при монтировании | `useState(() => ...)` |
| Данные, нужные нескольким компонентам | стор (см. ADR 2) |

##### Правила

1. **`ref.current` читаем и пишем только в обработчиках событий и эффектах**, но не во время рендера. Рендер должен быть чистой функцией: React может вызвать его повторно или отбросить результат, а React Compiler рассчитывает на чистоту.
2. **То, что показывается в интерфейсе, хранится в состоянии**, а не в `ref`.
3. **Ресурсы из `ref` освобождаем в cleanup эффекта:** таймеры, observers, соединения, экземпляры сторонних библиотек.
4. **`ref.current` может быть `null`** (до монтирования и после размонтирования), поэтому перед использованием проверяем его или пишем `?.`.
5. **`ref.current` не указываем в массиве зависимостей** `useEffect` и `useCallback`: он не реактивен, и его изменение не запустит эффект.
6. **Для DOM используем `ref`, а не `document.querySelector` и `getElementById`.**

##### Когда `useRef` подходит

```tsx
// ✅ доступ к DOM-элементу из обработчика
interface SearchBoxProps {
  value: string;
  onChange: (value: string) => void;
}

export const SearchBox = ({value, onChange}: SearchBoxProps) => {
  const inputRef = useRef<HTMLInputElement>(null);

  const handleClear = () => {
    onChange('');
    inputRef.current?.focus();
  };

  return (
    <div>
      <input ref={inputRef} value={value} onChange={event => onChange(event.target.value)} />
      <button type="button" onClick={handleClear}>
        Очистить
      </button>
    </div>
  );
};
```

```tsx
// ✅ id таймера не отображается, поэтому лежит в ref; секунды отображаются, поэтому в состоянии
export const Stopwatch = () => {
  const [seconds, setSeconds] = useState(0);
  const timerIdRef = useRef<number | null>(null);

  const start = () => {
    if (timerIdRef.current !== null) return;
    timerIdRef.current = window.setInterval(() => setSeconds(prev => prev + 1), 1000);
  };

  const stop = () => {
    if (timerIdRef.current === null) return;
    window.clearInterval(timerIdRef.current);
    timerIdRef.current = null;
  };

  // при размонтировании таймер нужно остановить
  useEffect(
    () => () => {
      if (timerIdRef.current !== null) window.clearInterval(timerIdRef.current);
    },
    []
  );

  return (
    <div>
      <span>{seconds} с</span>
      <button type="button" onClick={start}>
        Старт
      </button>
      <button type="button" onClick={stop}>
        Стоп
      </button>
    </div>
  );
};
```

```tsx
// ✅ экземпляр стороннего объекта (график, карта) и DOM-контейнер для него
export const ChartView = ({data}: ChartViewProps) => {
  const containerRef = useRef<HTMLDivElement>(null);
  const chartRef = useRef<ChartLibrary | null>(null);

  useEffect(() => {
    if (!containerRef.current) return;

    chartRef.current = new ChartLibrary(containerRef.current);

    return () => {
      chartRef.current?.destroy();
      chartRef.current = null;
    };
  }, []);

  useEffect(() => {
    chartRef.current?.setData(data);
  }, [data]);

  return <div ref={containerRef} />;
};
```

```tsx
// ✅ нужно реагировать на появление элемента (например, замерить размер) — callback ref и состояние
const [element, setElement] = useState<HTMLDivElement | null>(null);
const size = useElementSize(element); // хук подписывается на размер, когда element появился

return <div ref={setElement}>...</div>;
```

```tsx
// ✅ React 19+: ref передаётся как обычный проп, forwardRef не нужен
interface TextFieldProps extends ComponentPropsWithoutRef<'input'> {
  ref?: Ref<HTMLInputElement>;
}

export const TextField = ({ref, ...rest}: TextFieldProps) => <input ref={ref} {...rest} />;
```

##### Когда `useRef` НЕ подходит

```tsx
// ❌ значение отображается, но изменение ref не вызывает перерисовку: число на экране не обновится
const countRef = useRef(0);
const handleClick = () => {
  countRef.current += 1;
};

return <button onClick={handleClick}>Нажато: {countRef.current}</button>;

// ✅ то, что видно на экране, хранится в состоянии
const [count, setCount] = useState(0);

return <button onClick={() => setCount(prev => prev + 1)}>Нажато: {count}</button>;
```

```tsx
// ❌ чтение ref во время рендера: результат зависит от того, сколько раз и когда вызван рендер
const renderCountRef = useRef(0);
renderCountRef.current += 1;

return <p>Рендеров: {renderCountRef.current}</p>;
```

```ts
// ❌ значение из props копируется в ref, чтобы «обмануть» зависимости эффекта: эффект работает с устаревшими данными
const filterRef = useRef(filter);
filterRef.current = filter;

// ✅ указываем зависимости честно; если эффект не должен перезапускаться при смене значения,
// в React 19.2+ для этого есть useEffectEvent
useEffect(() => {
  loadUsers(filter);
}, [filter]);
```

```ts
// ❌ useRef(...) вычисляет аргумент на каждом рендере: фабрика вызывается снова и снова, а используется только первый результат
const store = useRef(createUserStore({userId})).current;

// ✅ функция-инициализатор выполняется один раз (см. ADR 2)
const [store] = useState(() => createUserStore({userId}));
```

```tsx
// ❌ значение поля берётся из DOM вручную
const nameRef = useRef<HTMLInputElement>(null);
const handleSubmit = () => save(nameRef.current?.value ?? '');

// ✅ для форм используем react-hook-form (см. ADR 4): он сам работает с неконтролируемыми полями;
// простая форма без библиотеки — контролируемые поля на useState
```

- **Не используем `ref` как замену общему состоянию.** Если данные нужны нескольким компонентам, это стор (ADR 2), а не `ref`, который передаётся вниз.
- **Не строим на `ref` взаимодействие родителя и потомка в обход пропсов.** Императивный доступ к потомку (`focus()`, `scrollTo()`, `reset()`) допустим через `useImperativeHandle` только тогда, когда состояние и пропсы не справляются. Публичный интерфейс при этом делаем минимальным.

#### Контроль

В актуальных версиях `eslint-plugin-react-hooks` есть правило `react-hooks/refs`, которое находит чтение и запись `ref.current` во время рендера. Оно входит в рекомендуемый набор вместе с правилами для React Compiler. Правило `react-hooks/exhaustive-deps` помогает не прятать зависимости эффектов за `ref`.

#### Последствия

- Видимые на экране данные всегда лежат в состоянии, и интерфейс не расходится с данными.
- Рендер остаётся чистым, компоненты совместимы с React Compiler.
- Ресурсы (таймеры, observers, экземпляры библиотек) освобождаются предсказуемо.
- Нужно различать «меняется и отображается» (состояние) и «меняется, но не отображается» (`ref`). Эту границу придётся держать в голове при ревью.
- Императивный код с `ref` сложнее для чтения, чем декларативный на состоянии, поэтому он остаётся исключением для DOM и внешних библиотек.
