# Test Plan

## Source
- Task: проверить packaged desktop app для транскрибации с diarization; primary macOS Apple Silicon, дополнительный target Windows 11 x64.
- Current build/run source: [README](../README.md#сборка-desktop-app), [npm scripts](../frontend/package.json), [Tauri config](../frontend/src-tauri/tauri.conf.json), [Python manifest](../pyproject.toml).
- Repo context: Python 3.11+ pipeline и stdlib HTTP API, React/TypeScript/Vite UI, Rust/Tauri v2 shell с app-owned backend.
- [Plans](plans.md) и [status](status.md) сохраняют прежний milestone/evidence контекст, не текущую readiness. Release/QA координация — [#78](https://github.com/MilevskyYakov/Mnema/issues/78); Windows device acceptance — [#84](https://github.com/MilevskyYakov/Mnema/issues/84).
- Last updated: 2026-09-16 (docs alignment; не новый app smoke).

## Validation Scope
- In scope: packaged Tauri app bootstrap/lifecycle, single/batch/retry/history, native file actions, transcript/job schema, ingest/media pipeline, ASR/alignment/diarization fallback paths, exporter contracts, local API, browser dev dashboard, watch-folder behavior, README-driven smoke checks.
- Out of scope: benchmark-гонка за максимальной скоростью, облачные backends, live microphone transcription, custom model training.

## Environment / Fixtures
- Data fixtures: короткие безопасные sample files для `single speaker`, `two speakers`, `video input`, `broken/unsupported file`, `watch-folder incoming copy`. В Git есть [smoke_ru.wav](../sample_data/smoke_ru.wav), [smoke_duo.wav](../sample_data/smoke_duo.wav) и [speaker manifest](../sample_data/smoke_duo_speakers.json); личные записи не использовать.
- Backend dev: Python 3.11+ и `.[dev]` в `.venv`, `ffmpeg`/`ffprobe` в PATH; [default](../configs/default.yaml) и [lightweight_test](../configs/lightweight_test.yaml) presets. Lightweight preset использует CPU/tiny и отключает diarization/alignment/summary; его PASS не закрывает их quality gates.
- Frontend: Node.js/npm и dev dependencies; Playwright browser установлен для `npm run e2e`. [Playwright config](../frontend/playwright.config.ts) запускает Vite на `127.0.0.1:5173` и может переиспользовать сервер: убедиться, что это сервер нужного checkout. Текущие [E2E tests](../frontend/e2e/dashboard.spec.ts) mock-ают API.
- macOS build: `aarch64-apple-darwin`, Rust/Cargo, Xcode Command Line Tools/SDK, Node/npm, Python 3.11, media tools в PATH и signing environment для updater artifacts.
- Windows build/runtime: нативная Windows 11 x64, `x86_64-pc-windows-msvc`, Python launcher `py -3.11`, Node/npm, Rust/MSVC, C++ Build Tools/Windows SDK, WebView2 и signing environment. Windows CI на `windows-2025` не равен device acceptance.
- Точные command chains, prerequisites и outputs — в [README](../README.md#сборка-desktop-app); keys и release gates — в [updater runbook](mac-updater.md). До запуска проверить disk/network side effects: packaging устанавливает зависимости и пишет в checkout, ASR может скачать модель, CLI пишет output/tmp, Windows `-Smoke` устанавливает и удаляет app. Использовать task-owned checkout/тестовый OS-профиль и отдельные output/temp/app data; не чистить пользовательские модели/историю.

## Test Levels

### Unit
- Загрузка и валидация YAML-конфига.
- Path/job resolution и генерация артефактных путей.
- Media probe и normalization command builder.
- Transcript cleanup и almost-verbatim guards.
- Speaker merge and mapping logic.
- Exporter formatting для `json`, `txt`, `md`, `srt`.

### Integration
- Single-file pipeline с audio input.
- Video ingest с extraction + normalization.
- Multi-speaker pipeline с diarization merge.
- Fallback без diarization.
- Fallback без alignment.
- Fallback без summary.
- DOCX/PDF generation из общей transcript model.
- API lifecycle: create job, poll status, list artifacts.
- API observability: list structured job events and inspect job log artifacts.
- App/frontend API client: bootstrap, health, jobs, transcript, artifacts, upload job, model defaults.

### End-to-End / Smoke
- Primary: текущий packaged app на целевой платформе запускается без ручного `mnema serve`; app-owned backend становится online, UI проходит сценарии ниже. Version/commit и путь artifact обязательны; старый установленный app не доказывает свежую сборку.
- Picker/native drag-and-drop создают single job; UI показывает processing/result, почти дословный transcript и сохраняет Markdown в выбранную папку.
- Batch обрабатывается последовательно; один failed item не блокирует остальные, retry относится только к нему, history/results сохраняются после restart.
- Надёжные labels допускают speaker review; низкая уверенность даёт короткие хронологические абзацы без ложных имён. Проверить minimum/normal/wide окно без horizontal overflow.
- Markdown открывается default app; reveal выделяет файл в Finder/Explorer. Проверить missing file и диагностичный backend failure/restart.
- Закрытие/восстановление окна и полный выход/повторный запуск не оставляют лишний app-owned backend; settings/models/jobs сохраняются. Notification inactive/denied paths не блокируют обработку.
- Signed update/install/restart и uninstall/reinstall — отдельные разрешённые native проверки по [runbook](mac-updater.md#manual-app-first-updater-smoke), не побочный эффект docs или build задачи.
- Supporting CLI/API/browser checks (не замена packaged UI):
- CLI `run` создаёт job directory и обязательные outputs.
- CLI `batch` продолжает работу при ошибке одного файла.
- CLI `watch` корректно обрабатывает файл после stability window.
- Service `serve` отвечает на `GET /health` и принимает `POST /jobs`.
- Platform package build создаёт ожидаемые artifacts; отдельно записывается, был ли реальный app launch. Build-only не означает smoke PASS.
- App UI renders and shows the upload/job/transcript shell.
- App UI shows processing progress and event timeline.
- README onboarding проходит без скрытых ручных шагов.

## Negative / Edge Cases
- Неподдерживаемый формат файла.
- Повреждённый media file.
- Отсутствие `ffmpeg`.
- Diarization backend unavailable.
- Summary backend unavailable.
- PDF export failure при сохранении остальных форматов.
- Низкая уверенность speaker mapping: внутренний fallback к machine label не должен превращаться в выдуманные подписи в app UI.
- Повторное появление того же файла в watch folder.

## Acceptance Gates
- [ ] `.venv/bin/python -m ruff check src tests`
- [ ] `.venv/bin/python -m mypy src`
- [ ] `.venv/bin/python -m pytest -q`
- [ ] Supporting CLI smoke на существующем fixture и service lifecycle из command matrix ниже.
- [ ] `cd frontend && npm test`
- [ ] `cd frontend && npm run build`
- [ ] `cd frontend && npm run e2e`
- [ ] Релевантные Rust gates: `cargo fmt --check`, `cargo test` из `frontend/src-tauri/` с подготовленными resources/toolchain.
- [ ] macOS: `npm run package:mac` из `frontend/`; `Mnema.app`, updater archive/signature в `src-tauri/target/aarch64-apple-darwin/release/bundle/macos/`.
- [ ] Windows: `npm run package:windows` из `frontend/`; NSIS installer/signature в `src-tauri/target/x86_64-pc-windows-msvc/release/bundle/nsis/`, runtime manifest в `.build/windows-x64/` от корня repo.
- [ ] Low-level `npm run tauri:build` при отдельной проверке Tauri — только Apple Silicon, после подготовки runtime/signing; не замена platform package.
- [ ] Packaged app-first smoke отдельно по каждой платформе; для отсутствующих prerequisites/device писать `not-run` с причиной, не ставить галочку по соседней проверке.

Каждую команду с `cd frontend && ...` выше запускают из корня repo независимо. Checklist — набор gates по затронутой поверхности, не утверждение, что они уже пройдены. Docs-only diff не требует app suites; кодовые изменения не освобождаются от native checks.

## Release / Demo Readiness
- [ ] Single-file user path работает end-to-end
- [ ] Batch path не ломается на одном ошибочном файле
- [ ] Watch-folder сценарий воспроизводим локально
- [ ] Local API отражает реальные job statuses
- [ ] Tauri app creates a single-file job and shows transcript/artifacts
- [ ] Browser dashboard remains available as dev/debug mode
- [ ] README покрывает установку, запуск и troubleshooting
- [ ] macOS clean-machine и production signing/notarization ограничения отражены явно; developer build не доказывает standalone runtime.
- [ ] Windows 11 x64 versioned checklist/go-no-go записан в #84, включая environment, installer hash, fixtures и каждый device scenario. Новый checklist/дубликат Issue здесь не создаётся.
- [ ] У каждого результата указаны commit/app version, OS/architecture, artifact path/hash для распространяемого файла, команда/ручные шаги, pass/fail/not-run и gaps; секреты/персональные данные исключены.

Исторический [macOS 0.1.1 evidence](mnema-0.1.1-integration-checklist.md) остаётся привязан к той версии и developer Mac. [#78](https://github.com/MilevskyYakov/Mnema/issues/78) и [#84](https://github.com/MilevskyYakov/Mnema/issues/84) не закрываются обновлением этого документа; текущий Windows manifest/installer не означает полный device PASS.

## Command Matrix
Целевые Python checks из корня repo, после активации `.venv`; названия соответствуют существующим tests. Полные backend gates выше сохраняются.

```sh
python -m pytest tests/test_config.py tests/test_models.py
python -m pytest tests/test_ingest.py tests/test_media.py
python -m pytest tests/test_pipeline_smoke.py tests/test_cleanup.py tests/test_speaker_mapper.py tests/test_speaker_smoothing.py
python -m pytest tests/test_export_writers.py tests/test_models.py
python -m pytest tests/test_batch.py tests/test_batch_sessions.py tests/test_service_api.py
python -m ruff check src tests
python -m mypy src
```

Supporting CLI/API smoke, только в изолированном checkout/OS-профиле с согласованной загрузкой модели и отдельными output/tmp/cache. `--out` меняет output, но не изолирует все config/model paths. Preset и CLI flags задаются [конфигом](../configs/lightweight_test.yaml) и [CLI parser](../src/mnema/cli/main.py); эти команды не исполнялись при docs alignment:

```sh
python -m mnema.cli.main --config configs/lightweight_test.yaml run sample_data/smoke_duo.wav --out ./output_smoke --speaker-manifest sample_data/smoke_duo_speakers.json --save-artifacts
python -m mnema.cli.main serve --host 127.0.0.1 --port 8765
```

`serve` — длительно работающий dev server, его запускают в отдельном терминале и останавливают после smoke. App-first smoke не использует этот вручную поднятый backend. Frontend/native build команды и side effects не дублируются: [README](../README.md#сборка-desktop-app), [acceptance gates](#acceptance-gates).

## Open Risks
- Stack уже реализован; контрактные tests с fake backends не доказывают качество реальной ASR/diarization или запуск packaged runtime.
- Тяжёлые модели могут сделать smoke-тесты слишком медленными без отдельного lightweight preset.
- PDF/DOCX экспорты могут потребовать платформенно-зависимую стабилизацию на macOS.
- macOS runtime — embedded venv с base-interpreter/fallback assumptions, Windows — PyInstaller runtime. Переносимость проверяется на целевой чистой среде, не выводится из наличия artifacts.
- Windows scripted smoke проверяет app bootstrap и API/filesystem, но не весь UI/updater/device контракт #84.

## Deferred Coverage
- Performance/regression benchmarks на длинных файлах.
- Полная native/device автоматизация сверх существующих packaging scripts; сами packaging и app-first gates не отложены.
- Полная автоматизация качества summary beyond structural validity.
