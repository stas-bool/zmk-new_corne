# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Что это за репозиторий

User-config репозиторий прошивки **ZMK** для клавиатуры **Eyelash Corne** (`eyelashperipherals,eyelash_corne`) — split-клавиатура на nrf52840 с nice_view дисплеями, энкодером, RGB underglow и эмуляцией мыши. Это форк `a741725193/zmk-new_corne`.

**НЕ совместим со стандартным `corne` (foostan crkbd)** — это другая плата со своим pinout'ом и board-определением.

## Стек (важно — не сломать при работе)

- **Ядро ZMK:** mainline `zmkfirmware/zmk main` (Zephyr 4.1, HW model v2). НЕ urob/zmk fork и НЕ cormoran fork — только официальный mainline.
- **Board:** standalone board в HW model v2 layout (`boards/eyelashperipherals/eyelash_corne/`), встроен в репо.
- **Модули urob** (zmk-helpers / adaptive-key / auto-layer / leader-key / tri-state): подключать **только по ветке `main`**, НЕ по тегам `v0.3.0` (теги написаны под старое urob-ядро, падают на mainline 2-arg `zmk_keymap_layer_activate`).

## Сборка и CI

Локальная сборка не настроена (требует полного Zephyr SDK). Основной путь — **GitHub Actions**:

- `.github/workflows/build.yml` — переиспользует reusable workflow `zmkfirmware/zmk/.github/workflows/build-user-config.yml@main`. Матрица сборок — в `build.yaml` (4 target'а: left/right с nice_view, left studio, settings_reset).
- `.github/workflows/draw.yml` — рисует диаграмму keymap (`keymap-drawer/`) через `caksoylar/keymap-drawer`. Срабатывает на push в `config/**`.

Для запуска сборки вручную: **Actions → Build ZMK firmware → Run workflow**. Артефакт — `firmware.zip` с `.uf2` файлами.

## Архитектура (big picture)

```
config/
  west.yml                 # manifest: core zmk + modules + сам board (см. ниже)
  eyelash_corne.keymap     # АКТИВНЫЙ keymap (слои, behaviors, urob helpers)
  eyelash_corne.conf       # Kconfig конфиг (RGB, backlight, pointing, BLE, NKRO)
boards/eyelashperipherals/eyelash_corne/
  board.yml                # HW v2: boards/vendor/socs/variants
  eyelash_corne.dtsi       # hardware: kscan, transform, ws2812, nice_view_spi, vbatt, flash, encoder
  eyelash_corne_{left,right}_nrf52840_zmk.{dts,defconfig}  # per-half
  Kconfig.defconfig        # ZMK_SPLIT=y (оба), ZMK_SPLIT_ROLE_CENTRAL=y (только left)
zephyr/module.yml          # board_root: .  → board берётся из этого репо через ZMK_EXTRA_MODULES
build.yaml                 # матрица CI с квалифицированными target'ами //zmk
keymap-drawer/             # сгенерированные SVG/YAML диаграммы (коммитятся автоматически)
```

**Механизм встраивания board:** `zephyr/module.yml` с `board_root: .` делает этот репо ZMK-модулем. CI передаёт `ZMK_EXTRA_MODULES=${GITHUB_WORKSPACE}`, поэтому board ищется в `boards/` этого репозитория, а не в upstream. Это заменило прежний upstream-подход, где board скачивался через west.

**Split:** левая половина = central (USB + BLE), правая = peripheral (только BLE). Роли заданы в `Kconfig.defconfig`. После перепрошивки обеих половинок требуется `settings_reset` на **обе** + re-pair (см. инструкцию ниже).

**Слои keymap:** `BASE`, `SYM`, `SYSTEM`, `GAME`, `NUM`, `ARROWS` (см. `#define` в начале keymap).

## Критичные ограничения (легко сломать)

- **HW v2 квалификатор:** все target'ы сборки обязаны использовать `//zmk` (например `eyelash_corne_left//zmk`). Plain-имя `eyelash_corne_left` резолвится в soc-qualifier `nrf52840` → stub-доска → `BOARD.dts = stub.dts` (ломает `nice_view_spi` и всё остальное).
- **`.gitignore` anchoring:** паттерны для west-клонов ДОЛЖНЫ иметь ведущий `/` (`/nice-view-mod/`, `/zmk-helpers/`). Un-anchored паттерн матчит директорию на любой глубине и **исключает board-файлы из git** — CI видит «No board named 'eyelash_corne_left' found» при наличии файлов локально. Диагностика: `git check-ignore -v <путь>`.
- **WS2812:** только через dts-binding (`worldsemi,ws2812-spi` в `.dtsi`), `CONFIG_WS2812_STRIP` удалён/не существует в Zephyr 4.1.
- **reusable workflow из внешнего репо** требует **полный 40-символьный SHA** в `uses:` (сокращённый SHA отвергается GitHub Actions).

## Сам-себе-self-reference в west.yml

В `config/west.yml` есть project `eyelash_corne` → `https://github.com/a741725193/zmk-new_corne`. Это рудимент upstream-дизайна (board скачивался отсюда). После миграции board встроен через `ZMK_EXTRA_MODULES`, поэтому этот project можно убрать — но он пока оставлен без вреда.
