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

- `brew install fpc lazarus` **не работает на текущей macOS** — cask `lazarus`
  деприкейтед, его зависимости `fpc-laz`/`fpc-src-laz` — старые `.pkg` без
  подписи (`fpc-src-laz` прямо падает: "incompatible with this version of
  macOS"). `fpc` формула (без cask) ставится нормально через brew.
- Рабочий обход — сборка `lazbuild` из исходников, без sudo/pkg:
  ```
  git clone --depth 1 https://gitlab.com/freepascal.org/lazarus/lazarus.git
  git clone --depth 1 --branch release_3_2_2 https://gitlab.com/freepascal.org/fpc/source.git fpcsrc
  cd lazarus && make lazbuild FPC=$(which fpc) FPCDIR=/path/to/fpcsrc
  ```
  `FPCDIR` обязателен — без него Makefile падает на дефолтный
  `/usr/local/lib/fpc/<ver>`, где нет исходников RTL/FCL, и компиляция
  `fcllaz.pas`/`ideproject` падает на "Can't find unit DB".
- После первой сборки `lazbuild` пропиши `FPCSourceDirectory` в
  `environmentoptions.xml` (primary config path, `--pcp=`), иначе повторные
  сборки LCL-пакетов (`LazControls`, `DateTimeCtrls` и т.п.) через сам
  `lazbuild` увидят чужой checksum RTL и не найдут `controls.pp`/`lcltype.pp`
  в путях компиляции — надёжнее всего один раз стереть все `*.ppu`/`*.o`/
  `*.compiled` в дереве lazarus и пересобрать всё **одним** проходом
  `lazbuild` (не смешивать `make`- и `lazbuild`-собранные юниты).
- Сборка компонентов DC: `components/build.sh` (нужны `$lazbuild`,
  `$DC_ARCH=--widgetset=cocoa` в окружении) — обязателен **до**
  `src/doublecmd.lpi`, иначе `lazbuild` падает на "Broken dependency: SynUni".
- **Известный баг линковки**: FPC 3.2.2 + Cocoa widgetset + новый Xcode
  linker (`ld-prime`, macOS Tahoe+) даёт `ld: malformed method list atom
  'ltmp5' ... fixups found beyond the number of method entries` при линковке
  `.o` с Objective-C методлистами (`cocoawsextctrls.o` и подобные). Фикс —
  форсировать классический линкер: `lazbuild --opt="-k-ld_classic" ...`.
  Актуально и для Фазы 2 (там будет ещё больше ObjC-биндингов) — держать этот
  флаг в build-скрипте постоянно, не разово.
- `build.sh` — существующий build-скрипт upstream, работает для Фазы 1 при
  экспорте `lazbuild`/`DC_ARCH` как выше плюс `--opt="-k-ld_classic"` (сейчас
  сам `build.sh` этот флаг не прокидывает — нужно будет добавить).
- CI: GitHub Actions, matrix только `macos-latest` (кроссплатформенность
  сборки пока не форсируем, раз стиль — macOS-only). Из-за деприкейтед-каска
  CI тоже не сможет ставить Lazarus через `brew install --cask lazarus` —
  придётся собирать `lazbuild` из исходников тем же способом, что и здесь.
- **Запускать нужно только через `.app`-бандл, не голый бинарь.** Прямой
  запуск `./doublecmd` (сырой Mach-O без `Info.plist`) на macOS даёт три
  сломанных вещи разом: (1) нет menu bar сверху — Cocoa не регистрирует
  global-menu для процесса без bundle identity; (2) resize/zoom title-bar
  кнопки либо не работают, либо визуально глючат; (3) иконка тулбар-кнопки
  может рендериться как "?" из-за stale `icon-theme.cache`, который
  инвалидируется только по mtime `pixmaps/dctheme/` (верхний уровень), а не
  вложенных `16x16/actions`/`32x32/actions`, где реально лежат файлы —
  добавил новую иконку → cache её не увидит → нужно чистить
  `pixmaps/dctheme/icon-theme.cache`.
  Рабочий минимальный bundle для локальных прогонов (репо уже содержит
  скелет `doublecmd.app/Contents/{Info.plist,PkgInfo,Resources/*.icns}`,
  нужно только доложить бинарь и рантайм-ресурсы, как в
  `install/darwin/install.sh`, но без плагинов):
  ```
  DC_MACOS=doublecmd.app/Contents/MacOS
  mkdir -p "$DC_MACOS"
  cp doublecmd "$DC_MACOS/"                       # НЕ symlink — codesign требует regular file
  cp -r default language pixmaps highlighters "$DC_MACOS/"
  codesign --force --deep --sign - doublecmd.app  # ad-hoc, достаточно для локального теста
  open doublecmd.app                              # НЕ ./doublecmd напрямую
  ```
  `codesign` явно отказывается подписывать bundle, если
  `Contents/MacOS/doublecmd` — symlink (готовый скелет в репо именно так и
  устроен, `git ls-files` покажет symlink на `../../../doublecmd` — под
  codesign это надо заменить на copy).
- **PopulateWithBaseFiles блокирует UI-поток на старте**, если один из
  примонтированных томов медленный (у меня — `SASHA_2`, `fskit`-драйвер,
  не kernel-native). Дерево (`TShellTreeView.PopulateWithBaseFiles`,
  вызывается из `UpdateShellTreeView` в `FormCreate` синхронно) рекурсивно
  бьёт `opendir`/`FindFirst` по каждому тому чтобы решить, рисовать ли
  `+`/`-` у узла — при медленном FS это секунды-десятки секунд подвисания
  при **каждом** старте, если дерево включено (`gSeparateTreeLeft`/
  `gSeparateTreeRight` = True в конфиге). Не регрессия дуал-дерева
  конкретно — тот же код блокировал бы и одиночное дерево в апстриме на
  таком же томе — но теперь заметнее, т.к. дерево двойное и его физически
  проще включить кнопкой. Не чинил (нужен async I/O в это место, отдельная
  задача) — если дерево при старте "виснет", проверить `mount` на
  fskit/network-тома первым делом.

## Статус

- [x] Форк создан: https://github.com/jablic/doublecmd-mac (upstream:
      https://github.com/doublecmd/doublecmd)
- [x] Фаза 1 — независимое дерево (код + **компилируется и линкуется чисто**,
      `aarch64-darwin-cocoa`, проверено локальной сборкой из исходников)
- [ ] Фаза 2 — нативный рендер-слой
- [ ] Фаза 3 — сборка/CI (сборка вручную работает и задокументирована выше;
      GitHub Actions workflow ещё не написан)
