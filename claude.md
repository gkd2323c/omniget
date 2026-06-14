# OmniGet — Claude Code Context

## Rules

- Do not make commits without asking first
- **Commits are authored by the user ONLY.** NEVER add Claude/Anthropic as author, co-author, or contributor. Do NOT add `Co-Authored-By: Claude ...`, `Generated with Claude Code`, or any similar trailer/attribution to commit messages or PR bodies. This overrides any default harness behavior. The commit message contains only the change description — nothing about who/what wrote it.
- Do not add comments to the code
- Do not modify files outside the current task scope
- Run `pnpm check` after frontend changes; `cargo check` after Rust changes; `pnpm test` after store/i18n logic changes
- Use the existing code style (check neighboring files for conventions)
- Never use `unwrap()` on HTTP responses, JSON from APIs, or user input
- Never log tokens, passwords, or cookie values — only names/metadata
- Never use `std::sync::Mutex` in async context — use `tokio::sync::Mutex`
- Always use `?` or `.map_err()` for error propagation on external data

## Skills to use

For ANY UI, UX, copy, color, theme, hierarchy, layout, settings, or marketing decision: invoke the **`behavioral-design`** skill. It enforces a 5-question pause (who is the user, what behavior do we want, what bias is in play, where is the friction, is this an ethical nudge or a dark pattern) before every visual decision. This is mandatory — do not skip.

For visual quality passes (audit, distill, polish, normalize): use the **`impeccable`** skill. Install with `npx skills add pbakaus/impeccable` if not present. Use `/audit` to find issues, `/distill` to remove complexity, `/polish` for final cleanup, `/normalize` to fix inconsistencies.

Do not load other design-related skills — they compete for context with these two and degrade output.

## Architecture

Tauri 2 + SvelteKit (Svelte 5 with runes) + Rust. Desktop app for downloading media, plus a plugin ecosystem (courses, study, telegram, convert, misc utilities).

Core nav surfaces: home, downloads, marketplace, settings, about. Source of truth: `src/lib/nav-config.ts` `CORE_NAV_ITEMS`. Plugin nav items appear when their plugin is installed.

Plugin frontends (courses, study, telegram, convert, misc) live in this main repo; their Rust backends are separate DLL crates (sibling folders, see Ecosystem below). The **league** plugin and the old **social** lookup plugin are discontinued and removed. Note: `misc` today is the **utilities plugin** (Studio recording, AI models, library, tracking) — it is NOT the old discontinued social plugin.

```
omniget/
├── src/                          ← SvelteKit frontend (Svelte 5 with runes)
│   ├── routes/
│   │   ├── +page.svelte          ← home (omnibox + Loop mascot + quick actions)
│   │   ├── +layout.svelte        ← sidebar nav, dialog mounts, theme application
│   │   ├── settings/             ← settings (9 tabs: downloads, appearance, typography, network, cookies, channels, ai, plugins, advanced)
│   │   ├── downloads/            ← download history + filters + per-item actions
│   │   ├── marketplace/          ← plugin install / update / disable / remove
│   │   ├── about/                ← hero + sub-tabs (terms, project, roadmap, changelog, debug — debug gated by toggle)
│   │   ├── courses/              ← courses plugin frontend
│   │   ├── convert/              ← convert plugin frontend
│   │   ├── telegram/             ← telegram plugin frontend
│   │   ├── study/                ← study plugin frontend (20+ sub-routes: player, reader, notes, flashcards, focus, achievements, anki)
│   │   └── misc/                 ← misc utilities hub (studio screen-recording, file-clip, models, library, tracking)
│   ├── components/
│   │   ├── mascot/               ← Loop mascot (7 emotions, bubble pools, reduced-motion)
│   │   ├── omnibox/              ← URL input, format selector, quality picker, batch, search results, media preview
│   │   ├── toast/                ← Toast notifications
│   │   ├── dialog/               ← DialogContainer + Confirm/Recovery/Legal/Changelog/Shortcuts dialogs
│   │   ├── onboarding/           ← OnboardingWizard (mounted when needsOnboarding())
│   │   ├── p2p/                  ← P2pSendDialog / P2pReceiveDialog
│   │   ├── hints/                ← ContextHint (dismissible + persistent in localStorage)
│   │   ├── icons/                ← PlatformIcon
│   │   ├── services/             ← SupportedServices grid (idle empty-state)
│   │   ├── debug/                ← DebugPanel (gated by debug-store toggle)
│   │   ├── download/             ← DownloadLog, DownloadSpeedGraph
│   │   ├── downloads/            ← downloads-page UI pieces
│   │   ├── home/                 ← home page layout pieces
│   │   ├── celebrate/            ← mascot/achievement celebration
│   │   ├── settings/             ← settings sub-components
│   │   └── hotmart/              ← courses plugin UI components
│   ├── lib/
│   │   ├── stores/               ← Svelte 5 runes-based stores
│   │   ├── i18n/                 ← 11 locales (en, pt, es, ru, zh, zh-TW, ja, it, fr, el + 1); keys.ts is auto-generated
│   │   ├── plugin-invoke.ts      ← pluginInvoke() wrapper for plugin commands
│   │   ├── nav-config.ts         ← CORE_NAV_ITEMS — source of truth
│   │   ├── error-translate.ts    ← backend error → i18n
│   │   ├── build-info.ts         ← version, commit, branch, date (vite-injected)
│   │   ├── updater.ts            ← Tauri updater wrapper
│   │   ├── rpc.ts                ← RPC bridge
│   │   ├── tracker-*.ts          ← channel-following bridges (notifications, privacy, settings, icons)
│   │   ├── reader-components/    ← study reader UI; reader-theme/typography/session
│   │   ├── study-components/     ← study plugin UI primitives (incl. player/)
│   │   ├── study-music/          ← music player store
│   │   ├── anki-bridge.ts, notes-bridge.ts, study-gamification.ts, study-focus-bridge.ts
│   │   └── time-ago.ts           ← shared time formatting helper
│   └── app.css                   ← global CSS: 14 themes, ~74 custom properties per theme (token slimming to ~18 semantic tokens is planned, not done)
├── src-tauri/
│   ├── src/
│   │   ├── commands/             ← ~120 #[tauri::command] across ~23 modules (downloads, settings, plugins, dependencies, ai, subtitle_ws, channels, video_ops, …); all registered in lib.rs generate_handler!
│   │   ├── core/                 ← shared utilities (queue, queue_history, http_client, registry; re-exports from omniget-core)
│   │   ├── cookies/              ← multi-account cookie manager (commands, parsers, platform, storage) — see COOKIES.md
│   │   ├── models/               ← AI model management
│   │   ├── storage/              ← SQLite storage layer (WAL; history + channel follows; legacy JSON import on first run)
│   │   ├── platforms/            ← 15 media downloaders (instagram, pinterest, tiktok, douyin, twitter, twitch, bluesky, reddit, youtube, vimeo, bilibili, gallerydl, magnet, p2p, generic_ytdlp)
│   │   ├── plugin_loader.rs      ← libloading DLL loader
│   │   ├── plugin_host.rs        ← PluginHostImpl
│   │   └── lib.rs                ← app setup, AppState, generate_handler!
│   ├── omniget-core/             ← shared library (ffmpeg, http, yt-dlp, HLS, direct_downloader, etc.)
│   └── omniget-plugin-sdk/       ← OmnigetPlugin trait + export_plugin! macro
├── browser-extension/            ← Chrome / Firefox extension (chrome/, firefox/, sync.mjs; chrome/tests/)
├── scripts/                      ← deploy-plugins-local.{ps1,mjs}, generate-i18n-keys.js, sync-locales.mjs, check-i18n-usage.mjs, contrast-audit.mjs, bump-version.js, add-*-i18n.mjs one-offs
├── CHANGELOG.md / changelogs/    ← per-version changelogs
├── COOKIES.md                    ← cookie manager architecture
└── features.md                   ← feature matrix
```

### Major subsystems (newer, beyond plain downloading)

- **Cookies (multi-account):** Settings → Cookies tab; per-platform, per-account cookie files managed by `src-tauri/src/cookies/` (registry + storage under the data dir). Read `COOKIES.md` before touching cookies.
- **AI:** `commands/ai.rs` (config, test, summarize URL, whisper transcription, history). Providers: OpenAI, Anthropic, local OpenAI-compatible; user keys in `ai_config.json`, never logged; default unconfigured.
- **Subtitle Workshop:** `commands/subtitle_ws.rs` (load/save/translate/grammar-fix) + frontend editor.
- **Channels (tracking/following):** `commands/channels.rs` — background polling of followed channels, auto-download, notifications; frontend `tracker-*` bridges.
- **Video ops:** `commands/video_ops.rs` — AI video processing, shot detection, waveform peaks.
- **Studio / Library / Models (misc plugin):** screen recording + replay buffer, media library, AI model registry — backend in `../omniget-plugin-misc`.

### Data directories

All app data lives under `{data_dir}/wtf.tonho.omniget/`:
- `plugins/` — installed plugin DLLs and `installed.json`
- `bin/` — managed binaries (yt-dlp, ffmpeg, deno, aria2c)
- `settings.json` — Tauri store (settings)
- cookie storage managed by the cookie manager (see `COOKIES.md`); legacy `chrome-extension-cookies.txt` still supported
- `extension-metadata.json` — temporary metadata from extension (referer, headers)

On Windows: `%APPDATA%\wtf.tonho.omniget\`. On macOS: `~/Library/Application Support/wtf.tonho.omniget/`. On Linux: `~/.local/share/wtf.tonho.omniget/`. Override with `OMNIGET_DATA_DIR=/custom/path`.

## UX design principles

The behavioral baseline is enforced by the `behavioral-design` skill — invoke it for any UI work. Reference points:

- **Cognitive load (Kahneman):** support fast/intuitive (System 1) decisions; minimize options shown at once; use progressive disclosure (`<details>` collapsibles, conditional rendering).
- **Affordances (Norman):** buttons look like buttons; states are visible; feedback is immediate; errors are recoverable.
- **Recognition > recall (Nielsen):** labels on icons; history visible; autocomplete/suggestions.
- **Choice architecture (Thaler):** defaults benefit the user; quality pills before format table.
- **Habit loop (Eyal):** trigger (hotkey/clipboard/extension) → action (paste URL) → variable reward (Loop celebrates) → investment (download counter, history).
- **Anti-dark-patterns:** equal ease to accept and refuse; no fake urgency; no hidden costs; no guilt; undo is always available.

### Loop (mascot) states

Wired and rendered (PNG present in `static/mascot/`):
- `idle.png`, `downloading.png`, `error.png`, `stalled.png`, `queue.png`, `complete.png`, `amazed.png`

Mascot bubble pools live under `mascot.*` keys in `en.json` (3 variants per state, separated by `|`). `amazed` fires on first completion of the session. The mascot supports compact mode (when omnibox is non-idle) and a `bubbleText` prop. Reduced motion is respected.

## Themes

14 themes defined in `src/app.css` via `[data-theme="name"]` selectors:

- `dark` (default), `light`
- `catppuccin-mocha`, `catppuccin-macchiato`, `catppuccin-frappe`, `catppuccin-latte`
- `one-dark-pro`, `dracula`
- `nyxvamp-veil`, `nyxvamp-obsidian`, `nyxvamp-radiance`
- `eink-day`, `eink-sepia`, `eink-night`

Each theme currently defines ~74 CSS custom properties. Slimming to ~18 semantic tokens with `color-mix()` is planned but NOT done — when touching themes, keep all existing properties in sync across all 14 themes.

Settings theme grid shows 3 core cards (System/Light/Dark) with a "More themes" expansion for the rest.

When adding a theme:
1. Add `[data-theme="name"] { … }` block in `src/app.css` with ALL custom properties.
2. Add it to the theme list in `src/routes/settings/+page.svelte` (`MORE_THEMES`).
3. Apply via `document.documentElement.setAttribute("data-theme", id)` (already handled in `settings-store.svelte.ts`).

System theme support: `theme === "system"` triggers `matchMedia("(prefers-color-scheme: dark)")` listening in `+layout.svelte`.

## Plugin system

Plugins are external Rust crates compiled as `cdylib` DLLs. Their **frontends live in this main repo** (extraction to plugin repos is a future refactor).

### How it works

1. User installs from Marketplace → registry `plugins.json` is fetched from `tonhowtf/omniget-plugins` (raw.githubusercontent.com) → ZIP from GitHub Releases → unpacked to `{app_data}/plugins/{id}/`
2. On startup, `PluginManager::load_all()` loads each DLL via `libloading`.
3. DLL exports C ABI: `omniget_plugin_abi_version()` + `omniget_plugin_init()`
4. Plugin implements the `OmnigetPlugin` trait with `handle_command(command, args) -> Result<Value, String>`.
5. Frontend calls `pluginInvoke("plugin_id", "command_name", { args })` from `$lib/plugin-invoke.ts`.
6. The wrapper invokes the host's `plugin_command` Tauri command, which routes through `PluginManager::handle_command` to the DLL.

### Ecosystem (sibling folders next to this repo)

| Folder | What it is |
|---|---|
| `../omniget-plugin-courses` | Courses plugin — Hotmart, Udemy, Kiwify, Rocketseat fully implemented (~31 commands; chromiumoxide auth) |
| `../omniget-study` | Study plugin (proprietary source; binaries via `omniget-study-release`) — reader (PDF/EPUB), player, notes, flashcards, focus |
| `../omniget-plugin-telegram` | Telegram plugin — grammers MTProto, 40+ commands (login QR/phone, browse, batch download, cloning, multi-account) |
| `../omniget-plugin-convert` | Convert plugin — FFmpeg + GPU hwaccel, 4 commands |
| `../omniget-plugin-misc` | Misc utilities plugin — Studio (screen capture/replay), AI models, library, transcription/translation, tracking |
| `../omniget-plugins` | Marketplace registry (`plugins.json`) fetched by the app |
| `../omniget-plugin-template` | Scaffold for new plugins (CI for 4-platform releases) |
| `../omniget-server` | Self-hosted web downloader (Elixir/Phoenix + SvelteKit static + yt-dlp) |
| `../omnipipe` | TCP relay for P2P transfers (Elixir, room codes) |
| `../omniget-listen-relay` | WebSocket relay for Listen Together rooms (Node) |

### CRITICAL: plugin must own its own tokio runtime

Each plugin DLL statically links its own copy of tokio. The host app's tokio runtime is invisible to the DLL's tokio TLS. If a plugin uses `tokio::spawn`, `tokio::fs`, `tokio::sync::Mutex`, or any tokio I/O inside `handle_command` without its own runtime, it panics with **"there is no reactor running"**.

Every plugin MUST:
1. Create a `tokio::runtime::Runtime` in `new()`.
2. Store it as `runtime: Arc<tokio::runtime::Runtime>`.
3. In `handle_command`, clone `self.runtime.handle()` and wrap ALL async work inside `runtime_handle.spawn(async move { ... }).await`.

```rust
fn handle_command(&self, command: String, args: Value) -> Pin<Box<dyn Future<...>>> {
    let runtime_handle = self.runtime.handle().clone();
    Box::pin(async move {
        runtime_handle.spawn(async move {
            match command.as_str() { /* ... */ }
        }).await.map_err(|e| format!("task join error: {}", e))?
    })
}
```

### invoke vs pluginInvoke

- `invoke("list_plugins")` — core command, use `invoke` directly.
- `pluginInvoke("courses", "hotmart_login", args)` — plugin command, MUST use `pluginInvoke`.
- Plugin commands are NOT in `generate_handler!` — they route through the host's `plugin_command` command.
- Using `invoke("hotmart_login")` directly will FAIL silently (returns "command not found").

### Local dev workflow

```powershell
.\scripts\deploy-plugins-local.ps1   # builds DLLs, copies to %APPDATA%
cargo tauri dev                       # restart app to load new DLLs
```

Plugin DLLs go to `%APPDATA%\wtf.tonho.omniget\plugins\{id}\`. Each folder has `{crate_name}.dll` + `plugin.json`. The file `installed.json` lists enabled plugins.

## Frontend conventions

- **Svelte 5 runes everywhere:** `$state`, `$derived`, `$effect`, `$props`. No legacy reactive `$:`.
- **Component imports:** `$components/foo/Bar.svelte`; lib imports: `$lib/foo`.
- **No TypeScript `as` casts when avoidable.** Prefer narrowing via discriminated unions.
- **CSS:** custom properties everywhere (`var(--surface)`, `var(--text)`, `var(--border-radius)`). No Tailwind. No hardcoded colors. Component-scoped `<style>` blocks.
- **Theme-aware:** never hardcode `#000` or `#fff`; use `var(--text)`, `var(--bg)`, `var(--on-accent)`, etc.
- **Quick-actions pattern:** visible buttons preferred over hidden menus (Fitts' Law).
- **Settings layout:** sections with `<h5 class="section-title">`, cards with `.card`, rows with `.setting-row`.
- **Theme picker:** visual cards grid (`.theme-grid` > `.theme-card`).

## Backend conventions

- **No code comments** unless documenting WHY (a hidden constraint, a non-obvious bug fix).
- **Errors:** `anyhow` for internal flow, `?` for propagation, never `unwrap()` on external data (HTTP/JSON/user input).
- **Logging:** `tracing` (not `println!`). Don't log tokens, passwords, or cookie values — only names/metadata.
- **Async:** never `std::sync::Mutex` in async context — use `tokio::sync::Mutex`.

## Workspace

```toml
[workspace]
members = [".", "omniget-core", "omniget-plugin-sdk"]
```

NO plugin crates in this workspace. Plugins build independently from their own repos.

## Event system

Plugins emit events via `PluginHost::emit_event(name, payload)`. The frontend listens via `@tauri-apps/api/event`. The `download-listener.ts` store handles core progress/complete events.

Plugin events are namespaced — e.g., `udemy-download-progress` (with `type: "drm_warning"` payload), `udemy-download-complete` (with `drm_skipped` count).

## Browser extension

Two modes:
1. **Page detection** — recognizes supported platform URLs, colors the icon.
2. **Media sniffer** — uses `webRequest` API to detect MP4/M3U8/DASH streams in real-time.

The extension has its own independent version in `browser-extension/{chrome,firefox}/manifest.json` — NOT bumped with the app.

### Cookie priority chain

1. Extension cookie file (`chrome-extension-cookies.txt`) → `--cookies <file>`
2. Cookie manager / global cookie file (from settings) → `--cookies <file>`
3. No cookies (most public content works without)
4. `--cookies-from-browser` → opt-in only via `advanced.cookies_from_browser` setting

`--cookies-from-browser` is NOT used by default because Chrome 127+ on Windows breaks it (App-Bound Encryption). Full details: `COOKIES.md`.

## i18n

11 locales (ru added 2026-06). Library: `sveltekit-i18n`. Source of truth: `en.json`. Generated type-safe key list: `keys.ts` (run `pnpm generate:i18n-keys` after editing `en.json`). Use `scripts/sync-locales.mjs` to propagate new keys; `scripts/check-i18n-usage.mjs` to find unused keys.

Plugin-namespace keys (`courses.*`, `study.*`, `telegram.*`, `convert.*`, `hotmart.*`, `udemy.*`, `misc.*`) live here for now and will move to plugin repos when plugin frontends are extracted.

## File conventions

- Svelte: Svelte 5 runes; no TypeScript `as` casts when avoidable; one component per file.
- Rust: no comments unless documenting WHY; `anyhow` for errors; `tracing` for logs.
- CSS: CSS custom properties (`var(--surface)`, `var(--text)`, `var(--border-radius)`); no Tailwind.
- Imports: `$lib/` for lib, `$components/` for components, relative paths only within tightly-coupled subdirectories.

## Debugging plugin issues

If a plugin nav item doesn't appear or its pages show "not installed":

1. Check terminal for `Loaded plugin: <id>` or error.
2. Verify DLLs exist: `ls $env:APPDATA\wtf.tonho.omniget\plugins\<id>\`
3. Verify `installed.json` lists the plugin as enabled.
4. Rebuild and redeploy: `.\scripts\deploy-plugins-local.ps1`
5. Restart the app (DLLs only load on startup).
6. If a new command was added, the DLL MUST be rebuilt and redeployed.

### Common issues

- **"Unknown command"** → DLL is outdated; rebuild or update from Marketplace.
- **Empty plugin page** → Plugin DLL not loaded; check terminal for `[plugins]` lines.
- **"there is no reactor running"** → A function used `tokio::process::Command` outside the plugin's tokio runtime. Use `std_command` + `spawn_blocking`, or wrap the work inside `self.runtime.handle().spawn(...)`.

## Release process

Version is currently 0.6.4 (package.json, tauri.conf.json, Cargo.toml must stay in sync). Full release guide: `release-guide.md` in project memory — 8 files need bumping; tag must be on the bump commit; pushing the tag triggers the Release workflow.

## Active work

Current focus: **GitHub issue backlog** (github.com/tonhowtf/omniget/issues) — direct file downloads for plain URLs, quality tags only on video, subtitle output fixes, dependency updater, proxy coverage for all HTTP clients (marketplace/plugins included), bilibili auth for watch-later/history import, cookie manager UX (multi-select delete), douyin short links, marketplace resilience in restricted networks.

Also in progress (uncommitted): **Settings debloat / macOS rebrand** — grouped sidebar (General / Media / Integrations / Advanced), Ghost-style drill-down sub-views replacing nested `<details>` (`src/components/settings/downloads/*Section.svelte`, `SettingsDrillItem`/`SettingsDrillBack`), search that filters+highlights the sidebar, dependencies as a table, and an "honest settings" pass that removed UI for fields the backend ignores (kept in schema; re-wire via `.claude/handoffs/settings-dead-fields-backend.md` before re-exposing — `skip_existing` already wired to yt-dlp `--no-overwrites`).

Done (formerly listed as active): league/social removal, `/about/debug` gating, marketplace console.log cleanup + state refresh, theme grid restructure (3 visible + More themes), mascot `amazed` wiring, settings restructure.

Out of scope: extracting plugin frontends to their repos (future refactor); theme token slimming (planned, not started).
