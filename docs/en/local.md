# Local layer (fork-only)

The PocketArk framework itself **contains no personal tools and downloads no resources**. When you fork this
repository into your own software, personal content should be concentrated in the "local layer" so it keeps
syncing with the upstream base. The base repository keeps only this file as a convention note; your fork can
replace it with actual content.

## Where things go

| Content                                          | Location                                                                                     |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------- |
| Personal plugins (frontend and backend together) | `plugins/<id>/` (auto-registered at build time, no manual registration)                      |
| Personal Rust dependencies                       | The `# ── local plugin deps（fork-owned）──` section at the end of `src-tauri/Cargo.toml`    |
| Personal assets (fonts/models/…)                 | `src-tauri/local-resources/`                                                                 |
| Lifecycle hook implementation                    | `src-tauri/local/lifecycle.rs` (implements `AppLifecycle`, see [lifecycle.md](lifecycle.md)) |
| Personal download/build scripts                  | `scripts/local/` (`scripts/prepare.mjs` automatically calls `scripts/local/prepare.mjs`)     |
| Extra browser-side permissions                   | `src-tauri/capabilities/local.json` (auto-discovered by Tauri; base has no such file)        |

## Resources and build

- Put assets in `src-tauri/local-resources/`: `bundle.resources` already ships `local-resources/**/*`,
  so there is no need to touch `tauri.conf.json`.
- Resource downloads go in `scripts/local/prepare.mjs`: it runs automatically via
  `scripts/prepare.mjs` before `tauri dev` / `tauri build`; in CI environments it is recommended to skip the download yourself.
- If you need C++ dependencies (bindgen / cc): on macOS export `CXXFLAGS` yourself (for example
  `export CXXFLAGS="-std=c++14 -I$(xcrun --sdk macosx --show-sdk-path)/usr/include/c++/v1"`);
  on Windows install LLVM (`LIBCLANG_PATH`).

## Syncing with base

```bash
git remote add upstream <base url>
git fetch upstream
git merge upstream/main
```

Personal content is concentrated in the locations above, so merges generally have no conflicts; when `Cargo.lock`
conflicts, run `cargo build` to rebuild it.

## Release

If you enable online updates: generate your own key (`pnpm tauri signer generate`), put the public key into
`tauri.conf.json > plugins.updater.pubkey`, turn on `bundle.createUpdaterArtifacts: true`,
build with signing, and use `pnpm release` to generate the manifest (see [release.md](release.md) for details).
