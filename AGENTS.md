# AGENTS.md

## Project Direction

- Canonical product: Tauri desktop app in `frontend/src-tauri`.
- Primary user surface: packaged macOS app that starts the local backend itself.
- Additional packaging target: Windows 11 x64; device-level readiness belongs to [#84](https://github.com/MilevskyYakov/Mnema/issues/84), coordinated by [#78](https://github.com/MilevskyYakov/Mnema/issues/78). Build success is not device QA.
- Supporting surfaces: CLI, local service API, and browser dashboard are development, automation, and debugging interfaces.
- Current stack: Python 3.11+ pipeline and stdlib local HTTP API, React/TypeScript/Vite UI, Rust/Tauri v2 desktop shell.

## App-First Rule

- New user-facing behavior must be designed and verified through the desktop app flow first.
- Do not treat CLI or API convenience as the main product path when it conflicts with app UX.
- Browser dashboard changes must stay compatible with Tauri runtime assumptions: local API base, no cloud auth, no public URLs, no direct Python internals in UI.

## Change Checklist

- Before code changes, identify whether the app flow, backend API contract, or packaging/runtime setup is affected.
- Keep the Python pipeline behind the local API; do not move speech processing logic into React/Tauri UI code.
- Preserve CLI/API compatibility unless the change explicitly updates those contracts.
- For app-facing work, check frontend tests/build and run a Tauri smoke/build when local prerequisites allow it.

## Context and Verification

- Start with the task, this file, and the relevant [README](README.md) / [test plan](docs/test-plan.md) sections. Trace changed symbols and all callers; open manifests, scripts, API contracts, or release instructions only when the change reaches that boundary.
- Example: correcting a macOS packaging instruction needs the README section, `frontend/package.json`, `scripts/package-mac.sh`, its prerequisite/runtime scripts, and Tauri config. Updater/signing wording also requires [the release runbook](docs/mac-updater.md); a runtime claim requires the sidecar launcher. It does not require reading every ASR adapter or running transcription.
- If accepted requirements conflict with implementation, record the exact discrepancy and resolve the material decision before changing behavior. Do not silently rewrite requirements to match code or fix packaging inside a docs-only task.
- Command roles and platform prerequisites live in [README build instructions](README.md#сборка-desktop-app): frontend build is not a desktop bundle; low-level `tauri:build` does not prepare runtime resources; platform packaging does not install or publish. `install:local` and Windows `-Smoke` do install and change local state; require the corresponding scope and a safe environment.
- For app acceptance, launch the current packaged artifact and exercise the UI with its app-owned backend. Record version/commit, platform, artifact path and actual results; CLI/API/browser PASS cannot replace this. Missing signing, clean-machine, or device evidence stays explicitly not-run.
- Docs-only changes use `git diff --check`, command/source and relative-link checks, and a diff allowlist; do not run app suites just to validate prose. Preserve existing backend/frontend/native/release gates for tasks that affect those surfaces. Fully replaced instructions may be removed only with an exact replacement, incoming-link check, and Git recovery; preserve useful requirements and historical evidence.

## Refactor Policy

- Prefer small vertical refactor slices with unchanged behavior and tests after each slice.
- Split large files by responsibility: app shell/components/view-model, Tauri commands/settings/backend process, service routes/storage/responses/model runtime.
- Optimize first for maintainability and clear boundaries; do not tune ASR/diarization performance without a measured bottleneck.
