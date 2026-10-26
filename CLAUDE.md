# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# SSH Manager

Cross-platform SSH server manager (Windows/macOS/Linux). Tauri 2 + Angular + Rust.
Inspired by OmnySSH; lightweight, no accounts, no telemetry.

## Commands
Run from the repo root unless noted. Cargo commands must run from `src-tauri/`.

- `npm run tauri dev` — full app in dev (launches the Angular dev server on port 1420, then the Tauri window). Primary way to run the app.
- `npm start` / `ng serve` — Angular frontend only, in a browser (no Rust/Tauri APIs; `invoke` calls fail).
- `ng build` — build the Angular frontend. Output: `dist/sshmanger/browser` (see architecture note on the `sshmanger` name).
- `npm run tauri build` — production desktop bundle.
- `cargo build` / `cargo clippy` — from `src-tauri/`; run after any Rust change.
- No test runner is configured yet (no `ng test`/`cargo test` targets wired up).

## Architecture
The app is one Angular SPA talking to a Rust backend over Tauri's IPC. Understanding three cross-cutting boundaries matters more than any single file:

- **Adding a Tauri command (the IPC contract).** A Rust function reachable from Angular needs two edits in lockstep: annotate it `#[tauri::command]`, and add it to the `tauri::generate_handler![...]` list in [src-tauri/src/lib.rs](src-tauri/src/lib.rs) (`run()` is the app entry point; [main.rs](src-tauri/src/main.rs) just calls it). The frontend then calls it via `invoke("name", { args })` from `@tauri-apps/api/core`. Custom commands do **not** need a capability grant — `capabilities/` is only for Tauri's own core/plugin APIs (see next point). `greet` is the scaffold example wired end-to-end through [app.component.ts](src/app/app.component.ts).
- **Capabilities / permissions.** [src-tauri/capabilities/default.json](src-tauri/capabilities/default.json) grants the main window access to core and plugin APIs. Per the rules below, add grants explicitly here when enabling a plugin/core permission; never use wildcards.
- **Config chain and the `sshmanger` name.** The npm package, the Rust crate, and `productName` are all spelled `sshmanger` (missing the second `a`). This name links the build output path `dist/sshmanger/browser` to `frontendDist` in [src-tauri/tauri.conf.json](src-tauri/tauri.conf.json), and the dev port 1420 is set in both [angular.json](angular.json) and `tauri.conf.json` (`devUrl` + `beforeDevCommand`). Changing any one of these breaks the chain — change them together or not at all.

Angular uses standalone components (no NgModules); providers live in [src/app/app.config.ts](src/app/app.config.ts) and routes in [src/app/app.routes.ts](src/app/app.routes.ts) (currently empty). Rust edition is 2024.

## Stack
- Frontend: Angular (standalone components, TypeScript), xterm.js for the terminal
- Backend: Rust in src-tauri/. SSH via the `russh` crate, SFTP via `russh-sftp`
- Config: TOML files in the OS config dir (hosts.toml, snippets.toml)
- Secrets: OS keychain via the `keyring` crate. Never store passwords or keys in config files.

(Note: `russh` + `tokio` are in [src-tauri/Cargo.toml](src-tauri/Cargo.toml) and drive SSH in [src-tauri/src/ssh.rs](src-tauri/src/ssh.rs). `russh-sftp` and `keyring` are not added yet — they land with SFTP and secret storage.)

## SSH session model
[ssh.rs](src-tauri/src/ssh.rs) opens an interactive shell per session, keyed by a generated session id in a `SessionManager` (Tauri-managed state). Flow: `connect` authenticates (key or password; `password` doubles as the key passphrase), opens a PTY+shell, and stores the session **Pending** (channel open but unread). The frontend subscribes to the session's events, then calls `attach`, which spawns the reader task and flips it to **Running** — this two-step avoids losing the initial shell banner. Output streams to the frontend as `pty-output-<id>` events (byte arrays); `pty-closed-<id>` fires on exit. `write_pty` / `resize_pty` / `disconnect` drive the live session. Host keys are trust-on-first-use in `known_hosts.toml` (keyed by `host:port`), refused on mismatch. The terminal UI is [terminal.component.ts](src/app/terminal.component.ts) (xterm.js + fit addon).

## Rules
- Keep the Rust layer thin: only networking, filesystem, PTY, keychain. All product logic in Angular.
- Expose Tauri commands explicitly; grant them in src-tauri/capabilities. No wildcard permissions.
- Strict host-key verification: save fingerprint on first connect, refuse on mismatch.
- Never log passwords, passphrases, or key contents. Zeroize key material after use.
- Private keys are referenced by path only; never copied into app data.
- Run `cargo build` and `cargo clippy` after Rust changes; `ng build` after Angular changes.
- Commit after each working step with a clear message.
- Explain any Rust concept you use that a TypeScript developer might not know, briefly, in your summary.

## Roadmap
Milestone 1 (MVP): hosts list (TOML) → SSH connect with key/password → terminal in xterm.js → dashboard metrics (CPU/RAM/disk)
Milestone 2: SFTP two-panel browser, snippets, broadcast to multiple hosts
Milestone 3: port forwarding UI, host groups/tags, alerts, per-host notes
Later: .ssh/config import, Docker/systemd panels, session logging

## Status
- [x] M1 step 1: hosts store + UI
- [x] M1 step 2: SSH connect + terminal
- [ ] M1 step 3: dashboard metrics
