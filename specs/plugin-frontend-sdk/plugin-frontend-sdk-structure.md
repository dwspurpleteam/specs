# Frontend Plugin SDK: структура, функции и домены

## 1. Назначение

Документ описывает внутреннюю структуру MVP SDK, публичные функции, используемые домены и примеры интеграции frontend-плагина с Host Frontend.

SDK работает внутри plugin UI, размещённого в `iframe`. Он запрашивает у Host Frontend контекст текущего звонка через `window.postMessage`.

SDK не получает LaunchParams напрямую из backend и не содержит криптографической авторизации, JWT или проверки подписей.

## 2. Термины и домены

| Домен | Ответственность |
|---|---|
| Plugin UI | Frontend-приложение плагина, загруженное в `iframe`. Использует SDK. |
| Plugin SDK | Клиентская библиотека, формирующая запросы и принимающая ответы Host Frontend. |
| Host Frontend | Основное приложение платформы. Хранит контекст звонка и отвечает на запросы плагина. |
| Call | Единая модель информации о звонке в LaunchParams. `call.id` одновременно является идентификатором LiveKit-комнаты. |
| Room | Техническая LiveKit-комната. Её идентификатор не дублируется отдельным объектом в LaunchParams: используется `call.id`. |
| User | Пользователь платформы; идентифицируется `user.id`. |
| Session | Конкретное участие пользователя в звонке; идентифицируется `session.id`. |
| Window bridge | Канал обмена между iframe и Host Frontend через `postMessage`. Проверка origin в MVP не выполняется. |

## 3. Структура пакета

```text
plugin-frontend-sdk/
├── package.json
├── README.md
├── src/
│   ├── index.ts          # публичная точка входа
│   ├── sdk.ts            # init и жизненный цикл SDK
│   ├── bridge.ts         # request/response через postMessage
│   ├── messages.ts       # типы сообщений и коды ошибок
│   ├── types.ts          # LaunchParams и публичные типы
│   ├── validation.ts     # проверка формы ответа
└── test/
    ├── bridge.test.ts
    ├── validation.test.ts
    └── sdk.test.ts
```

## 4. Публичный API

```typescript
export interface PluginSdkOptions {
  /** Максимальное время ожидания ответа, по умолчанию 5000 мс. */
  timeoutMs?: number;
  /** Опциональный объект окна для тестирования. */
  window?: Window;
}

export interface PluginSdk {
  getLaunchParams(): Promise<LaunchParams>;
  destroy(): void;
}

export function init(options: PluginSdkOptions): PluginSdk;
```

### `init(options)`

Создаёт экземпляр SDK. В MVP SDK не принимает и не проверяет origin конфигурации.

### `getLaunchParams()`

Отправляет Host Frontend сообщение `PluginGetLaunchParams` и возвращает Promise с проверенными LaunchParams.

Поведение:

- каждый вызов получает новый `requestId`;
- ответы принимаются по типу сообщения и совпадению `requestId`;
- ответ должен иметь тот же `requestId`;
- тайм-аут завершается ошибкой `HOST_RESPONSE_TIMEOUT`;
- ошибка Host Frontend преобразуется в `PluginSdkError`.

### `destroy()`

Удаляет обработчик `message` и завершает ожидающие запросы. Вызывается при размонтировании plugin UI.

## 5. Внутренние модули

### `bridge.ts`

Отвечает за транспорт:

```typescript
interface Bridge {
  request<TResponse>(message: unknown): Promise<TResponse>;
  destroy(): void;
}
```

Bridge не знает бизнес-смысл LaunchParams. Он сопоставляет `requestId` и направляет ответ нужному Promise. Проверка `event.origin` в MVP не выполняется.

### `validation.ts`

Проверяет, что ответ содержит непустые строки и boolean-поле `isGuest`:

```typescript
function parseLaunchParams(value: unknown): LaunchParams;
```

Проверка формы не является проверкой подлинности. SDK использует данные как контекст запуска.

### `messages.ts`

```typescript
export const MESSAGE_TYPES = {
  getLaunchParams: 'PluginGetLaunchParams',
  launchParamsResult: 'PluginGetLaunchParamsResult',
  launchParamsError: 'PluginGetLaunchParamsError'
} as const;
```

## 6. Ошибки SDK

```typescript
type PluginSdkErrorCode =
  | 'INVALID_OPTIONS'
  | 'HOST_RESPONSE_TIMEOUT'
  | 'HOST_ERROR'
  | 'INVALID_RESPONSE'
  | 'SDK_DESTROYED';

class PluginSdkError extends Error {
  constructor(
    public readonly code: PluginSdkErrorCode,
    message: string
  ) {
    super(message);
  }
}
```

SDK не должен автоматически повторять запросы без явной настройки: повтор может создать несколько активных запросов и усложнить диагностику.

## 7. Пример использования

```typescript
import { init } from '@company/plugin-frontend-sdk';

const sdk = init({
});

const params = await sdk.getLaunchParams();

console.log(params.call.name);
console.log(params.call.id);
console.log(params.user.displayName);
console.log(params.session.id);
```

В React:

```typescript
useEffect(() => {
  const sdk = init({
  });

  sdk.getLaunchParams()
    .then(setLaunchParams)
    .catch(setError);

  return () => sdk.destroy();
}, []);
```

## 8. Требования к Host Frontend

Host Frontend должен:

1. отвечать на запросы, поступающие от plugin iframe;
2. использовать `targetOrigin: '*'`, поскольку проверка origin в MVP не выполняется;
3. не изменять LaunchParams между запросом и ответом;
4. возвращать ошибку, если контекст звонка ещё не готов;
5. очищать сохранённый контекст при выходе из звонка.

## 9. Тестирование

Минимальные тесты:

- успешный запрос и ответ с совпадающим `requestId`;
- обработка сообщения независимо от его origin;
- игнорирование ответа с другим `requestId`;
- тайм-аут при отсутствии ответа;
- обработка `PluginGetLaunchParamsError`;
- отказ при неполном LaunchParams;
- удаление обработчика после `destroy()`;
- два параллельных вызова `getLaunchParams()` с разными `requestId`.
