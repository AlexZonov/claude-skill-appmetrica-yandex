# appmetrica-yandex-versions

Claude Code skill: подбирает совместимые версии **AppMetrica Unity plugin** и **AppLovin MAX Yandex Ads adapter**.

Оба тянут один и тот же нативный AppMetrica SDK. При расхождении Gradle разрешает конфликт молча — билд зелёный, комбинация SDK непротестированная, сообщения об ошибке нет. Скилл делает подбор по таблице вместо угадывания.

## Установка

```bash
git clone https://github.com/AlexZonov/claude-skill-appmetrica-yandex.git ~/.claude/skills/appmetrica-yandex-versions
```

Для одного проекта — клонировать в `<project>/.claude/skills/` вместо `~/.claude/`.

## Использование

Скилл срабатывает сам, когда в запросе есть версия любого из двух плагинов или упоминание `io.appmetrica.analytics` / `com.applovin.mediation:yandex-adapter` / `com.yandex.android:mobileads` в контексте совместимости. Можно вызвать явно: `/appmetrica-yandex-versions`.

```
Хочу поднять AppMetrica Unity до 6.9.0 — какой yandex-adapter ставить?
```

## Что внутри

| Файл | Что это |
|---|---|
| [SKILL.md](SKILL.md) | инструкция и формат ответа |
| [references/version-matrix.md](references/version-matrix.md) | таблицы соответствий + скрипт обновления из Maven / CocoaPods |
| [evals/evals.json](evals/evals.json) | тест-кейсы для `skill-creator` |

## Обновление таблицы

Таблица — снапшот, оба SDK выходят раз в несколько недель. Скрипт регенерации лежит в конце `references/version-matrix.md`; результат коммитить обратно в этот репозиторий.
