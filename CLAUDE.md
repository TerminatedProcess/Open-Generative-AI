# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Setup

`npm install` alone will NOT work. The repo depends on three git submodules and four npm workspaces that must be built first:

```bash
npm run setup   # = git submodule update --init --recursive && npm install && npm run build:packages
```

Submodules live at `packages/Vibe-Workflow`, `packages/Open-Poe-AI`, `packages/Open-AI-Design-Agent`. If any of them are empty, re-run `npm run setup`.

## Common commands

| Command | What it runs |
|---|---|
| `npm run dev` | Next.js dev server (hosted-style web build) at `http://localhost:3000` |
| `npm run electron:dev` | Vite-built bundle wrapped in Electron (desktop app) |
| `npm run build:packages` | Rebuild all workspace packages (run after editing `packages/studio` or any submodule) |
| `npm run build:studio` | Rebuild only `packages/studio` |
| `npm run vite:build` | Build the Electron renderer bundle into `dist/` |
| `npm run electron:build:{mac,win,linux}` | Build platform installers into `release/` |
| `node --test tests/` | Run unit tests (`node:test`; there is no `npm test` script) |
| `node --test tests/wan2gpModelAvailability.test.js` | Run a single test file |

Tests cover the Electron `electron/lib/*` utilities only. There is no UI test suite.

## Architecture: two parallel UIs, one shared model catalog

The repo ships **two independent renderers** of the same studio:

1. **Next.js + React** (`npm run dev`) — the hosted-style web app.
   - Entry: `app/studio/page.js` → `components/StandaloneShell.js`
   - UI components: `packages/studio/src/components/*.jsx` (React, exported from `packages/studio/src/index.js`)
   - Next.js transpiles the workspace packages via `transpilePackages: ['studio', 'ai-agent', 'workflow-builder', 'design-agent']` in `next.config.mjs`.

2. **Vite + Electron** (`npm run electron:dev`) — the desktop app.
   - Entry: `index.html` → `src/main.js`
   - UI components: `src/components/*.js` (**vanilla JS**, a completely separate implementation — not React)
   - Electron main: `electron/main.js`, preload: `electron/preload.js`

Both renderers import from the **same model catalog**. `packages/studio/src/models.js` (auto-generated from `models_dump.json`) is the single source of truth; `src/lib/models.js` is a 5-line re-export shim so the vanilla-JS components in `src/` see the same data. Edit models in `packages/studio/src/models.js` only.

The UI components themselves are NOT shared. A new feature added to `packages/studio/src/components/ImageStudio.jsx` will not appear in the Electron app unless the same change is also made in `src/components/ImageStudio.js`, and vice-versa.

## API integration: BYOK + three different proxy paths

The app calls `api.muapi.ai` with a `x-api-key` header (NOT `Authorization: Bearer`). The key lives in `localStorage` under `muapi_key`; missing/401/403 surfaces a `muapi:auth-required` window event that the shell catches to show the API-key modal.

All upstream calls follow a **submit → poll** pattern: `POST /api/v1/<endpoint>` returns a `request_id`, then poll `GET /api/v1/predictions/<id>/result` until `status` is `completed`/`succeeded`/`failed`. `pollForResult()` and `submitAndPoll()` in `packages/studio/src/muapi.js` (and the parallel `src/lib/muapi.js`) handle the normalization — output `url` may live under `outputs[0]` depending on the model.

Three proxy mechanisms route `/api/*` to `api.muapi.ai`, depending on which renderer is running:

- **Next.js dev/prod**: `middleware.js` rewrites `/api/workflow/*`, `/api/app/*`, and `/api/v1/*` to `https://api.muapi.ai`. Three paths are **excluded** because they have their own route handlers with custom logic: `/api/v1/creative-agent`, `/api/v1/get_upload_url`, `/api/v1/upload-binary`. If you add a route under those prefixes, add it to the exclusion list in `middleware.js`.
- **Vite dev server**: `vite.config.mjs` proxies `/api` → `https://api.muapi.ai` (used during `npm run electron:dev` build).
- **Electron renderer (`file://`)**: `BASE_URL` in `muapi.js` detects the non-http protocol and calls `https://api.muapi.ai` directly (no proxy).

`muapi.js` picks the right `BASE_URL` automatically; you generally do not need to touch this.

## Local inference (Electron only)

The desktop app supports two local engines, both wired through Electron IPC. The renderer talks to them via `window.localAI`, exposed by `electron/preload.js`:

- **sd.cpp** (bundled) — `electron/lib/localInference.js` registers `local-ai:*` IPC handlers (`binary-status`, `download-binary`, `list-models`, `generate`, …). The native binary and model weights live under `app.getPath('userData')/local-ai` by default.
- **Wan2GP** (BYO Gradio server) — `electron/lib/wan2gpProvider.js` registers `wan2gp:*` IPC handlers. Config persisted to `<userData>/local-ai/wan2gp.json`. The provider remaps Gradio `api_name`s at probe time because Wan2GP renames them between releases; see `WAN2GP_CATALOG` and `resolveFnNames()`.

Set `OPEN_GENERATIVE_AI_LOCAL_AI_DIR` to override the local-AI data dir (resolved in `electron/lib/localInferencePaths.js`). Both engines emit progress on the `local-ai:progress` IPC channel.

The hosted web build has no IPC bridge and cannot use local inference — `window.localAI` is undefined; UI code must guard for this.

## File-layout gotchas

- `src/lib/models.js` is a 5-line `export * from "studio/src/models.js"` shim. Do not put model definitions here.
- `models_dump.json` (605 KB) is upstream data. Don't hand-edit; `packages/studio/src/models.js` is regenerated from it.
- `project_knowledge.md` describes an older single-renderer version of the project. Treat it as historical context, not current architecture.
- `afterPack.js` and `scripts/stage-local-ai-binary.js` run during `electron-builder` packaging — they stage the sd.cpp binary into `build/local-ai/` so it can be shipped under `extraResources`.

## Conventions

- Use **named exports**, not default exports, for new helpers in `packages/studio/src/muapi.js` (matches existing style — the file has both, but new APIs are named).
- New studio features that should appear in **both** the Next.js shell and the Electron desktop app must be implemented in two places: the React component under `packages/studio/src/components/` and the vanilla-JS component under `src/components/`. There is no automated parity check.
- New API endpoints under `/api/v1/` that should be proxied to Muapi need no code; the middleware handles them. New routes that need server-side logic must be added to the exclusion list in `middleware.js`.
