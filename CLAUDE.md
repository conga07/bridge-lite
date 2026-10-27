# CLAUDE.md — форк bridge-lite (личное использование)

Форк https://github.com/hoge128/bridge-lite (origin: conga07/bridge-lite, upstream: hoge128). Лицензии у upstream нет:
форк только для личных правок, **не распространять** сборки и не публиковать (App Store и т.п.).
Исходные документы автора (на японском): `PHILOSOPHY.md`, `CORE-GUI.md`, `knowledge/`. Часть `CORE-GUI.md`
устарела (имена файлов, количество тестов) — при расхождении верить коду.

## Сборка и проверка
- Окружение: Xcode 27.1, Rust через Homebrew rustup (`/opt/homebrew/opt/rustup/bin` в PATH), xcodegen.
- Порядок: `./tools/build-rust-xcframework.sh --release` → `cd xcode/BridgeLite && xcodegen generate` → `make xcode`
  (или `xcodebuild -project xcode/BridgeLite/BridgeLite.xcodeproj -scheme BridgeLite -configuration Debug -destination "platform=macOS,arch=arm64" build`).
- После правок Rust-API (`crates/bridge-ffi`) всегда перезапускать build-rust-xcframework.sh: он пересоздаёт и патчит
  `xcode/BridgeLite/Generated/` (патчи Swift 6 через sed; не править эти файлы вручную).
- Тесты Rust: `cargo test -p bridge-core`. Тестовые файлы лежат в `crates/bridge-core/tests/fixtures/`
  (`test/` в .gitignore — на него не опираться, абсолютные пути не использовать).
  Не открывать `tests/fixtures/` в самом приложении: оно переписывает XMP sidecar (например `xmp:Label="選択"` → `Red`)
  и портит фикстуры. Для ручной проверки копировать файлы во временную папку.
- **Никогда не править `project.pbxproj` вручную.** Файлы добавлять/удалять и менять настройки через
  `xcode/BridgeLite/project.yml` + `xcodegen generate`. Если пропал `Package.resolved` — закрыть Xcode и
  `git checkout HEAD -- <путь к Package.resolved>` (см. README).
- Проверять изменения реальной сборкой; UI на глаз проверяет пользователь.

## Изменения форка относительно upstream
- Sparkle отключён: нет `SUFeedURL`, `startingUpdater: false`, убраны пункт меню и секция настроек «Updates».
  Не возвращать проверку обновлений на сайт автора.
- Тесты xmp используют `fixture()` вместо путей автора.
- Открытие папки из терминала: `tools/bridgelite [папка]` (через `open -b io.github.bridge-lite`, чтобы не попасть
  в App Store-версию `io.github.bridge-lite.ios`) или аргументом бинарника. Все внешние пути (Finder, Dock, терминал)
  идут через `LibraryStore.openFolderWithChecks` — так же, как Open Folder….

## Философия (PHILOSOPHY.md) — ограничения на любые новые функции
1. Приложение только **смотрит, сравнивает, оценивает**. Никакой проявки, ретуши, экспорта, правки пикселей.
2. Файлы пользователя не меняем. Единственное исключение — XMP (рейтинг/метка/флаг): sidecar, у JPG — embedded.
3. Источник истины — файловая система (фото + XMP sidecar), совместимость с Adobe Bridge. БД — только кэш.
4. RAW+JPG+проявленные варианты — одна «съёмка» (ShotGroup), между вариантами можно переключаться.
5. Функций добавлять как можно меньше: каждая замедляет приложение. Сначала взвесить, нужна ли она.

## Архитектура (CORE-GUI.md)
- `SwiftUI → bridge-ffi → bridge-core → Adobe XMP Toolkit`. Зависимость только в эту сторону.
- **Rust (bridge-core)**: доступ к файлам, EXIF/XMP, SQLite (WAL), pHash, группировка (`pairing.rs`), сохранение btime.
  Не зависит от GUI-крейтов и не вызывает ImageIO/CGImage/NSCache.
- **Swift**: отрисовка, CGImageSource/ImageIO, NSCache, ввод, локализация, OS-интеграция.
  Swift не парсит XMP, не ходит в SQLite напрямую, не вызывает `setattrlist`, не реализует группировку/pHash.
- Запись XMP целиком в Rust: tmp → fsync → atomic rename → восстановление btime. Swift-код файлы не пишет.
  В UI — оптимистичное обновление с откатом при ошибке.
- Типы FFI (`FfiImageEntry` и др.) не выходят за слой Bridging: сразу конвертируются в чистые Swift-типы (`PhotoEntry`).
- ImageIO: `kCGImageSourceShouldCache: false`, миниатюры с `CreateThumbnailWithTransform`; кэш — NSCache (L1) → SQLite (L2) → генерация (L3).
  RAW: не через ImageIO, а `extract_raw_embedded_jpeg` (обход ошибок RawCamera на macOS 26).
- Ограничение параллелизма через `ConcurrencyLimiter` (миниатюры 6, pHash 4). Типы между акторами — `Sendable`.
- Ошибки: `CoreError` в Rust → Swift `throws`; пользовательские тексты только через `Localizable.xcstrings` (ja/en).
  Любую новую строку UI добавлять в оба языка. Rust логирует на английском в stderr, Swift — через `os.Logger`.

## Производительность UI (knowledge/grid-scroll-performance.md) — запреты
Сетка миниатюр на 3000+ фото деградировала из-за AppKit-объектов в каждой ячейке. Не повторять:
- В `ThumbnailCellView` и любых ячейках сетки **нельзя**: `.contextMenu`, `.onHover`, `NSViewRepresentable`
  (правый клик, drag), per-cell `NSEvent` monitor, `DragGesture`.
- Мышь (клик, ⌘/⇧-выделение, двойной клик, рамка выделения, drag в Finder, правое меню) добавлять только в
  `GridInteractionNSView` (один слой на всю сетку, `ThumbnailGridView.swift`). Двойной клик — через `NSEvent.doubleClickInterval`.
- В теле ячейки не читать `store.selectedIDs`: использовать `@State isSelected` + подписку на `selectionDidUpdate`
  (аналогично `thumbnailDidUpdate`/`exifDidUpdate`/`xmpDidUpdate` — уведомления по id, а не перерисовка всего).
- Запрещено на горячем пути: `TapGesture(count: 2)` вместе с одиночным тапом, spring-анимация выбора,
  `.ultraThinMaterial` в корне `SidebarView`/`FilterPanelView` и на ячейках, создание `DateFormatter` в `body`, `print`.
- Сравнивать скорость только в одинаковой конфигурации (Debug с Debug, Release с Release): Debug в разы медленнее.
- Если стало лениво скроллиться — сначала искать нарушение этих правил, а не кэш памяти.

## Как работать в этом репозитории
- Правки делать в отдельной ветке (например `custom`), `master` держать зеркалом upstream (`git fetch upstream`).
- Коммиты и push — только по просьбе пользователя.
- Не смешивать несвязанные изменения; не трогать Generated/ и `.xcodeproj` руками.
