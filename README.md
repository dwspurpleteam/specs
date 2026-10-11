# Spec-Driven Development

- `specs/<name>/spec.md` - каждая спецификация в отдельной папке с именем без нумерации.
- [templates/spec.md](templates/spec.md) - шаблон спецификации.

До реализации описываем архитектуру, взаимодействия и требования. Спека описывает устройство сервиса; отчеты о реализации и проверках в нее не добавляем.

```sh
mkdir -p specs/plugin-service
cp templates/spec.md specs/plugin-service/spec.md
```
