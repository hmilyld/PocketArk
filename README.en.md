# PocketArk

> [中文](README.md) · English

A reusable **desktop application foundation** (Tauri 2 + Vue 3 + TypeScript + Tailwind 4).
Core design goal: **make adding a new tool as cheap as possible** — copy a template or generate a
skeleton with `pnpm create-plugin` and you immediately get navigation, routing, start/stop management,
a settings panel, database, logging, error handling, system integration and in-app updates;
the framework code (`src/core`, `src-tauri/src`) is reference-only and never modified.

The repo itself is "framework + examples": `_template` (scaffold), `hello-world` (tutorial example)
and `system` (data maintenance). To build your own app with it: clone, run `pnpm scaffold` and follow
[`docs/start.md`](docs/start.md); put your personal tools in the "local layer"
(see [`docs/local.md`](docs/local.md)) so they stay in sync with the upstream base.

## Tech stack

| Layer             | Choice                                                                                                                 |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Desktop framework | Tauri 2 (tray-icon / macos-private-api feature)                                                                        |
| Frontend          | Vue 3 + TypeScript + Vite 6                                                                                            |
| Visuals           | Tailwind CSS 4 + shadcn-vue (Reka UI) + lucide icons                                                                   |
| State / routing   | Pinia + vue-router (hash; routes driven by the plugin registry)                                                        |
| Data              | Drizzle ORM (object-style queries on the frontend) + a Rust-built sqlx channel (SQLite) + plugin-store (settings JSON) |
| Infrastructure    | tauri-plugin-log / window-state, system tray, unified error pipeline                                                   |

## Quick start

```bash
pnpm install
pnpm tauri dev      # develop (the first Rust build takes a few minutes)
pnpm tauri build    # package
pnpm lint           # eslint + design / plugin-structure / docs checks
pnpm build          # type check + frontend build
pnpm test           # vitest unit tests
```

Requirements: Rust, Node 20+, pnpm. macOS needs Xcode Command Line Tools; Linux needs system libraries
such as webkit2gtk (see [Tauri prerequisites](https://tauri.app/start/prerequisites)).

> The base ships with no bundled assets; if your fork adds tools that need assets such as fonts or OCR
> models, see [`docs/local.md`](docs/local.md) (the local layer).

## Documentation

| Want to know                                                      | Look at                                                |
| ----------------------------------------------------------------- | ------------------------------------------------------ |
| Full documentation index                                          | [`docs/README.md`](docs/README.md)                     |
| Build your own app from scratch (rename / icons / packaging)      | [`docs/start.md`](docs/start.md)                       |
| Architecture and extension development (commands / DB / settings) | [`docs/extending.md`](docs/extending.md)               |
| UI design spec                                                    | [`docs/design.md`](docs/design.md) + platform chapters |
| Packaging, release and in-app updates                             | [`docs/release.md`](docs/release.md)                   |
| Local layer (personal dependencies / assets)                      | [`docs/local.md`](docs/local.md)                       |
| Development conventions, capability index, pitfalls               | [`AGENTS.md`](AGENTS.md)                               |

> The documentation under `docs/` is currently written in Chinese.

## Collaboration and upstream

This repository is the framework base. Forks sync framework changes through the `upstream` git remote;
personal content lives in the "local layer" (see [`docs/local.md`](docs/local.md)). Change ownership is
determined by [`docs/ownership.json`](docs/ownership.json).

## License

This project is open-sourced under the [MIT License](LICENSE), copyright hmilyld.
