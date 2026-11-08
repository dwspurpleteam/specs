# Frontend Plugin SDK — спецификация MVP

## 1. Назначение

SDK предназначен для передачи контекста текущего звонка из Host Frontend во frontend-плагин, открытый внутри `iframe`.

В MVP SDK используется для первичной инициализации plugin UI. Контекст формируется на backend при подготовке подключения к LiveKit, попадает в браузер вместе с данными подключения, хранится в Host Frontend и передаётся плагину через `postMessage`.

При открытии iframe дополнительные запросы к backend для получения LaunchParams не выполняются.

## 2. Возможности SDK

SDK предоставляет:

- единый тип `LaunchParams`;
- получение параметров из Host Frontend через bridge;
- обмен сообщениями между iframe и Host Frontend через `postMessage`;
- корреляцию запроса и ответа по `requestId`;
- обмен сообщениями через `postMessage` без проверки `origin`;
- понятные ошибки, если Host Frontend не подключён или контекст ещё не готов.

## 3. LaunchParams

```typescript
export interface LaunchParams {
  call: {
    /** Единый идентификатор звонка и LiveKit-комнаты. */
    id: string;
    /** Отображаемое название звонка/комнаты. */
    name: string;
  };

  user: {
    /** Идентификатор пользователя платформы. */
    id: string;
    /** Отображаемое имя пользователя. */
    displayName: string;
    /** Необязательный URL фотографии пользователя. */
    photoUrl?: string;
    /** true, если пользователь не авторизован в платформе. */
    isGuest: boolean;
  };

  session: {
    /** Уникальный идентификатор конкретного участия в звонке. */
    id: string;
  };
}
```

`user.id` может быть одинаковым на нескольких устройствах, а `session.id` должен различаться для независимых подключений одного пользователя. При временном reconnect обычно сохраняется тот же `session.id`.

Пример:

```json
{
  "call": {
    "id": "room-123",
    "name": "Команда разработки"
  },
  "user": {
    "id": "user-789",
    "displayName": "Александр",
    "photoUrl": "https://cdn.example.com/users/user-789/avatar.jpg",
    "isGuest": false
  },
  "session": {
    "id": "session-device-a"
  }
}
```

## 4. Формирование параметров на backend

Backend, который выдаёт LiveKit Access Token, формирует значения из доверенного серверного контекста:

1. определяет `user.id`, `displayName`, необязательный `photoUrl` и `isGuest` на основании текущей сессии;
2. получает или создаёт единый `call.id`, который используется как LiveKit `room.name`;
3. определяет `call.name`;
4. создаёт `session.id` для конкретного участия;
5. передаёт необходимые значения в LiveKit при подготовке подключения.

Пример формирования токена:

```typescript
const sessionId = crypto.randomUUID();

const token = new AccessToken(LIVEKIT_API_KEY, LIVEKIT_API_SECRET, {
  identity: sessionId,
  name: user.displayName,
  attributes: {
    'app.userId': user.id,
    ...(user.photoUrl ? { 'app.photoUrl': user.photoUrl } : {}),
    'app.isGuest': String(user.isGuest)
  }
});

token.addGrant({
  roomJoin: true,
  room: room.id,
  canPublish: true,
  canSubscribe: true,
  canUpdateOwnMetadata: false
});

return {
  token: await token.toJwt(),
  sessionId
};
```

Метаданные комнаты могут содержать данные звонка:

```json
{
  "callName": "Команда разработки"
}
```

Создание комнаты и установка её metadata должны завершиться до подключения участника, чтобы Host Frontend получил полный контекст после `room.connect()`.

## 5. Получение контекста в Host Frontend

После подключения к LiveKit Host Frontend собирает параметры из `Room` и `LocalParticipant`:

```typescript
function getLaunchParams(room: Room): LaunchParams {
  const participant = room.localParticipant;
  const metadata = JSON.parse(room.metadata || '{}') as {
    callName?: string;
  };

  const userId = participant.attributes['app.userId'];
  const photoUrl = participant.attributes['app.photoUrl'];
  const guestFlag = participant.attributes['app.isGuest'];

  if (
    !room.name ||
    !metadata.callName ||
    !userId ||
    (guestFlag !== 'true' && guestFlag !== 'false') ||
    !participant.identity
  ) {
    throw new Error('Incomplete plugin launch context');
  }

  return {
    call: { id: room.name, name: metadata.callName },
    user: {
      id: userId,
      displayName: participant.name ?? '',
      ...(photoUrl ? { photoUrl } : {}),
      isGuest: guestFlag === 'true'
    },
    session: { id: participant.identity }
  };
}
```

Host сохраняет результат во внутреннем состоянии приложения до закрытия или смены звонка. Например:

```typescript
await room.connect(livekitUrl, token);

const launchParams = getLaunchParams(room);
pluginContextStore.set(launchParams);
```

## 6. Передача в iframe через postMessage

### 6.1. Запрос из plugin UI

Плагин отправляет Host Frontend запрос после загрузки iframe:

```typescript
window.parent.postMessage(
  {
    type: 'PluginGetLaunchParams',
    requestId: crypto.randomUUID()
  },
  '*'
);
```

### 6.2. Ответ Host Frontend

Host проверяет тип сообщения и готовность контекста, затем отправляет ответ:

```typescript
iframe.contentWindow?.postMessage(
  {
    type: 'PluginGetLaunchParamsResult',
    requestId,
    data: launchParams
  },
  '*'
);
```

Если контекст недоступен:

```typescript
iframe.contentWindow?.postMessage(
  {
    type: 'PluginGetLaunchParamsError',
    requestId,
    error: {
      code: 'LAUNCH_CONTEXT_NOT_READY',
      message: 'Launch context is not ready'
    }
  },
  '*'
);
```

В MVP проверка `origin` не выполняется, поэтому используется `targetOrigin: '*'`.

## 7. Структура SDK

Минимальная структура пакета:

```text
src/
  index.ts              # публичный API
  bridge.ts             # request/response через postMessage
  messages.ts           # типы сообщений и кодов ошибок
  types.ts              # LaunchParams и публичные типы
```

Публичный API может выглядеть так:

```typescript
export interface PluginSdk {
  getLaunchParams(): Promise<LaunchParams>;
}

export function init(options: {
  timeoutMs?: number;
}): PluginSdk;
```

## 8. Пример чтения параметров в плагине

```typescript
import { init } from '@company/plugin-frontend-sdk';

const sdk = init({
  timeoutMs: 5000
});

try {
  const params = await sdk.getLaunchParams();

  document.querySelector('[data-room-name]')!.textContent =
    params.call.name;
  document.querySelector('[data-user-name]')!.textContent =
    params.user.displayName;

  console.log('Call:', params.call.id);
  console.log('User:', params.user.id);
  console.log('Session:', params.session.id);
} catch (error) {
  console.error('Unable to load plugin launch context', error);
}
```

Низкоуровневый вариант без SDK:

```typescript
const requestId = crypto.randomUUID();

window.parent.postMessage(
  { type: 'PluginGetLaunchParams', requestId },
  'https://app.example.com'
);

window.addEventListener('message', (event) => {
  if (event.data?.type !== 'PluginGetLaunchParamsResult') return;
  if (event.data.requestId !== requestId) return;

  const params = event.data.data as LaunchParams;
  console.log(params.user.displayName);
});
```

## 9. Полная последовательность работы

1. Пользователь открывает или присоединяется к звонку.
2. Backend формирует LiveKit Access Token и данные участия.
3. Host Frontend подключается к LiveKit.
4. Host получает metadata комнаты и данные локального участника.
5. Host собирает `LaunchParams` и сохраняет их в состоянии приложения.
6. Host открывает plugin UI в iframe.
7. Plugin SDK отправляет `PluginGetLaunchParams` через `postMessage`.
8. Host возвращает сохранённые параметры через `PluginGetLaunchParamsResult`.
9. Плагин использует параметры для инициализации интерфейса.

На шаге 7 отдельный запрос к backend не выполняется.

## 10. Критерии приёмки MVP

- `LaunchParams` содержит только согласованные поля `room`, `call`, `user` и `session`.
- Параметры формируются на backend при подготовке подключения к LiveKit.
- `session.id` различается для параллельных подключений одного пользователя с разных устройств.
- Host получает и сохраняет контекст после подключения к комнате.
- Плагин получает контекст через `postMessage` без дополнительного backend-запроса.
- Запрос и ответ связываются через `requestId`.
- Host и SDK не выполняют проверку `origin`.
- В MVP отсутствуют `sign`, JWT, алгоритмы подписи, ключи, `iat`, `exp`, `jti`, проверка подписи и авторизация через подпись.

## 11. Будущие расширения

Если позже потребуется доверенная авторизация backend плагина, это следует добавлять отдельным протоколом и отдельной версией API. Такая функция не входит в текущий MVP и не должна подразумеваться при использовании `LaunchParams`.
