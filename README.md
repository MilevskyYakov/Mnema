# Mnema desktop app

Проект предназначен для локальной обработки аудио- и видеофайлов с получением почти дословного транскрипта, diarization, summary и экспортов в несколько форматов. Основная пользовательская платформа — macOS Apple Silicon; дополнительный packaging target — Windows 11 x64. Полная Windows app-first QA ведётся отдельно в [#84](https://github.com/MilevskyYakov/Mnema/issues/84).

## Download

macOS Apple Silicon и Windows x64 artifacts:

- [Download the latest release](https://github.com/MilevskyYakov/Mnema/releases/latest)
- Direct updater feed: `https://github.com/MilevskyYakov/Mnema/releases/latest/download/latest.json`

Пользовательский путь — packaged desktop app, который сам запускает local backend и хранит ASR-модели, настройки, результаты и историю в app data. Наличие release artifact или installer не означает, что все native/device сценарии проверены.

Current distribution note: builds are signed for the Tauri updater, but public macOS code signing/notarization may still require an extra Gatekeeper confirmation on first launch.

Privacy note: do not upload private audio/video, transcripts, API keys, or updater signing keys to public GitHub Issues. See `SECURITY.md`.

## Что делает проект

Система принимает медиафайл или набор файлов, извлекает и нормализует аудио, распознаёт речь, определяет смены спикеров, собирает структурированный transcript и сохраняет результат в человекочитаемом и техническом виде.

Главная версия продукта — packaged desktop-приложение на Tauri с собственным local backend. Поддерживающие режимы работы:
- один файл;
- список файлов;
- директория;
- watch folder;
- локальный mini-service API;
- browser dashboard для разработки и диагностики.

## Цели

- локальная обработка без облачных API;
- приоритет качества;
- пригодность для личного использования через desktop app;
- app-first архитектура: UI, local API и pipeline разделены, но пользовательский путь проектируется вокруг Tauri app;
- JSON-совместимость для дальнейшей автоматизации.

## Планируемые форматы входа

### Audio
- mp3
- wav
- m4a
- aac
- flac
- ogg

### Video
- mp4
- mov
- mkv
- avi
- webm

## Форматы выхода

- txt
- md
- docx
- pdf
- srt
- json

Во время обработки приложение создаёт промежуточные артефакты:
extracted/normalized audio, raw ASR payload, diarization dump, logs/events и
config snapshot. После успешного завершения job эти internal/session файлы
автоматически удаляются. Постоянно сохраняются только пользовательские результаты
и компактные данные, нужные для app history, просмотра transcript и повторного
сохранения результата: final markdown, выбранные exports, `segments.json`,
`words.json`, summary и `job.json`.

Failed jobs могут временно сохранять диагностический минимум. Старые failed/orphan
temp files удаляются retention cleanup'ом. Локальные ASR-модели хранятся в
durable app data/cache directory и не попадают под cleanup временных job-файлов.

## Архитектурная идея

Pipeline:

`input -> ffmpeg normalization -> ASR -> alignment -> diarization -> speaker merge -> conservative cleanup -> summary -> export`

Ключевая идея: не строить систему вокруг одной “магической” модели, а разделять проект на независимые слои, чтобы можно было менять backend без переписывания всего приложения.

## Основные свойства

- local-first;
- canonical Tauri desktop app;
- local mini-service API как backend приложения;
- CLI и browser dashboard как поддерживающие/dev-интерфейсы;
- batch processing;
- watch folder;
- almost-verbatim transcript;
- diarization;
- summary;
- stable JSON schema;
- graceful degradation.

## Transcript policy

Итоговый transcript должен быть почти дословным:
- не переписывать содержание литературно;
- не удалять слова-паразиты по умолчанию;
- не менять смысл;
- допускать только мягкую нормализацию пробелов, пунктуации и сегментов.

Хранить два представления:
- `text_raw`
- `text_clean`

## Speaker policy

Система должна:
- автоматически выполнять diarization;
- поддерживать передачу expected speaker names заранее;
- сопоставлять имена только при достаточной уверенности;
- не выдумывать сопоставление.

## Пример структуры проекта

```text
project-root/
  task.md
  README.md
  decisions.md
  acceptance_checklist.md
  configs/
  frontend/
  src/
  tests/
  output/
  tmp/
```

## Пример CLI

### Один файл
```bash
mnema run input.mp4 --out ./output
```

### Несколько файлов
```bash
mnema batch ./a.mp4 ./b.mp3 --out ./output
```

### Папка
```bash
mnema dir ./incoming --out ./output
```

### Watch folder
```bash
mnema watch ./incoming --out ./output
```

### Локальный сервис
```bash
mnema serve --host 127.0.0.1 --port 8765
```

## App и frontend

Главная пользовательская поверхность — Tauri v2 desktop app (Rust shell). Он сам запускает Python 3.11+ backend на свободном `127.0.0.1` порту и использует React/TypeScript/Vite UI поверх local service API на stdlib `http.server`. Источники stack: [pyproject.toml](pyproject.toml), [frontend/package.json](frontend/package.json), [Cargo.toml](frontend/src-tauri/Cargo.toml), [HTTP server](src/mnema/service/server.py).

Browser frontend остаётся dev/debug режимом поверх `mnema serve`. Он тонкий: не запускает speech pipeline напрямую, а работает через локальный API и общую job-state модель.

Первый экран приложения:
- добавление одного или нескольких media-файлов через picker или drag-and-drop;
- последовательная настройка batch, фоновые jobs, item-only retry и grouped history;
- выбор папки результата глобально, для batch или отдельной job;
- история, поиск и отдельные состояния processing/result/error;
- надёжные speaker labels либо хронологический transcript без ложных имён;
- macOS notifications для inactive app: single terminal event, batch errors и один summary.

Архитектурное ограничение: UI обслуживает canonical desktop app и не должен зависеть от облачных API, внешней авторизации, публичных URL или прямого доступа к Python internals.

## Спикеры

Desktop app определяет спикеров автоматически. При надёжном разделении результат предлагает
проверить подписи; при низкой уверенности сохраняет хронологический transcript без ложных labels.
Предварительное поле участников в основном app flow отсутствует.

Для продвинутых CLI-сценариев всё ещё поддерживается JSON-файл через `--speaker-manifest`.

```json
{
  "expected_speakers": [
    { "name": "Алексей", "role": "Интервьюер" },
    { "name": "Марина", "role": "Кандидат" }
  ]
}
```

## Пример выходных файлов на один job

```text
output/<job_id>/
  job.json
  segments.json
  words.json
  final_speech_text.md
  transcript_clean.txt
  transcript_clean.md
  transcript_clean.docx
  transcript_clean.pdf
  subtitles.srt
  summary.md
  summary.json
  artifacts/        # только временные diagnostics во время processing/failed retention
```

## Среда разработки

Ниже — backend/browser dev setup на macOS Apple Silicon, не установка готового приложения. Platform packaging описан [отдельно](#сборка-desktop-app).

На macOS не используйте системный `/usr/bin/python3` для команд проекта: он
может быть Python 3.9 и не соответствует `pyproject.toml`. В репозитории есть
`.python-version` со значением `3.11`; используйте Homebrew/uv/pyenv Python 3.11
и проектный virtualenv.

Для backend dev нужны:
- `ffmpeg` и `ffprobe` в PATH;
- установленные Python-зависимости;
- конфиг через YAML;
- локальное хранение временных и выходных файлов.

### Backend

```bash
uv venv --python python3.11 .venv
uv pip install -e ".[dev]"
.venv/bin/python -m mnema.cli.main --help
```

`uv pip` здесь намеренно используется вместо `.venv/bin/python -m pip`: `uv venv`
может создать окружение без установленного `pip`, но `uv pip install ...` всё равно
ставит зависимости в проектный `.venv`.

Для обычной работы можно активировать окружение один раз:

```bash
source .venv/bin/activate
python --version  # должно быть 3.11+
pytest
```

### Browser dashboard для разработки

```bash
cd frontend
npm install
npm run dev
```

### Локальный запуск browser dashboard

В одном терминале:

```bash
.venv/bin/python -m mnema.cli.main serve --host 127.0.0.1 --port 8765
```

Во втором терминале:

```bash
cd frontend
npm run dev -- --host 127.0.0.1 --port 5173
```

После этого dev dashboard доступен на `http://127.0.0.1:5173/`.

## Сборка desktop app

Команды npm выполняются из `frontend/`; их точные определения — в [package.json](frontend/package.json). Системные SDK/toolchain prerequisites — в [официальной документации Tauri](https://v2.tauri.app/start/prerequisites/); targets и runtime layout Mnema задаются скриптами ниже.

- `npm run build` — TypeScript check и Vite build в `frontend/dist`; без Rust, sidecars и установки.
- `npm run tauri:dev` — dev shell с Vite; требует native toolchain и доступного backend/runtime, не проверяет установленный bundle.
- `npm run tauri:build` — low-level `tauri build --target aarch64-apple-darwin`, не универсальная Windows-команда. [Tauri config](frontend/src-tauri/tauri.conf.json) запускает frontend build, подключает уже подготовленные sidecars/resources и включает updater artifacts. Команда сама не подготавливает Python и media tools; для свежего полного bundle используйте platform packaging.
- `npm run package:mac` / `npm run package:windows` — подготовка platform runtime и Tauri bundle. Обе команды пишут generated resources/build outputs в checkout и скачивают зависимости; они не устанавливают приложение и не публикуют release без дополнительных действий.
- `npm run install:local` — macOS build, замена установленного приложения и запуск. Windows `package:windows -- -Smoke` тоже выполняет установку, запуск и удаление; это не build-only проверка.

### macOS Apple Silicon

Target: `aarch64-apple-darwin`. Нужны Node.js/npm с frontend dev dependencies, Rust/Cargo для Apple Silicon, Xcode Command Line Tools/SDK, Python 3.11 (по умолчанию `python3.11`, override `PYTHON_BIN`), `ffmpeg` и `ffprobe` в PATH. При текущем `createUpdaterArtifacts=true` полный подписанный build требует `TAURI_SIGNING_PRIVATE_KEY` и пароль для защищённого ключа; [signing prerequisites](docs/mac-updater.md#signing-keys-and-secret-boundary). Не подставляйте ключи в репозиторий или чат.

Dev shell после подготовки runtime скриптами `setup-tauri-prereqs.sh` и `build-embedded-runtime.sh` из цепочки ниже (или после полного `package:mac`):

```bash
cd frontend
npm ci --include=dev
npm run tauri:dev
```

Полная bundle-сборка после подготовки toolchain/signing environment:

```bash
cd frontend
npm ci --include=dev
npm run package:mac
```

Цепочка: [package-mac.sh](scripts/package-mac.sh) вызывает [setup-tauri-prereqs.sh](scripts/setup-tauri-prereqs.sh) (Rust/npm и host target), затем [build-embedded-runtime.sh](scripts/build-embedded-runtime.sh) (Python venv в `frontend/src-tauri/resources/python`, config copy, Python dependencies, `ffmpeg`/`ffprobe` из PATH), затем `tauri:build`.

Результат: `frontend/src-tauri/target/aarch64-apple-darwin/release/bundle/macos/Mnema.app`, а при успешном signing — `Mnema.app.tar.gz` и `Mnema.app.tar.gz.sig` рядом. Это не установка в `/Applications` и не публикация.

Embedded venv не является доказательством переносимого standalone Python: [macOS launcher](frontend/src-tauri/binaries/mnema-backend-aarch64-apple-darwin) использует base interpreter из `pyvenv.cfg`, затем fallback paths. Media tools копируются из PATH без отдельного аудита их динамических библиотек. Поэтому developer-Mac build/launch не закрывает clean-machine gate; его проверяют на конкретном artifact, не обещают по слову «embedded».

### Windows 11 x64 desktop runtime

Target: `x86_64-pc-windows-msvc`, сборка нативно на Windows 11 x64, не cross-build с Mac. Нужны Python 3.11 с launcher `py -3.11`, Rust/Cargo с MSVC target, Node.js/npm (CI использует Node 22), Visual Studio Build Tools с C++/Windows SDK и WebView2 для app runtime. Скрипт сам выполняет `npm ci --include=dev`. Подготовьте signing environment по [release runbook](docs/mac-updater.md), затем запустите из PowerShell:

```powershell
cd frontend
npm run package:windows
```

[package-windows.ps1](scripts/package-windows.ps1) собирает standalone `mnema-backend.exe` через pinned PyInstaller,
скачивает pinned FFmpeg/FFprobe archive с обязательной SHA-256 проверкой и
создаёт один Tauri-native NSIS installer и подписанный updater artifact. Нужен
`TAURI_SIGNING_PRIVATE_KEY`; private key остаётся вне repository. Installer,
его `.sig`, sidecars и `mnema.exe` записываются в manifest с размерами и SHA-256:
`.build/windows-x64/runtime-manifest.json`. Sidecars подготавливаются в `frontend/src-tauri/binaries/*-x86_64-pc-windows-msvc.exe`; `mnema.exe` и runtime — в `frontend/src-tauri/target/x86_64-pc-windows-msvc/release/`, installer `*-setup.exe` и `.sig` — в его `bundle/nsis/`.

Опциональный автоматизированный native smoke запускается только в разрешённой disposable Windows-среде без существующей установки Mnema. Он делает silent install в `%LOCALAPPDATA%\Mnema`, запускает приложение, проверяет app-owned backend/health/media tools, скачивает tiny Whisper и создаёт job через API с Unicode-путями. Затем повторно устанавливает installer, проверяет запуск backend и наличие файлов job/settings/Markdown после upgrade/uninstall. Model files проверяются после download; их доступность в UI после upgrade этим скриптом не доказана.

```powershell
npm run package:windows -- -Smoke
```

Без `-PreviousInstallerUrl` это reinstall той же версии; с этим параметром — upgrade с разрешённого GitHub release URL. Скрипт создаёт app data и временные fixtures, останавливает запущенное им приложение и удаляет установку; не запускайте его на пользовательских данных. CI callers: [runtime workflow](.github/workflows/windows-runtime.yml) и [protected release workflow](.github/workflows/windows-release.yml). CI build/API smoke, в том числе на `windows-2025`, не заменяет [Windows 11 x64 device QA #84](https://github.com/MilevskyYakov/Mnema/issues/84): picker/drop, single/batch UI, retry, Explorer, layout, updater UI и durable data после restart.

### Установка macOS bundle

`package:mac` только создаёт bundle в репозитории. Уже установленное приложение
в `/Applications/Mnema.app` после изменений в коде само не обновляется:
для обычного локального обновления установленного `.app` используйте:

```bash
cd frontend
npm run install:local
```

`install:local` сначала выполняет `package:mac`, затем безопасно заменяет
`/Applications/Mnema.app`, снимает quarantine metadata best-effort и
открывает приложение. Если уже запущен старый app, команда остановится с явным
сообщением; чтобы автоматически закрыть его перед заменой:

```bash
npm run install:local -- --quit-running
```

Источник — [install-local-app.sh](scripts/install-local-app.sh). Без release key он временно подставляет локальный updater key в config и восстанавливает исходный config после сборки; такой bundle не доказывает доверие к production feed. Команда также чинит Python symlinks и останавливает найденные orphaned backends. Это изменение локальной установки, не проверка документации.

Для повторной установки уже собранного bundle из корня репозитория:

```bash
./scripts/install-local-app.sh --no-build --no-open
```

`--no-build` допустим только после проверки, что bundle соответствует нужным commit/version; наличие старого `.app` не подтверждает свежую сборку.

Полезные флаги: `--no-open` для automation, `--install-dir DIR` для установки не
в `/Applications`, `--help` для справки.

### Signed in-app updates

Packaged macOS and Windows apps use the same Tauri v2 signed updater. In the installed app,
use the sidebar card “Обновление” to check the configured release endpoint,
show no-update/update/error states, download a signed update, and install it.
After install the app asks for a restart so the new version opens cleanly.

Default release endpoint: GitHub Releases static `latest.json` at
`https://github.com/MilevskyYakov/Mnema/releases/latest/download/latest.json`.
Updater artifacts are generated during Tauri build when the release environment
provides `TAURI_SIGNING_PRIVATE_KEY` (and optional
`TAURI_SIGNING_PRIVATE_KEY_PASSWORD`). The private key must stay outside the repo;
only the public updater key is committed in `frontend/src-tauri/tauri.conf.json`.
Full multi-platform release/update runbook: [docs/mac-updater.md](docs/mac-updater.md). Сборка не разрешает публикацию feed, rollout или смену ключей.

В desktop-режиме приложение само запускает локальный backend на свободном
`127.0.0.1` порту. Runtime data хранится в app data (`~/Library/Application Support/local.mnema` на macOS, `%APPDATA%\local.mnema` на Windows):
`output`, `tmp`, `cache` и durable ASR-модели не зависят от папки репозитория.

Локальные ASR-модели в packaged desktop app — постоянные данные приложения, а
не disposable cache. Backend sidecar получает `--app-data-dir`, после чего
использует `MNEMA_MODEL_DIR=<app_data_dir>/models`: Whisper-файлы лежат
в `<app_data_dir>/models/whisper`, external/ONNX ASR runtime — в
`<app_data_dir>/models/external`. При локальном upgrade/replacement через
`npm run install:local` не удаляйте macOS Application Support; модель должна
оставаться `ready` в `/models` и в UI без повторного скачивания. Старые валидные
модели из `<app_data_dir>/cache/whisper`, `<app_data_dir>/cache/mnema/models`
или пользовательского `~/.cache` копируются в canonical model dir при проверке
статуса; повреждённые/недокачанные файлы остаются `corrupt` и не считаются
готовыми.

## Что должно быть в финальной реализации

- Tauri desktop app как главная версия продукта;
- CLI;
- mini-service;
- browser dashboard для разработки и диагностики;
- watch folder;
- batch processing;
- quality-first pipeline;
- экспорт во все заявленные форматы;
- README с инструкциями запуска;
- тесты;
- documented JSON schema.

## Проверка

```bash
.venv/bin/python -m pytest -q
.venv/bin/python -m ruff check src tests
.venv/bin/python -m mypy src
cd frontend && npm test
npm run build
npm run e2e
```

Последние команды выполняются после одного перехода в `frontend/`. `npm run e2e` — Playwright browser checks с mocked API, не native app smoke. Для low-level `npm run tauri:build` нужны уже подготовленные resources/signing; полный platform build и outputs описаны [выше](#сборка-desktop-app).

App-first acceptance требует запуска текущего packaged artifact и проверки UI вместе с его backend; build, CLI/API и browser PASS записываются отдельно. [Test plan](docs/test-plan.md) задаёт desktop smoke и native gates. Для docs-only diff достаточно сверки команд/ссылок/allowlist и `git diff --check`; app suites не запускаются только ради текста.

Если `.venv` активирован, первые три команды можно запускать короче:

```bash
pytest -q
ruff check src tests
mypy src
```

## Ограничения

В MVP не требуется:
- публичный интернет-сервис;
- live microphone transcription;
- collaborative editing;
- custom model training.

## Диагностика и отказоустойчивость

Если отдельный этап упал, проект должен по возможности завершать job частично:
- без diarization;
- без alignment;
- без summary;
- без PDF.

Остальные результаты должны сохраняться, если это возможно.

## Статус проекта

README описывает текущий build/run путь, а не стартовый blueprint. Продуктовые требования — [task.md](task.md), решения — [decisions.md](decisions.md), критерии — [acceptance_checklist.md](acceptance_checklist.md), актуальные проверки — [test plan](docs/test-plan.md). Старые milestone/status записи сохраняют исторический контекст, но не задают текущую очередь.

macOS и Windows readiness учитываются отдельно: [эпик #78](https://github.com/MilevskyYakov/Mnema/issues/78) координирует release/QA, [#84](https://github.com/MilevskyYakov/Mnema/issues/84) остаётся владельцем полного Windows 11 x64 app-first прохода. Исторический [macOS 0.1.1 smoke](docs/mnema-0.1.1-integration-checklist.md) не доказывает readiness новой версии или чистой машины. Docs-only изменение не закрывает эти gates.
