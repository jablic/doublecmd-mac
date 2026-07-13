# doublecmd-mac — план реализации

Форк [doublecmd/doublecmd](https://github.com/doublecmd/doublecmd) (upstream remote).
Цели: (1) независимое дерево каталогов на каждой панели, (2) нативный
macOS/Cocoa рендер-слой в духе ForkLift.

## Текущая архитектура (как есть)

- Дерево — **один** глобальный экземпляр `TShellTreeView` (`src/fmain.pas:538`),
  живёт в `TreePanel`, включается/выключается флагом `gSeparateTree`
  (`src/uglobs.pas:312`, дефолт `False`).
- Создание/апдейт — `TfrmMain.UpdateShellTreeView` (`src/fmain.pas:4956`).
  Дерево следует за активной панелью через `SetFileSystemPath(ActiveFrame, Path)`
  (`ShellTreeViewSelect`, `src/fmain.pas:1784`) и `UpdateTreeViewPath`
  (`src/fmain.pas:5005`, читает `ActiveFrame.CurrentPath`).
- Layout в `src/fmain.lfm`: `TreePanel` → `TreeSplitter` → `pnlLeft` (хост
  `FrameLeft`) → `MainSplitter` → `pnlRight` (хост `FrameRight`). Дерево
  физически прибито к левому краю окна, не к панели.
- Тоггл — `cm_TreeView` (`src/umaincommands.pas:2403`), просто инвертирует
  `gSeparateTree` и переcоздаёт/показывает `TreePanel`.
- Оба файловых фрейма (`FrameLeft`/`FrameRight`) — экземпляры одного класса
  `TColumnsFileView`/`TFileView`, полностью симметричны — это важно: дублировать
  дерево не потребует дублировать логику панелей, только сам `TShellTreeView`.

## Фаза 1 — независимое дерево на обеих панелях

Цель: `FrameLeft` и `FrameRight` каждый получает свой `TShellTreeView`,
синхронный со своим списком файлов, свой toggle.

1. **Данные**: заменить `gSeparateTree: Boolean` на `gSeparateTreeLeft`,
   `gSeparateTreeRight` (или массив `[0..1]`) в `uglobs.pas`, с миграцией
   старого XML-ключа `SeparateTree` → `SeparateTreeLeft` при загрузке конфига
   (`uglobs.pas:2955`).
2. **Layout**: в `fmain.lfm` добавить зеркальную пару `TreePanelRight` +
   `TreeSplitterRight`, пристыкованные `alRight` сразу после `pnlRight`
   (перед правым краем формы, симметрично `TreePanel`/`TreeSplitter` слева).
3. **Компонент**: превратить `TShellTreeView`-логику из `fmain.pas` в
   переиспользуемый инстанс, параметризованный `TFileView` (какой фрейм
   слушать) и `TPanel` (куда монтироваться) — по сути тот же код
   `UpdateShellTreeView`, вызываемый дважды с разными аргументами вместо
   единственного глобального поля `ShellTreeView: TCustomTreeView`.
4. **События**: `ShellTreeViewSelect`/`ShellTreeViewKeyDown`/
   `ShellTreeViewMouseUp` сейчас жёстко используют `ActiveFrame` — переписать
   так, чтобы каждый инстанс дерева знал свой закреплённый фрейм (левое дерево
   всегда бьёт по `FrameLeft`, даже если активна правая панель).
5. **Синхронизация**: `UpdateTreeViewPath` дергать из обоих
   `TFileView.OnAfterChangePath` (уже есть точка входа — сейчас общий колбэк
   вызывает `UpdateTreeViewPath()` один раз для активного фрейма; нужно два
   вызова, по одному на панель).
6. **Команды**: `cm_TreeView` разделить на `cm_LeftTreeView` /
   `cm_RightTreeView` (по аналогии с уже существующими
   `cm_LeftThumbView`/`cm_RightThumbView`, `src/umaincommands.pas:2390`) +
   бинды в меню/шорткаты (`fmain.lfm` пункты `actTreeView`).
7. **Тест-кейсы**: включить только левое дерево (как сейчас) — регрессия
   нулевая; включить оба — независимая навигация; переключение вкладок внутри
   панели должно двигать её собственное дерево, не соседнее.

Объём: ~400-600 строк diff, 1 файл layout + правки в `fmain.pas`/
`umaincommands.pas`/`uglobs.pas`. Реализуемо без переписывания рендер-слоя,
можно мёржить и тестировать независимо от Фазы 2.

## Фаза 2 — нативный Cocoa рендер-слой (ForkLift-style)

LCL на macOS рисует через **Cocoa widgetset** (`lcl/interfaces/cocoa`), не через
Carbon — Carbon давно выпилен. Значит нативные `NSView`/`NSTableView`/
`NSOutlineView`/`CALayer` уже физически под капотом каждого `TListView`/
`TTreeView`/`TCustomControl`. Задача — не писать всё с нуля, а:

1. **Аудит текущей отрисовки**:
   - Список файлов — `TColumnsFileView`/`uColumnsView` (owner-draw поверх
     `TCustomDrawGrid`, не нативный `NSTableView`) — искать `OnDrawCell`/
     `AdvancedCustomDraw` в `src/ucolumnsview*.pas`.
   - Дерево — тот же паттерн, `OnAdvancedCustomDrawItem`
     (`ShellTreeViewAdvancedCustomDrawItem`, `src/fmain.pas:4367`) — уже есть
     крючок для кастомной отрисовки строк, это наш основной рычаг для стиля
     ForkLift (закруглённые plus/minus, приглушённые цвета выделения).
   - Тулбар/статусбар — стандартные LCL `TToolBar`/`TPanel`, не нативные.
2. **Vibrancy/blur сайдбара** — LCL Cocoa даёт доступ к `NSView` через
   `TCocoaWSWinControl` / `Handle` каждого компонента. Путь: получить
   `NSView` хендл `TreePanel`/`TreePanelRight`, обернуть в
   `NSVisualEffectView` (material `.sidebar`) через объект-паскаль
   Cocoa-биндинги (`lcl/interfaces/cocoa/cocoaint.pp` как референс, плюс
   `CocoaAll`/`CocoaConfig` юниты уже в дереве LCL). Это Objective-C bridging,
   пишется в units с `{$mode objfpc}{$modeswitch objectivec1}`.
3. **Закруглённые углы / тени панелей** — `CALayer.cornerRadius` /
   `shadowOpacity` на том же `NSView`, доступно из той же прослойки.
4. **Тулбар в стиле ForkLift** (сегментированный переключатель вида,
   унифицированная toolbar-титульная область) — либо кастомный `TPanel` +
   owner-draw кнопки (быстрее, кроссплатформенно деградирует), либо
   `NSToolbar` нативно (красивее, но платформо-специфично и рвёт общий UI-код
   с Windows/Linux сборками — решить, форкаем ли мы кроссплатформенность
   вообще, раз цель — только macOS).
5. **Шрифты/метрики** — заменить `gFonts[dcfMain]` дефолт на SF Pro
   (`.AppleSystemUIFont`), поднять `gIconsSize`/`RowHeight` под привычные
   ForkLift-пропорции (совпадает с `FontOptionsToFont` вызовами, уже видел в
   `UpdateShellTreeView`).
6. **Цветовая схема** — `gColors.FilePanel` (тот же record, что уже красит
   дерево в Фазе 1) свести к системным `NSColor.controlBackgroundColor` /
   `selectedContentBackgroundColor` через Cocoa-биндинг, чтобы тема сама
   переключалась light/dark вместе с macOS.

Это отдельный, открытый по объёму трек (недели, не дни) — требует Objective-C
bridging поверх LCL, которого сейчас в кодовой базе нет. Рекомендация: начать
с owner-draw полировки (пункт 1, дёшево и уже поддерживается крючками
`OnAdvancedCustomDraw*`), и только потом решать, стоит ли лезть в
`NSVisualEffectView`/`NSToolbar` — там риск сломать кроссплатформенность и
привязать форк намертво к Cocoa-интерфейсу LCL.

## Фаза 3 — сборка и релиз

- Нужен Lazarus/FPC (`lazbuild`/`fpc` сейчас не установлены на машине) —
  `brew install fpc lazarus` или сборка из `libraries/` (в дереве уже
  вендорится нужный `lazarus`/`fpcsrc`? — проверить `libraries/` перед тем
  как ставить через brew, чтобы не тянуть конфликтующую версию).
- `build.sh` — существующий build-скрипт upstream, должен работать без правок
  для Фазы 1; для Фазы 2 может понадобиться отдельный `Makefile.mac` с
  `-Fu` на новые Objective-C юниты.
- CI: GitHub Actions, matrix только `macos-latest` (кроссплатформенность
  сборки пока не форсируем, раз стиль — macOS-only).

## Статус

- [x] Форк создан: https://github.com/jablic/doublecmd-mac (upstream:
      https://github.com/doublecmd/doublecmd)
- [ ] Фаза 1 — независимое дерево
- [ ] Фаза 2 — нативный рендер-слой
- [ ] Фаза 3 — сборка/CI
