# START — Build Your Own Desktop Software with This Repository

> This document is for: people who have cloned ArkDesk and want to turn it into **their own desktop tool software**.
> Follow it to the end and you will get a desktop application named after your software, carrying your icon and theme, loaded with your own tools.
>
> Division of the documentation: **start.md** (this file) = the from-scratch handbook; **extending.md** = architecture and extension development reference;
> **AGENTS.md** = daily development conventions and pitfalls checklist; **release.md** = packaging / release / online updates;
> **local.md** = the fork local layer; `plugins/_template/README.md` = single-plugin scaffold documentation.

---

## 0. What Is This Project

A **desktop toolbox foundation**: Tauri 2 + Vue 3 + TypeScript + Tailwind CSS 4 + shadcn-vue.
It has already done all the dirty work "unrelated to the application itself" for you—windows, tray, sidebar navigation, routing, theme,
SQLite, settings persistence, logging, error handling, HTTP channel—all you have to do is: write your tool pages.

The framework code (`src/core/`, and db/http/tray/updater/menu/tasks/open/… in `src-tauri/src`) is **referenced only, never modified**;
all extensions are done through the plugin mechanism (see Section 6).

## 1. Environment Preparation

| Dependency | Version        | Description                       |
| ---------- | -------------- | --------------------------------- |
| Node.js    | 20+            | Includes npm                      |
| pnpm       | 9+             | Package manager (`npm i -g pnpm`) |
| Rust       | stable (1.77+) | Installed with `rustup`           |

Additional platform dependencies:

- **macOS**: Xcode Command Line Tools (`xcode-select --install`)
- **Windows**: MSVC Build Tools (in the VS Installer, check "Desktop development with C++") + WebView2 Runtime (bundled with Win11)
- **Linux**: system libraries such as webkit2gtk, see [Tauri prerequisites](https://tauri.app/start/prerequisites/)

## 2. First Run

```bash
pnpm install
pnpm tauri dev      # the first run compiles Rust, taking a few minutes; afterwards incremental builds take seconds
```

When the window appears, it worked. At this point it is ArkDesk in the "dark command console" style, with example tools—
the next two steps turn it into your software: **rename** (Section 3) → **delete the examples and write tools** (Sections 5 and 6).

## 3. Rename: Make It Your Own Software (One-Click Script)

```bash
pnpm scaffold
```

Answer the 5 questions as prompted (press Enter to keep the default):

| Question             | Description                                                                                                                              |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| App display name     | Appears in the window, Dock/taskbar, tray tooltip, and settings page (Chinese/English both fine)                                         |
| English identifier   | kebab-case (such as `my-kit`), used for the package name and binary name                                                                 |
| App identifier       | `com.arkdesk.<English identifier>`, **cannot be changed again after release** (changing it makes the system treat it as a different app) |
| Default accent color | Indigo / Teal / Amber / Rose / Azure / State Grid Green                                                                                  |
| Default theme        | dark / light                                                                                                                             |

The script automatically rewrites all names in `tauri.conf.json`, `Cargo.toml`, `main.rs`, `package.json`,
the documentation, and the descriptions. When it is done, run as prompted:

```bash
pnpm install        # refresh the package name in the lockfile
pnpm tauri dev      # run under the new name
```

<details>
<summary>Prefer to change it by hand? Here is the complete rename checklist (equivalent to the script)</summary>

| File                                                             | What to change                                                         |
| ---------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `src-tauri/tauri.conf.json`                                      | `productName`, `identifier` (cannot be changed after release), `title` |
| `src-tauri/Cargo.toml`                                           | `[package] name` (kebab), `[lib] name` (snake + `_lib` suffix)         |
| `src-tauri/src/main.rs`                                          | `arkdesk_lib::run()` → the new lib name                                |
| `package.json`                                                   | `name`                                                                 |
| `src/core/theme/index.ts`                                        | the `DEFAULT_THEME` / `DEFAULT_ACCENT` constants (optional)            |
| `src/content/about.md`, `README.md`, `docs/en/*.md`, `AGENTS.md` | self-introduction and titles                                           |
| `index.html`                                                     | `<title>`                                                              |

The frontend UI (sidebar/top bar) and the tray tooltip **read** productName **dynamically**, so no code changes are needed.
</details>

## 4. Change the Icon (Dual Source, Best per Platform)

The icon has two source files (same graphic paths, only the frame scaling/corner radius differ per platform convention):

| Source file                        | Style                                                                | Generated artifacts             | Platform used                        |
| ---------------------------------- | -------------------------------------------------------------------- | ------------------------------- | ------------------------------------ |
| `src-tauri/icons/icon-macos.svg`   | HIG grid padding (main graphic 80.5%, equivalent corner radius ≈185) | `icon.icns` + Store tiles       | macOS Dock / Finder                  |
| `src-tauri/icons/icon-windows.svg` | Full bleed (the graphic fills the canvas)                            | `icon.ico` + PNGs at every size | Windows taskbar/File Explorer, Linux |

Modify the **graphic in both SVGs** (keeping each one's frame parameters), then regenerate everything with one command:

```bash
pnpm icons
```

The script runs `tauri icon` twice and assembles per platform: icns/tiles = the macOS version, ico/png = the Windows version,
and `public/icon.svg` is synced as well. Each platform build (`pnpm tauri build`) automatically uses the icon in its own format.

If the macOS Dock still shows the old icon: `killall Dock`.

## 5. Delete the Example Plugins

The base repository ships with only two built-in plugins; keep or delete them as needed:

| Plugin        | Description                                                       | Recommendation                    |
| ------------- | ----------------------------------------------------------------- | --------------------------------- |
| `hello-world` | Demonstrates the full pipeline (Rust/SQLite/HTTP/layout), 5 tools | Can be deleted for production use |
| `system`      | Data maintenance (frontend-only, built into the framework)        | Keep                              |

```bash
# for example, delete the example (keeping system / _template)
rm -rf plugins/hello-world
```

- Both the frontend (sidebar/routes/settings page) and the backend (Rust commands/migrations) are scanned automatically at build time, so **deleting the directory removes it completely**
  (migration records are scope-isolated, so deleting a plugin does not affect other plugins)
- `_template/` is the scaffold for your new tools, **be sure to keep it**
- If your fork adds tools that need resources/heavy dependencies, see [local.md](local.md) (local layer documentation)

## 6. Develop Your First Tool

**Two steps**:

Generating it interactively with the scaffold is recommended (it fills in the id/name/icon and the optional backend, settings, database, etc. for you):

```bash
pnpm create-plugin
```

Or copy the template manually:

```bash
cp -r plugins/_template plugins/my-tools
```

Then edit `plugins/my-tools/plugin.json`:

```json
{
  "id": "my-tools",
  "name": "My tool",
  "tools": [
    {
      "id": "my-tools-hello",
      "name": "Hello",
      "description": "My first tool",
      "icon": "Sparkles",
      "entry": "frontend/views/Tool.vue"
    }
  ]
}
```

Then rewrite `frontend/views/Tool.vue`—done. The sidebar navigation, routing, titles, enable/disable toggle, settings panel,
error boundary, and KeepAlive all take effect automatically, **with no manual registration of any kind required**.

Reference implementations (all in the hello-world plugin, to compare against before deleting it):

| What you want to do                                                 | Which file to look at                                              |
| ------------------------------------------------------------------- | ------------------------------------------------------------------ |
| Call Rust commands, read/write SQLite                               | `frontend/views/Tool.vue` (a comprehensive Rust integration guide) |
| Table + pagination + modal create/update/delete + import/export     | `frontend/views/TableTool.vue`                                     |
| Various form controls and validation                                | `frontend/views/FormTool.vue`                                      |
| Send HTTP requests and display the results                          | `frontend/views/HttpTool.vue`                                      |
| Page layout and how to write the three states (empty/loading/error) | `frontend/views/TemplateTool.vue`                                  |

For the complete steps of advanced capabilities (Rust commands, database tables, settings panel, HTTP collection),
see [extending.md](extending.md).

## 7. Capability Extension Cheat Sheet

| Need                | Entry point                                                                                                                     | Files involved                                     |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------- |
| Rust commands       | Write `#[tauri::command]` in the plugin's `backend/mod.rs` (registered automatically at build time)                             | `backend/mod.rs`                                   |
| Database tables     | Define in `frontend/schema.ts` + append a migration in `backend/migrations.rs` (scope = plugin id, version +1 within the scope) | `schema.ts` + `migrations.rs`                      |
| Settings panel      | `useToolSettings` + declare `settings.entry` in `plugin.json`                                                                   | `frontend/settings/Settings.vue`                   |
| HTTP requests       | `core/http` (`http.get/post/getJson`, no CORS)                                                                                  | Called directly from the frontend                  |
| Import/export files | `tauri-plugin-dialog` (already integrated) + custom Rust commands                                                               | See `TableTool.vue` + `hello-world/backend/mod.rs` |

## 8. Theme and Branding

- **Accent color**: the settings page can switch between 6 options; to add a custom color, you must keep three places in sync:
  the accent class in `assets/index.css` → the ACCENTS in `core/theme` → the anti-white-flash script in `index.html`
- **Default accent color / default theme**: change `DEFAULT_ACCENT` / `DEFAULT_THEME` in `core/theme/index.ts`
  (`pnpm scaffold` can also set them interactively)
- **Root font size**: three levels in the settings page (Small 13 / Normal 14 / Large 15), with the entire UI scaling proportionally;
  for the level scale definitions, see the comment table in `assets/index.css`
- **Dark base tone**: dark by default (deep-space blue-gray), with the light theme maintained in parallel; when modifying the palette, refer to
  both the `:root` / `.dark` sections in `index.css` and `NATIVE_BG` in `core/theme` (the native window background color)

## 9. Page Layout

`ToolShell` provides a page header (title/description/action buttons) + a full-width content area; a centered column / split layout is organized
inside the page with the native Tailwind grid (no custom grid concept is introduced):

```vue
<ToolShell title="My tool" description="A short description">
  <div class="mx-auto grid w-full grid-cols-12">
    <div class="col-span-12 lg:col-start-3 lg:col-span-8">
      <!-- centered 8 columns; full-width automatically below lg -->
    </div>
  </div>
</ToolShell>
```

Common widths: a table is full-width (no class added) / wide content `col-span-10` / mixed `col-span-8` /
forms `col-span-6`—a centered column must use an **even span**. For the standard way to write the empty/loading/error states,
see the "Page Template" tool (`TemplateTool.vue`).

## 10. Packaging and Distribution

```bash
pnpm tauri build
```

Artifact locations (when `--target <triple>` is given explicitly, there is an extra triple directory level):

- macOS: `src-tauri/target/**/release/bundle/macos/*.app` (with updates enabled there is also
  `<Product>.app.tar.gz` + `.sig`)
- Windows: `src-tauri/target/**/release/bundle/nsis/*-setup.exe` (msi depends on `--bundles`)

> For releasing a version with **online updates** (signing, manifest, upload, automatic release), see [release.md](release.md).

Security note (can be shown to a reviewer): the installed build **does not listen on any local port** (frontend assets are loaded
through an in-process custom protocol, with no web server); the only network activity is HTTP requests initiated by the application itself.
To edit the changelog and the about page, just edit `src/content/*.md`.

## 11. FAQ

<details>
<summary><b>Restarting dev on macOS reports "Port 1420 is already in use"</b></summary>

On macOS the tauri CLI's cleanup script may be created as an empty file without the execute bit, leaving the
vite process behind on exit. One-time fix:

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
lsof -ti:1420 | xargs -r kill -9   # clear the current leftovers
```

(Upstream tauri#15098; you can ignore this entry once fix PR #15108 is merged and the CLI is upgraded)
</details>

<details>
<summary><b>The app is still running after I close the window?</b></summary>

By design: closing the window **hides to the tray** by default (resident in the background, ready to be summoned at any time). To truly quit:
right-click the tray icon → Quit (on macOS you can also use Cmd+Q). You can turn off "Hide to tray" on the settings page.
</details>

<details>
<summary><b>The console reports ELIFECYCLE / a large numeric exit code during dev (Windows)</b></summary>

When you quit the app, the tauri CLI force-terminates the vite dev server, and on Windows the terminated process
reports an NTSTATUS code (such as 4294967295). This is dev-mode noise and does not exist in the packaged build.
</details>

<details>
<summary><b>Some parts of the UI don't change after I adjust the font size / theme?</b></summary>

All UI scales with the settings; if you find a part of the UI that doesn't follow, it most likely has hard-coded px values—
switch to `rem`/semantic tokens (`text-sm`, `h-9`, etc.).
</details>

## 12. Pre-Release Checklist

- [ ] `pnpm scaffold` has renamed the app (identifier confirmed correct—it cannot be changed after release)
- [ ] The icon has been replaced and regenerated
- [ ] `src/content/about.md` / `changelog.md` contain your own introduction and the first version's changelog entry
- [ ] The example plugins have been deleted or disabled
- [ ] `pnpm lint && pnpm build` passes
- [ ] The `pnpm tauri build` artifact passes an install smoke test on the target platform (startup/exit/tray/maximize without flicker)

## 13. Where to Look During Daily Development

- **AGENTS.md**: code conventions and pitfalls (must-read before changing code)
- **extending.md**: architecture details and the complete steps for extension development (Rust commands/database/settings panel)
- **`plugins/_template/README.md`**: file-level documentation for a single plugin

Happy developing.
