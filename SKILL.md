name: dota-surgical-vpk-optimizer
description: Оптимизация и кастомизация VPK-архивов Dota 2 для ArdysaModsTools. Собирает безопасный filtered VPK из оригинального архива по vpk_mod_config.json, оставляя только активных героев и нужные shared-активы.
---

# Dota 2 Surgical VPK Optimizer (Hero Purge)

Этот навык предназначен для создания оптимизированных VPK-файлов для Dota 2, совместимых с ArdysaModsTools. Он оставляет в паке только героев из `vpk_mod_config.json`, но сохраняет системные и shared-активы, чтобы не ломать окружение, HUD, звуки и специальные модели.

Главный принцип текущей версии: **не удалять из оригинального экстракта**, а строить новое чистое дерево через copy-filter и уже его паковать в VPK. Это обходит проблемы с ACL после распаковки VPKEdit и снижает риск случайно испортить исходник.

## Основной процесс (Workflow)

1. **Подготовка:** держите `pak01_dir_original.vpk` как эталонный исходник, а `vpk_mod_config.json` как единственный источник правды по активным героям.
   - `active_roster` — единственный список героев, чьи hero/item/material/particle ассеты должны попасть в итоговый VPK.
   - `compat_keep_roster` — только временный диагностический механизм; в strict lightweight-сборке выключенные герои не должны сохраняться через него.
2. **Распаковка:** используйте `vpkeditcli` для полной распаковки оригинального архива в отдельную временную папку.
3. **Filtered tree:** запустите `scripts/build_filtered_tree.ps1`, чтобы скопировать в новое дерево только разрешённые файлы.
4. **Упаковка:** соберите filtered tree обратно в VPK через `vpkeditcli`.
5. **Проверка:** выполните `vpkeditcli --verify-checksums all` для собранного VPK.
6. **Замена:** сохраните старый `pak01_dir.vpk` в backup и замените его новым.
7. **Установка:** следуйте инструкции по установке через ArdysaModsTools.

## Основная команда фильтрации

```powershell
<project>\dota-surgical-vpk-optimizer\scripts\build_filtered_tree.ps1 `
  -SourceRoot <build>\source\pak01_dir_original `
  -DestinationRoot <build>\filtered\pak01_dir `
  -ConfigPath <project>\vpk_mod_config.json
```

`scripts/optimize.ps1` оставлен как вспомогательный инструмент для зачистки папки на месте, но основной безопасный путь для новых сборок - `build_filtered_tree.ps1`.

## Иерархия проекта и VPK

### Структура рабочего каталога (Project Root)
```text
VPK_Project/
├── pak01_dir_original.vpk   # Оригинальный файл (Бэкап)
├── pak01_dir.vpk            # Твой готовый оптимизированный файл
├── vpk_mod_config.json      # Список активных и удаленных героев
├── REPACK_PLAN.md           # План и текущий статус пересборки
├── REPACK_LOG.md            # Хронология действий
└── build_repack_*/          # Рабочая папка пересборки
    ├── source/              # Распакованный оригинал
    ├── filtered/pak01_dir/  # Чистое дерево для упаковки
    └── pak01_dir_rebuilt.vpk
```

### Внутренняя структура VPK (После оптимизации)
```text
root/
├── panorama/                # HUD, Интерфейс (ОСТАВЛЯЕМ)
├── scripts/                 # Логика игры (ОСТАВЛЯЕМ)
├── maps/                    # Карта/Ландшафт (ОСТАВЛЯЕМ)
├── models/
│   ├── heroes/              # Только папки выбранных героев
│   └── items/               # Только вещи выбранных героев (Арканы)
├── materials/
│   └── models/
│       ├── heroes/          # Текстуры героев
│       └── items/           # Текстуры предметов
├── particles/               # Эффекты (Только нужные герои + общие)
└── sounds/                  # Озвучка и звуки способностей
```

## Важные правила

Чтобы моды на окружение продолжали работать, не фильтруйте следующие папки из корня архива:
- `panorama` (Интерфейс, HUD)
- `scripts` (Логика игры и модов)
- `maps` (Ландшафт)
- `music` / `soundevents` (Звуковые моды)
- `models/props_structures` (Кастомные вышки)
- `materials/environment` (Текстуры карты)

Фильтрация применяется только к hero/item/material/particle/sound областям, где лежат ассеты конкретных героев.

## Алиасы героев

Скрипт учитывает несовпадения имён папок и конфигов:

- `wisp` / `io` / `wips`
- `sniper` / `kardel`
- `phantom_lancer` / `phantomlancer`
- `bounty_hunter` / `bountyhunter` / `gondar`
- спец-активы Io: `portal`, `cube`, `companion`

Короткие алиасы сравниваются через нормализованные имена папок, а не через простое `-match`, чтобы не цеплять чужие имена вроде `lion`, `furion`, `legion_commander`.

## Strict Lightweight и выключенные герои

Если в паке сохраняется общий `scripts/items/items_game.txt`, нельзя бездумно удалять зависимости и ожидать, что обычные сеты неактивных героев всегда будут работать: записи предметов могут ссылаться на модовые `models/heroes`, `materials/models/items`, `particles/units/heroes` или `kisilev_ind` пути.

Важно: по strict lightweight-правилу, если героя нет в `active_roster`, его ассеты не должны попадать в итоговый VPK. Не исправляйте баги выключенных героев добавлением их в `compat_keep_roster`, если пользователь явно хочет, чтобы выключенных героев не было в VPK.

Перед агрессивной очисткой всегда проверяйте, сохраняется ли `scripts/items/items_game.txt`. Если он сохраняется целиком, удаление hero/item/material/particle зависимостей неактивных героев может поломать их обычные сеты. Правильный фикс для выключенных героев — чистить/адаптировать их записи в `items_game.txt`, а не возвращать их ассеты в VPK.

Рекомендуемое будущее улучшение: добавить отдельный pass для `items_game.txt`, который для героев вне `active_roster` точечно чистит модовые `model_player`, `asset_modifier`, `particle`, `modifier`, `image_inventory`, `visuals` и похожие поля. Dependency-preserve нужен для активных героев, чтобы не потерять их арканы/имморталки.

Критичное ограничение: не заменяйте item-блоки в `items_game.txt` целиком. Econ schema содержит cross-reference связи между item id, `event_id`, `effects_item_def`, bundles, styles и эффектами. Уже были проваленные тесты: sanitize-сборки падали при запуске Dota на `EVENT_ID_WINTER_MAJOR_2016`, потому что схема не находила `effects_item_def 16844`.

## Lessons Learned

В этом проекте уже проверены и признаны нерабочими такие подходы:

- Добавить Lion/Bounty Hunter в `active_roster` или `compat_keep_roster`. Это возвращает ассеты выключенных героев в VPK и нарушает strict lightweight-правило пользователя.
- Массово заменить item-блоки выключенных героев дефолтными блоками из Dota. Итог: fatal error при запуске, `EVENT_ID_WINTER_MAJOR_2016` / `Unable to find effects_item_def 16844`.
- Заменить только один дефолтный item-блок Lion (`572`). Итог: запуск не падал, но сеты не починились полностью; оставались модовые иконки/эффекты и отсутствующие модели.
- Заменить целиком все найденные Lion блоки (`365`, `366`, `369`, `572`) и Bounty Hunter back (`52`). Итог: снова fatal error `effects_item_def 16844`.

Практический вывод для будущих агентов: **никаких full-block replacements в `items_game.txt` для production VPK**. Даже маленькая замена может нарушить event/econ schema или скрытые связи.

Следующий безопасный алгоритм:

1. Работать по одному герою и по одному item-блоку.
2. Сначала сделать offline-анализ: какие модовые пути есть в блоке и есть ли соответствующие файлы в итоговом VPK.
3. Не менять id, имя блока, `event_id`, bundle/style/reward/effects секции и порядок больших секций.
4. Точечно заменить только строки путей на дефолтные значения или удалить только конкретный модовый `asset_modifier`, если он ссылается на отсутствующий файл.
5. Собрать отдельный тестовый VPK, проверить `vpkeditcli --verify-checksums all`, затем запуск Dota.
6. Если запуск падает, сразу откатить рабочий `pak01_dir.vpk` на последний стабильный backup.

## Bundled Resources

- **Основной скрипт сборки дерева:** `scripts/build_filtered_tree.ps1` — копирует только разрешённые файлы из оригинального экстракта.
- **Скрипт зачистки на месте:** `scripts/optimize.ps1` — вспомогательный инструмент, когда нужно чистить уже существующую папку.
- **Экспериментальный sanitize-прототип:** `scripts/sanitize_items_game.py` — не использовать для production VPK; full-block replacement уже ломал Dota.
- **Reference analyzer:** `scripts/analyze_items_game_refs.py` — вспомогательный отчёт для исследования item-блоков, не доказательство безопасности замены блока целиком.
- **Инструкция по установке:** `references/installation.md` — как правильно добавить VPK в игру.
- **Структура VPK:** `references/vpk_structure.md` — справочник по папкам Dota 2.

## Ссылки
- [Документация ArdysaModsTools](https://github.com/Anneardysa/ArdysaModsTools/tree/main/docs)
- [Спецификация структуры модов](https://github.com/Anneardysa/ArdysaModsTools/blob/main/docs/developer/api/mod-file-structure.md)
- [Discord (Ardysa Mods)](https://discord.gg/GXuhAwte) — здесь можно найти оригинальные VPK-файлы в канале #update-mods.
