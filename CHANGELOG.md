# Changelog форка

Изменения ветки `custom` относительно upstream ([hoge128/bridge-lite](https://github.com/hoge128/bridge-lite)).
История самого BridgeLite — в `docs/releases/`.

Версии: `CFBundleShortVersionString` остаётся равной версии upstream, на которой основан форк;
растёт только номер сборки `CFBundleVersion` (`./tools/bump-version.sh <версия upstream>`).
Здесь релиз форка обозначается `<версия upstream>-fork.<N>`.

## 0.5.2-fork.1 (build 23) — 2026-10-27

Основа: upstream 0.5.2 (build 22).

### Добавлено
- Открытие папки из терминала: `tools/bridgelite [папка]` (по умолчанию текущая). Папка становится рабочей,
  как при File → Open Folder…. Работает и аргументом бинарника `BridgeLite.app/Contents/MacOS/BridgeLite <папка>`.
- `CLAUDE.md` с правилами сборки, архитектуры и производительности для работы с Claude Code.

### Изменено
- Открытие папки из Finder, Dock и терминала теперь проходит те же проверки, что и Open Folder…
  (предупреждение о медленном диске HDD/сеть).

### Отключено
- Проверка и установка обновлений Sparkle: убраны `SUFeedURL`, пункт меню «Check for Updates…»
  и секция «Updates» в настройках. Сборка форка больше не обращается к appcast автора.

### Тесты
- Тесты XMP используют файлы из `crates/bridge-core/tests/fixtures/` вместо локальных путей автора:
  `cargo test -p bridge-core` проходит полностью (раньше 3 теста падали, 2 молча пропускались).
