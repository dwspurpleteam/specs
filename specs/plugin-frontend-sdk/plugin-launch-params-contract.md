# Контракт передачи LaunchParams

## 1. Назначение

Контракт определяет структуру LaunchParams, правила формирования контекста на Backend платформы, его хранение в Host и обмен между Host и plugin UI.

Контракт предназначен для первичной инициализации плагина. Он не является механизмом аутентификации или криптографического доказательства.

В текущем контракте отсутствуют `sign`, JWT, ключи, подписи, сроки действия подписей и проверка подписи.

## 2. Модель данных

```typescript
export interface LaunchParams {
  call: {
    /** Идентификатор звонка; совпадает с идентификатором LiveKit-комнаты. */
    id: string;
    /** Отображаемое название звонка/комнаты. */
    name: string;
  };
  user: {
    id: string;
    displayName: string;
    photoUrl?: string;
    isGuest: boolean;
  };
  session: {
    id: string;
  };
}
```

| Путь | Тип | Обязательность | Источник | Описание |
|---|---|---|---|---|
| `call.id` | `string` | Да | LiveKit `room.name` и backend платформы | Единый идентификатор звонка и комнаты. |
| `call.name` | `string` | Да | Room metadata | Отображаемое название звонка/комнаты. |
| `user.id` | `string` | Да | Серверная сессия пользователя | Идентификатор пользователя платформы. |
| `user.displayName` | `string` | Да | LiveKit participant name | Имя для отображения. |
| `user.photoUrl` | `string` | Нет | Серверный профиль пользователя | URL фотографии пользователя. |
| `user.isGuest` | `boolean` | Да | Серверная сессия пользователя | Признак неавторизованного гостя. |
| `session.id` | `string` | Да | LiveKit participant identity | Идентификатор конкретного участия. |

## 3. Правила идентификаторов

- `user.id` идентифицирует пользователя и может совпадать на нескольких устройствах.
- `session.id` идентифицирует конкретное участие в звонке.
- Параллельные подключения одного пользователя должны иметь разные `session.id`.
- Временный reconnect должен сохранять `session.id`, если это продолжение того же участия.
- Новое независимое участие получает новый `session.id`.
- `call` является единственным источником информации о звонке в LaunchParams.
- `call.id` общий для всех участников одного звонка и равен идентификатору LiveKit-комнаты.
- Отдельный объект `room` в LaunchParams не используется, чтобы не дублировать информацию о звонке.

## 4. Пример JSON

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

## 5. Компоненты и формирование контекста

**Backend платформы** — серверная часть приложения видеоконференций, отвечающая за подключение пользователя к звонку, подготовку данных участия и выдачу LiveKit Access Token. Именно он определяет `user.id`, `session.id` и данные звонка и передаёт соответствующую информацию в LiveKit.

**Backend плагина** — серверная часть конкретного плагина, реализующая его бизнес-логику. При необходимости она принимает запросы frontend-плагина и использует переданный контекст после надлежащей проверки авторизации.

**LiveKit Server** — инфраструктурный компонент видеоконференций. Он принимает данные подключения и обслуживает комнату, но не является Backend платформы.

Backend платформы формирует значения до выдачи LiveKit Access Token. Клиент не должен самостоятельно задавать `user.id`, `isGuest` или `call.id` для последующей передачи в LaunchParams.

```typescript
const sessionId = crypto.randomUUID();

const token = new AccessToken(LIVEKIT_API_KEY, LIVEKIT_API_SECRET, {
  identity: sessionId,
  name: user.displayName,
  attributes: {
    'app.userId': user.id,
    'app.isGuest': String(user.isGuest)
  }
});

return {
  token: await token.toJwt(),
  sessionId,
  launchContext: {
    call: { id: room.id, name: room.name },
    user: {
      id: user.id,
      displayName: user.displayName,
      ...(user.photoUrl ? { photoUrl: user.photoUrl } : {}),
      isGuest: user.isGuest
    },
    session: { id: sessionId }
  }
};
```

Рекомендуется использовать один источник истины: значения в `launchContext` и данные, передаваемые в LiveKit, должны быть сформированы из одного серверного объекта участия.

## 6. Передача контекста через LiveKit

Под передачей контекста через LiveKit здесь понимается использование полей уже существующего подключения, а не отдельный сетевой протокол Frontend SDK. Backend платформы формирует данные пользователя и звонка при подготовке подключения и помещает их в соответствующие поля LiveKit. После подключения Host считывает эти поля и преобразует их в публичный объект `LaunchParams`.

Минимальное размещение полей:

| LiveKit-механизм | Поля |
|---|---|
| `room.name` | `call.id` |
| `room.metadata` | `call.name` |
| `participant.identity` | `session.id` |
| `participant.name` | `user.displayName` |
| `participant.attributes` | `user.id`, `user.photoUrl`, `user.isGuest` |

`callName` — ключ внутри `room.metadata`, а `call.name` — поле публичного объекта `LaunchParams`. Host преобразует значение `room.metadata.callName` в `LaunchParams.call.name`:

```text
room.metadata.callName = "Команда разработки"
→ launchParams.call.name = "Команда разработки"
```

Пример metadata комнаты:

```json
{
  "callName": "Команда разработки"
}
```

Host собирает финальный объект после успешного `room.connect()` и валидирует обязательные поля до сохранения.

## 7. Контракт postMessage

### Запрос

```typescript
interface GetLaunchParamsMessage {
  type: 'PluginGetLaunchParams';
  requestId: string;
}
```

Пример:

```json
{
  "type": "PluginGetLaunchParams",
  "requestId": "request-001"
}
```

### Успешный ответ

```typescript
interface LaunchParamsResultMessage {
  type: 'PluginGetLaunchParamsResult';
  requestId: string;
  data: LaunchParams;
}
```

### Ответ с ошибкой

```typescript
interface LaunchParamsErrorMessage {
  type: 'PluginGetLaunchParamsError';
  requestId: string;
  error: {
    code: 'LAUNCH_CONTEXT_NOT_READY' | 'INVALID_REQUEST';
    message: string;
  };
}
```

### Правила доставки

- Plugin UI отправляет сообщение в `window.parent`.
- Plugin UI отправляет запрос в `window.parent` без проверки origin.
- Host отправляет ответ в `iframe.contentWindow` с `targetOrigin: '*'`.
- Ответ содержит тот же `requestId`, что и запрос.
- Проверка `event.origin` в текущем контракте не выполняется.

Пример:

```typescript
const requestId = crypto.randomUUID();

window.parent.postMessage(
  { type: 'PluginGetLaunchParams', requestId },
  '*'
);
```

## 8. Жизненный цикл

1. Backend платформы определяет пользователя, звонок, комнату и участие.
2. Backend платформы создаёт LiveKit Access Token с `session.id` и атрибутами участника.
3. Host подключается к комнате.
4. Host получает metadata комнаты и данные локального участника.
5. Host создаёт и сохраняет LaunchParams.
6. Plugin UI открывается в iframe.
7. SDK запрашивает LaunchParams через `postMessage`.
8. Host возвращает сохранённый объект.
9. При выходе из звонка Host удаляет LaunchParams.

## 9. Ошибки и ограничения

Host возвращает `LAUNCH_CONTEXT_NOT_READY`, если участник ещё не подключён или обязательная metadata отсутствует. Пустые значения не подставляются.

LaunchParams отражает состояние на момент подключения и не заменяет отдельный API для получения актуальных данных звонка или пользователя.
