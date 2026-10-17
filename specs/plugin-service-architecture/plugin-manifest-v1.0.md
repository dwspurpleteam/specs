# Манифест плагина

```json
{
  // мета-информация о плагине
  "manifest-version": "1.0",
  "plugin-id": "plugin-1",
  "plugin-version": "0.0.0",
  "name": "test-plugin",
  "description": "xxx",
  // настройки отображения в iframe
  "frontend": {
    // от версии sdk в теории зависят передаваемые параметры, 
    // но еще можно использовать чтобы блокировать плагины с sdk версии меньше чем разрешенная
    "sdk-version": "1.0.0",
    // url-web есть префикс, т.к. в дальнейшем можно разные ссылки иметь для разных платформ 
    // по аналогии с мини приложениями вконтакте
    "url-web": "https://example.com/plugin-ui/web",
  },
  // настройки бекенда
  "backend": {
    // аналогично как и на фронте
    "sdk-version": "1.0.0",
    // урл для проверки состояния бекенда плагина
    "health-url": "https://example.com/api/healthzz",
    // описание получаемых событий
    "events": {
      // url для отправки событий
      "event-url": "https://example.com/api/events",
      // описание ивентов
      "available-event-types": [
        "participant.joined", // подключился участник
        "participant.left", // отключился участник
        "participant.connection_aborted", // соединение отклонило запрос
        "participant.reconnecting", // переподключение участнкиа
        "participant.reconnected", // участник переподключился
        "participant.kicked", // участника выгнали :)

        "media.audio_muted", // замьютил микрофон
        "media.audio_unmuted", // размьютил микрофон
        "media.video_enabled", // включил видео
        "media.video_disabled", // отключил видео
        "media.screen_share_started", // начал трансляцию экрана
        "media.screen_share_stopped", // закончил трансляцию экрана
        "media.track_published", // начал публиковать любой медиа-поток
        "media.track_unpublished", // закончил публиковать любой медиа-поток

        "interaction.chat_message_received", // получено сообщение
      ],
      // описание групп ивентов
      "available-event-groups": ["participant", "media", "interaction"],
    },

    "media": {
      "input": ["AUDIO", "VIDEO", "SCREEN_SHARE", "SCREEN_SHARE_AUDIO"],
      "output": ["AUDIO", "VIDEO", "SCREEN_SHARE", "SCREEN_SHARE_AUDIO"],
    }
  }
}
```

# Доп описание

Блоки `available-event-groups` и `available-event-types` являются взаимоисключающими, при наличии обоих `available-event-groups` будет проигнорирован и будет использоваться исключительно частное перечисление событий

# Возможные доработки в следующих версиях манифеста
Добавление stop-policy в настройку плагина

Для блока media можно добавить параметр `"connection-type": "PERMANENT | ON_CONNECTION"`. 1 значение означает, что плагин будет держать соединение всегда пока не придет сообщение о завершении звонка, 2 значение означает, что будем выполнять соединение только при получении track_published и завершать при получении track_unpublished.
