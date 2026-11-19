# Доработки платформы OpenVidu для передачи LaunchParams

## 1. Цель

Платформа должна передавать контекст звонка в Host при подключении к OpenVidu/LiveKit, чтобы затем Host передал его plugin UI без дополнительного запроса при открытии iframe.

## 2. Компоненты

| Компонент | Изменение |
|---|---|
| Backend платформы | Формировать единый объект `call` и `user.*`, `session.id`, выдавать LiveKit Access Token; `call.id` равен идентификатору комнаты. |
| Backend плагина | Серверная часть конкретного плагина, реализующая его бизнес-логику и при необходимости принимающая запросы frontend-плагина после собственной проверки авторизации. |
| LiveKit Server | Инфраструктурный компонент видеоконференций, обслуживающий комнаты и подключения; не является Backend платформы. |
| Token service | Передавать `session.id`, имя и атрибуты пользователя в LiveKit Access Token. |
| Room provisioning | Создавать комнату и записывать metadata до подключения клиента. |
| Host | Frontend приложения видеоконференций, внутри которого открыт iframe плагина; собирает LaunchParams после подключения и хранит их в состоянии звонка. В документе `Host` и `Host Frontend` — синонимы. |
| Plugin host integration | Обрабатывать `postMessage` и отвечать plugin iframe. Проверка origin не выполняется. |
| Plugin registry | Хранить URL и plugin id без требований к whitelist origin. |
| Tests/observability | Добавить контрактные тесты, логи ошибок и метрики готовности контекста. |

## 3. Backend платформы: модель участия

```typescript
interface CallParticipantContext {
  callId: string;
  callName: string;
  userId: string;
  displayName: string;
  photoUrl?: string;
  isGuest: boolean;
  sessionId: string;
}
```

Правила:

- `userId`, `displayName` и `isGuest` определяются сервером, а не принимаются из недоверенных полей клиента;
- `sessionId` создаётся Backend платформы для нового участия;
- `sessionId` переиспользуется только при восстановлении того же участия;
- разные устройства одного пользователя получают разные `sessionId`;
- `callId` стабилен для одного звонка и одновременно является `room.name` в LiveKit.
- `call` является единым источником информации о звонке для LaunchParams.

## 4. Backend платформы: LiveKit Access Token

```typescript
const token = new AccessToken(API_KEY, API_SECRET, {
  identity: context.sessionId,
  name: context.displayName,
  attributes: {
    'app.userId': context.userId,
    ...(context.photoUrl ? { 'app.photoUrl': context.photoUrl } : {}),
    'app.isGuest': String(context.isGuest)
  }
});

token.addGrant({
  roomJoin: true,
  room: context.callId,
  canPublish: true,
  canSubscribe: true,
  canUpdateOwnMetadata: false
});
```

`participant.identity` используется как `session.id`, поэтому Host получает идентификатор участия из `localParticipant.identity` без дополнительного запроса.

## 5. Создание комнаты и metadata

До выдачи токена клиенту необходимо обеспечить наличие комнаты и metadata:

```typescript
await roomService.createRoom({
  name: context.callId,
  metadata: JSON.stringify({
    callName: context.callName
  })
});
```

Если комната уже существует, metadata должна быть установлена или проверена до подключения первого участника. Иначе Host может получить неполный LaunchParams.

## 6. API выдачи подключения

Существующий endpoint, выдающий OpenVidu/LiveKit token, должен гарантировать наличие данных участия:

```typescript
interface JoinCallResponse {
  token: string;
  livekitUrl: string;
  sessionId: string;
}
```

Если часть metadata комнаты недоступна клиенту, endpoint может вернуть `launchContext` в том же ответе:

```typescript
interface JoinCallResponseWithContext extends JoinCallResponse {
  launchContext: LaunchParams;
}
```

Это не является отдельным запросом при открытии плагина. Значения должны формироваться сервером.

## 7. Host: сбор и хранение

После `room.connect()` Host должен:

1. прочитать `room.name` и `room.metadata`;
2. прочитать `localParticipant.identity`, `name` и `attributes`;
3. преобразовать значения к типам контракта;
4. отклонить неполный контекст;
5. сохранить LaunchParams в состоянии текущего звонка;
6. удалить их при disconnect или смене звонка.

```typescript
await room.connect(livekitUrl, token);

const launchParams = getLaunchParams(room);
callStore.setLaunchParams(launchParams);
```

Состояние должно быть привязано к текущему подключению, чтобы контекст предыдущего звонка не попал в новый iframe.

## 8. Host: bridge с iframe

```typescript
window.addEventListener('message', (event) => {
  if (event.data?.type !== 'PluginGetLaunchParams') return;

  const { requestId } = event.data;
  const launchParams = callStore.getLaunchParams();

  if (!launchParams) {
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
    return;
  }

  iframe.contentWindow?.postMessage(
    {
      type: 'PluginGetLaunchParamsResult',
      requestId,
      data: launchParams
    },
    '*'
  );
});
```

Проверка `event.origin` и `event.source` в текущем контракте не выполняется. Ответ отправляется с `targetOrigin: '*'`.

## 9. Реестр плагинов и домены

```typescript
interface PluginRegistration {
  pluginId: string;
  iframeUrl: string;
  enabled: boolean;
}
```

Платформа не ведёт whitelist origin для передачи LaunchParams. Plugin UI может выполнять любые сетевые запросы, разрешённые политиками браузера и окружения.

## 10. Изменения OpenVidu/LiveKit-интеграции

Адаптер платформы должен обеспечить:

- прокидывание server-generated `identity` в LiveKit participant identity;
- прокидывание имени в participant name;
- прокидывание `userId`, необязательного `photoUrl` и `isGuest` в participant attributes;
- доступ Host к room metadata после подключения;
- установку room metadata до подключения участников;
- сохранение связи между OpenVidu call id и LiveKit room id.

Если текущая версия OpenVidu не предоставляет нужные значения в клиентском API, расширяется адаптер событий подключения или endpoint выдачи токена. SDK плагина не должен обращаться к внутренним OpenVidu API.

## 11. Ограничения безопасности текущего контракта

Даже без подписей необходимо:

- не помещать в LaunchParams секреты и access token;
- не принимать идентификаторы пользователя от plugin UI;
- учитывать, что данные передаются в iframe без проверки origin;
- разрешать plugin UI выполнять сетевые запросы без ограничений со стороны SDK;
- считать LaunchParams контекстом UI, а не основанием для доверенных операций Backend плагина;
- для защищённых операций использовать отдельную серверную авторизацию платформы.

## 12. Наблюдаемость

Рекомендуемые события:

- `call.launch_context.created`;
- `call.launch_context.ready`;
- `call.launch_context.incomplete`;
- `plugin.launch_params.requested`;
- `plugin.launch_params.responded`;
- `plugin.launch_params.failed`.

В логи не следует записывать access token и полные персональные данные. Для диагностики достаточно `call.id`, `session.id`, `pluginId` и кода ошибки. `call.id` одновременно идентифицирует LiveKit-комнату.

## 13. План реализации

1. Зафиксировать тип `LaunchParams` и сообщения в общем контрактном пакете.
2. Доработать Backend платформы для выдачи OpenVidu/LiveKit token.
3. Добавить создание и metadata комнаты.
4. Доработать Host Frontend: сбор, хранение и очистку контекста.
5. Реализовать bridge для iframe без проверки origin.
6. Реализовать SDK поверх bridge.
7. Добавить контрактные и интеграционные тесты.
8. Проверить два устройства одного пользователя и reconnect.

## 14. Критерии готовности

- После подключения Host получает полный LaunchParams без нового запроса при открытии iframe.
- Разные устройства получают разные `session.id`.
- Reconnect не создаёт ошибочно новый контекст, если это то же участие.
- Плагин получает параметры через `postMessage` без проверки origin.
- Host отвечает iframe без проверки origin.
- При неполном контексте плагин получает явную ошибку.
- В текущем контракте не требуется подпись LaunchParams.
