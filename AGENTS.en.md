# PocketArk — AGENTS.md

> [中文](AGENTS.md) · English

A collection of desktop tools built with Tauri 2 + Vue 3 + TypeScript + Tailwind 4 (Mac / Windows).
Positioned as a **base project**: the cost of adding a new tool should be as low as possible, extensions follow plugin conventions, and framework code (`src/core`, the db/http/tray/updater/menu/tasks/open/… under `src-tauri/src`) is only referenced, never modified.

## Common Commands

```bash
pnpm dev            # vite dev (frontend)
pnpm tauri dev      # Tauri dev (compiles Rust first, takes a few minutes)
pnpm build          # vue-tsc --noEmit + vite build (must run after frontend changes)
pnpm preview        # preview the vite build output
pnpm test           # vitest unit tests (core pure logic)
pnpm test:watch     # vitest watch mode
pnpm lint           # eslint (must run after frontend changes)
pnpm lint:fix
pnpm format         # prettier --write . (whole repo)
pnpm format:check
pnpm lint:rs        # cargo clippy -D warnings (must run after Rust changes)
pnpm fmt:rs         # cargo fmt
pnpm test:rs        # cargo test (http smoke tests ignored by default, need network)
pnpm tauri build    # package
pnpm scaffold       # one-click rename after clone (interactive: display name/identifier/accent color, etc.)
pnpm create-plugin  # interactively generate a new plugin skeleton (plugins/<id>/, frontend and backend auto-registered)
pnpm icons          # generate all-platform icons from two sources (macOS HIG grid icns + Windows full-bleed ico/png)
pnpm version:bump   # bump the version (syncs tauri.conf.json / Cargo.toml / package.json)
pnpm release        # generate the update manifest latest.json + checksums (for release)
```

Convention: after frontend changes run `pnpm lint && pnpm build`; after Rust changes run `pnpm lint:rs && pnpm fmt:rs && pnpm test:rs`.

## Local Layer (fork-specific)

The framework itself **downloads no resources and contains no personal tools**. If your fork layers on personal tools (fonts/OCR and other resources, extra Rust dependencies, extra permissions), the conventions are centralized in the "local layer"; see `docs/en/local.md` at the repository root:
`src-tauri/local-resources/` (resources), `scripts/local/` (download scripts),
the `local plugin deps` section of `Cargo.toml`, `capabilities/local.json` (if present).

## Plugin Extensions (the most common task)

Adding a tool = `pnpm create-plugin` (interactively generates a skeleton, recommended), or copy `plugins/_template/` → `plugins/<plugin-id>/`, rewrite `plugin.json` (the single source of truth), and implement `frontend/views/*.vue`. **Both the frontend and the backend are auto-registered at build time**, with no manual registration needed.

- Frontend: Vite scans `plugins/*/frontend/**` (views / settings / setup.ts / schema.ts).
- Backend: `src-tauri/build.rs` scans `plugins/*/backend/mod.rs`, parses `#[tauri::command]` to auto-register commands, and reads `migrations.rs` to aggregate migrations. Commands must be defined in `backend/mod.rs`, the function name = the name called from the frontend (`<plugin_id>_` prefix), and they always return `Result<T, AppError>`.
- Migrations are **isolated by scope**: `migration(scope, version, ...)`, scope = plugin id, and version increments from 1 within the scope. For legacy databases, historical versions are bridged once via `plugin.json`'s `legacyMigrations` (old global version → new local version).
- Directories starting with `_` or `.` do not participate in registration (e.g. `_template`).

### Plugin Directory Conventions

Each plugin's internal structure, naming and required files follow a unified convention (a top-level whitelist for `frontend/`, pure logic in `frontend/lib/`, composables using `useXxx.ts`, backend commands only in `backend/mod.rs`, etc.); the full specification is in **[`plugins/README.en.md`](plugins/README.en.md)**.

- Machine validation: `scripts/lint-plugins.mjs` (R-P1~R-P6) is hooked into `pnpm lint`; `plugins/_template/` is validated too,
  guaranteeing that scaffold output is a spec-compliant sample. If an exception is truly needed, write `lint-plugins-ignore` in the file and note the reason.
- Always generate new plugins with `pnpm create-plugin` (the skeleton is already compliant), or copy `plugins/_template/`.

### Auto-Registration Conventions and Caveats

- **Commands must be written in `plugins/<id>/backend/mod.rs`**: `build.rs` only scans that file for `#[tauri::command]`; the remaining `.rs` files are its submodules (`pub mod xxx;`). A command may be on its own line or on the same line as `#[tauri::command]`, and attributes may take arguments (e.g. `#[tauri::command(rename_all = "camelCase")]`).
- **Command names must be globally unique and must carry the `<plugin_id>_` prefix**: duplicates would produce repeated match arms in the generated `generate_handler!`; `build.rs` detects this at build time and errors out directly.
- **Migrations must go in `plugins/<id>/backend/migrations.rs`**, exporting `pub fn all() -> Vec<Migration>`; in `migration(scope, version, ...)`, scope = plugin id and version increments from 1 within the scope. **Published migrations must not be modified, only appended.**
- **Legacy DB bridging**: if a plugin ever used old global version numbers, declare `legacyMigrations: { "old global version": new local version }` in `plugin.json`; at startup they are registered into `plugin_migrations` according to the mapping (DDL is not re-run).
- The **framework's own commands** are listed in `FRAMEWORK_COMMANDS` in `src-tauri/build.rs`: new framework commands (db/http/updater, etc.) must be appended there, otherwise they will not be registered.
- **Frontend entry paths**: `plugin.json`'s `tools[].entry` / `settings.entry` are relative to the plugin directory and **include the `frontend/` prefix** (e.g. `frontend/views/Tool.vue`).
- **Build-time mechanism**: `build.rs` locates the repo root `plugins/` via `CARGO_MANIFEST_DIR`, brings in each plugin backend with an absolute `#[path]`, and emits `cargo:rerun-if-changed=plugins`; running `cargo build` directly from `src-tauri` also works (no need to go through pnpm).
- **After adding a plugin**: if it does not appear automatically after the frontend glob changes, restart `pnpm dev`/`pnpm tauri dev`.
- **The template is not compiled**: `_template` is skipped by both the frontend and backend scans because of its `_` prefix; after copying it, change the command prefix to the new plugin id (`template_xxx` → `<id>_xxx`).
- **Tailwind**: plugin frontends live in the repo root `plugins/` (outside `src/`), and `assets/index.css` already has `@source '../../plugins'`; do not remove it.
- **Platform note**: `#[path]` is generated with forward slashes and should also work on Windows, but it has **not yet been tested on Windows**.

## App Updates (framework capability)

The framework has built-in online updates: frontend `src/core/updater`, Rust `src-tauri/src/updater.rs` (`updater_check` / `updater_install` / `updater_restart`). It uses the official `tauri-plugin-updater` plus a self-hosted static manifest.

- **The public key is hardcoded** in `tauri.conf.json > plugins.updater.pubkey` (the trust root, not changeable from the UI); the UI only configures the server address, **HTTPS only**.
- **Checked once after startup only** (controlled by `updateEnabled` + `updateAutoCheck`, no polling); `initUpdater()` is called by `main.ts`.
- On the Rust side, `updater_builder().endpoints([...])` overrides the endpoints at runtime; download progress is reported back via the `updater://progress` event.
- Settings keys: `updateEnabled` / `updateServerUrl` / `updateAutoCheck` / `updateLastCheckAt`.
- The single source of truth for the version = `tauri.conf.json > version`; before releasing, run `pnpm version:bump x.y.z` to sync all three places.
- Update selection is strict semver (remote > local), and **version numbers must be monotonically increasing**; the manifest `version` must match the build version.
- The private key (`~/.tauri/pocketark.key`) is not committed; local builds use `TAURI_SIGNING_PRIVATE_KEY_PATH`, and CI uses the Secrets `TAURI_SIGNING_PRIVATE_KEY` / `TAURI_SIGNING_PRIVATE_KEY_PASSWORD`.
- Automated release: the base provides `.github/workflows/release-reusable.yml` (reusable); a fork adds one caller. For the full flow see [release.md](docs/en/release.md).

## Framework Capability Index

`src/core/` is split into modules by capability; plugins only reference it, never modify it. The same-named capabilities on the Rust side are in `src-tauri/src/`.

| Capability                    | Frontend entry                    | Rust                                         | Description                                                                                                                                              |
| ----------------------------- | --------------------------------- | -------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| IPC                           | `@/core/ipc`                      | `generate_handler` (generated at build time) | `ipc<T>(cmd, args)`, command names constrained by `commands.gen.ts`; raw `invoke` forbidden                                                              |
| Event bus                     | `@/core/events`                   | `src/events.rs`                              | Typed `emitEvent/onEvent`, unified event prefix                                                                                                          |
| Logging                       | `@/core/logger`                   | `log`                                        | Unified across both ends, takes over console                                                                                                             |
| Errors                        | `@/core/errors`                   | `error.rs`                                   | `AppError{code,message}` normalization + global toasts                                                                                                   |
| Database                      | `@/core/db`                       | `db.rs`                                      | Drizzle(kdb) + generic CRUD + scoped migrations                                                                                                          |
| HTTP                          | `@/core/http`                     | `http.rs`                                    | reqwest, no CORS; advanced `send()` (redirect/SSL/timeout/cookie/multipart/binary/cancel); streaming `download()` + progress; driven by proxy settings   |
| Theme                         | `@/core/theme`                    | `set_window_appearance`                      | Light/dark/system + accent color + font size                                                                                                             |
| Plugin registration           | `@/core/plugins`                  | `plugins/mod.rs` (generated)                 | Auto-registered at build time for both frontend and backend                                                                                              |
| Online updates                | `@/core/updater`                  | `updater.rs`                                 | tauri-plugin-updater + static manifest                                                                                                                   |
| System notifications          | `@/core/notify`                   | `tauri-plugin-notification`                  | Permission request + settings toggle                                                                                                                     |
| Launch at login               | `@/core/autostart`                | `tauri-plugin-autostart`                     | Source of truth is the OS                                                                                                                                |
| DB transactions               | `runInTransaction` in `@/core/db` | `db.rs`                                      | Batch execution in a single transaction, rollback on failure                                                                                             |
| Data backup/restore/reset     | Settings page "Data"              | `db.rs`                                      | Auto-restart after restore/reset                                                                                                                         |
| Settings import/export        | `@/core/settings-transfer`        | `files.rs`                                   | Export JSON; takes effect after restart on import                                                                                                        |
| Diagnostics report            | `@/core/diagnostics`              | `diagnostics.rs`                             | Environment info + recent logs                                                                                                                           |
| Single instance               | —                                 | `tauri-plugin-single-instance`               | Second launch raises the main window + forwards args                                                                                                     |
| In-app shortcuts              | `@/core/shortcuts`                | —                                            | `registerShortcut('mod+k', fn)`                                                                                                                          |
| Global shortcuts              | `@/core/global-shortcut`          | `tauri-plugin-global-shortcut`               | Driven by a setting, raises the main window                                                                                                              |
| Global search/command palette | `@/core/search`                   | —                                            | `Cmd/Ctrl+K` aggregates navigation and tools                                                                                                             |
| Taskbar progress/badge        | `@/core/taskbar`                  | Tauri Window API                             | Progress 0–100 / Dock badge                                                                                                                              |
| Background tasks              | `@/core/tasks`                    | `tasks.rs`                                   | Cancel token + `task://` progress events                                                                                                                 |
| Open content                  | `@/core/open-with`                | `open.rs`                                    | Unified dispatch of CLI / deep links / drag-and-drop (`onOpenFiles`)                                                                                     |
| Multi-window                  | `@/core/windows`                  | `capabilities/windows.json`                  | `openAppWindow()`, label `win-*`                                                                                                                         |
| Native app menu               | `@/core/events` (`app://menu`)    | `menu.rs`                                    | macOS menubar / Win window menu; items forwarded to the frontend via events                                                                              |
| Platform detection            | `@/core/platform.ts`              | —                                            | `isMac` (styling and shortcut modifier-key differences)                                                                                                  |
| Lifecycle hooks               | —                                 | `lifecycle.rs`                               | Pre/post-start, pre/post-exit SPI; a fork implementing `src-tauri/local/lifecycle.rs` is auto-mounted (see [docs/en/lifecycle.md](docs/en/lifecycle.md)) |

> When adding a framework capability, register it in this table and add a note in the README's "Architecture conventions".

## Key Conventions

- **IPC**: the frontend always uses `await ipc<T>(cmd, args)` (`@/core/ipc`); raw `invoke` is forbidden. Command names are generated by `scripts/gen-commands.mjs` into `src/core/ipc/commands.gen.ts` (reading `src-tauri/framework-commands.json` and scanning plugin `#[tauri::command]`, as a union type of framework + plugin commands); unregistered commands error out during type checking; this file is generated (updated by `prepare`/the build, do not edit by hand).
- **Errors**: Rust `AppError{code, message}`, where code aligns with the frontend `core/errors` `ErrorCode`.
- **Database**: use `kdb` (Drizzle sqlite-proxy) for object-style queries; use `@/core/db`'s `db.insert/findAll` etc. for quick CRUD; always parameterize hand-written SQL with `$1`. Get auto-increment ids with `.returning()`. BLOB columns are returned over the channel as base64 (the frontend decodes them). Table/column names are validated against an identifier whitelist.
- **HTTP**: go through `@/core/http` (Rust reqwest, no CORS); `fetch` inside the webview is forbidden. Use `getJson/postJson/request` for simple/fetch requests; use `http.send()` (Rust `http_send`, isolated from the global Client used for fetching) when you need custom redirects, SSL verification, timeouts, cookie modes, multipart uploads, binary responses, or cancellation.
- **Logging**: use `@/core/logger`'s `logger` or the plugin's `ctx.logger`; raw `println!` is forbidden.
- **Icons**: only `@lucide/vue` is allowed (`lucide-vue-next` is deprecated, do not introduce it again).
- **Design spec (single source of truth)**: all UI visuals and interactions follow `docs/en/design.md` (shared core: tokens, component semantics, tri-state and feedback, copy, counterexample list) +
  `docs/en/design-macos.md` / `docs/en/design-windows.md` (platform layer) + `docs/en/design-appendix.md` (per-control Do/Don't). Visual values may only come from tokens;
  machine validation is handled by `scripts/lint-design.mjs` in `pnpm lint` (R1 old card recipe / R2 arbitrary-opacity surfaces / R3 rounded-xl on non-overlays /
  R4 hardcoded control sizes / R5 suppressed motion / R6 focus-ring expressions), with `pnpm lint:design` as strict mode. **Platform differences are only allowed in**:
  material and fallbacks, window chrome, menu and shortcut presentation, dialog button semantics, focus visuals, corner-radius tiers (implemented by `[data-platform]` token overrides).
- **Doc index**: all specs and guides live in `docs/`; for the index see [`docs/README.md`](docs/README.md) (including doc governance conventions).
- **Design walkthrough**: `src/dev/preview-bridge.ts` (active only in dev + non-Tauri) lets the Web layer render in a browser,
  and together with `scripts/design-shot.mjs` it can script screenshots to check against the spec (usage in `docs/en/design-appendix.md §3.5`).
- **Styling**: shadcn-vue semantic colors (`bg-primary`, etc.); hardcoded color values are forbidden; tokens like `text-success/warning/info` and `bg-console` already exist.
- **Tool page modules**: every feature module on a page (settings / input / output / result / list) must be wrapped in `@/components/tool/Panel` (outer frame + header-bar title + top-right actions + body); do not leave modules bare on the page, and do not nest cards inside a Panel — components like previews and tables carry no frame of their own; the frame belongs to the Panel. Use `text-xs font-medium text-muted-foreground` for the header title, `size="sm"` for action buttons and `size-3.5` for icons; the body defaults to `space-y-3 p-4` (override with `body-class` for full-bleed cases); error bars use `rounded-md border border-destructive/30 bg-destructive/10 px-3 py-2 text-xs text-destructive`.
- **Settings page UI**: the system settings page and all plugin settings panels uniformly use `SettingsSection` / `SettingsRow` / `SettingsField` from `@/components/settings` (single source of truth, visuals follow the system settings page); do not invent your own section headings or card styles; for dense numeric fields, use a grid + `SettingsField` inside `SettingsSection`.
- **Code style**: prettier single quotes, 100 columns, es5 trailing commas; run `pnpm format` before committing.
- **Theme**: accent color / light-dark live in `core/theme` + the accent class in `assets/index.css`; adding an accent color requires syncing three places (CSS / ACCENTS / the inline anti-flash script in `index.html`).
- **Lifecycle hooks**: `src-tauri/src/lifecycle.rs` only defines the contract and the call sites; the base contains no implementation; a fork writing `pub struct Hooks;` in `src-tauri/local/lifecycle.rs` is auto-mounted by `build.rs` (no file means a no-op). Hooks run in a separate async task — **do not block the main thread inside a hook** (`block_on` will freeze startup, so that not even self-created windows can be drawn). After `before_start` returns `Hold`, you must call `lifecycle::resume_startup()` before the main window can be shown; during that time, tray/single-instance/Dock activation is uniformly rejected by `tray.rs` (do not bypass this gate).

## Pitfalls (already fixed, do not regress)

- **Log loop**: `core/logger` uses `attachLogger` + an `echoing` guard. Do not revert to `attachConsole` or let the console takeover function forward logs during echo — that causes an infinite loop.
- **keepAlive**: `MainLayout` branches on `route.meta.keepAlive !== false` to cache; only `keepAlive: false` in the manifest makes it not cache.
- **Ordering**: plugin group order = the smallest `order` within the group; `manifest.order` participates in ordering — do not revert to directory-name ordering.
- **Security boundary**: App commands such as `db_*` / `http_request` are not gated by capabilities, so arbitrary code inside the webview can execute arbitrary SQL/requests. Keep CSP strict and do not load remote content.
- **Theme-appearance coupling**: `applyTheme` also syncs NSWindow.appearance / NSWindow.backgroundColor (macOS). Forcing darkAqua/aqua pins the webview's `prefers-color-scheme` to that appearance — "follow system" must pass `dark: null` to clear the override (see the tri-state design of `set_window_appearance`).
- **Tool page layout**: `ToolShell` only provides a page header + full-width container; centered columns / split columns are done inside the page with native Tailwind grid (`grid grid-cols-12` + `col-start-*` / `col-span-*`, using an even span for a centered column); the `mx-auto max-w-*` centered container is forbidden; if a new registry component uses bare boolean variants like `data-checked:`/`data-open:`, convert them to `data-[state=...]:` (reka-ui 2.10 only emits the latter).
- **macOS dev port in use**: on exit, the tauri CLI (≤2.11.4) relies on `$TMPDIR/tauri-stop-dev-processes.sh` to clean up the dev server process tree, but in practice the script is created as **0 bytes + 0o644** (both the content and the permission writes fail silently), and the `!exists()` guard never recreates it → vite becomes orphaned → the next `tauri dev` reports the port in use. chmod alone is not enough (an empty script = no-op); you must write the original content + the execute bit (once is enough):

  ```bash
  cat > "${TMPDIR}tauri-stop-dev-processes.sh" << 'EOF'
  #!/usr/bin/env sh
  getcpid() {
      cpids=$(pgrep -P $1|xargs)
      for cpid in $cpids; do
          echo "$cpid"
          getcpid $cpid
      done
  }
  kill $(getcpid $1)
  EOF
  chmod 755 "${TMPDIR}tauri-stop-dev-processes.sh"
  ```

  Emergency fallback: `lsof -ti:1420 | xargs -r kill -9`. Upstream tauri#15098; this item can be removed once fix PR #15108 (open) is merged and the CLI is upgraded.

- **macOS tray click does not activate the window**: with `show_menu_on_left_click(false)`, clicking the menu bar icon does not make AppKit activate the owning app; `show()`/`set_focus()` only do `makeKeyAndOrderFront`, so the window is sent to a lower layer — it looks like "clicking did nothing", and only after switching to another app do you see the window is shown (the Dock icon works because macOS activates the app). Upstream tauri#14795 (still unfixed in tray-icon 0.25.0). `tray::show_main_window` has been changed to retry immediately + one beat later (next runloop), and on macOS to call `NSApp.activateIgnoringOtherApps(true)`; also `RunEvent::Reopen` explicitly raises the main window. Do not delete these two, otherwise it regresses.

- **Local-layer build dependencies (ocr-rs, etc.)**: heavy dependencies like these belong to the fork's local layer; the related build issues (macOS `CXXFLAGS`, Windows libclang) are handled by the fork itself and documented in `docs/en/local.md`; the base does not include these dependencies, so it has no such problem.

## Directory Overview

```
src/core/            # framework core (logger/errors/ipc/db/http/theme/plugins), extensions only reference
src/components/tool/ # ToolShell tool page shell (page header + full-width container) + Panel module panel
src/components/ui/   # shadcn-vue generated components (managed by CLI)
src/content/         # About/changelog Markdown (read by the settings page)
scripts/             # scaffold / create-plugin / gen-icons / gen-commands / prepare / bump-version / release
  local/             # ★local-layer scripts (fork-owned; none in base): resource download, CI skip
plugins/<id>/        # ★tool plugins (frontend and backend together): plugin.json + README.md
  frontend/          #   views/ settings/ components/ composables/ lib/ schema.ts setup.ts shared.ts
  backend/           #   mod.rs (#[tauri::command] commands) + migrations.rs + other .rs / resources
src/layouts/         # layout shell (title bar/sidebar/error boundary/settings page)
src/stores/          # Pinia (global settings)
src-tauri/src/       # db.rs (sqlx + scoped migrations), http.rs, tray.rs, updater.rs, tasks.rs, open.rs, menu.rs, diagnostics.rs, files.rs, lifecycle.rs, plugins/mod.rs (include generated)
src-tauri/build.rs   # scans plugins/ to generate command registration and migration aggregation; scans local/lifecycle.rs to generate hook mounting
src-tauri/local/     # ★local-layer code (fork-owned; base keeps only the README convention): lifecycle.rs = lifecycle hook implementation
src-tauri/local-resources/  # ★local-layer resources (fork-owned; none in base)
docs/                # ★specs and guides (index at docs/README.md, doc governance in that file)
  local.md           #   local layer notes (fork-specific)
  lifecycle.md       #   lifecycle hook (SPI) contract and integration
```

The generated shadcn components `src/components/ui/**` are managed by the CLI: you may change styles, but not structure/logic.
