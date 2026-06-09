## Context

mimo2codex boots a single Node process whose runtime configuration comes from three layers, in increasing priority:

1. Built-in defaults (compiled in `src/config.ts`).
2. Process environment (populated by `.env` via `scripts/load-env.sh` or the desktop launcher, and by the user's shell).
3. CLI flags (`--port`, `--host`, `--data-dir`, `--auth`, ...).

The admin UI today adds a fourth layer for a small set of toggles: the `settings` sqlite table (`src/db/settings.ts`) accessed through `GET/PUT/DELETE /admin/api/settings/:key`. The boot-time resolvers in `server.ts` and `config.ts` (e.g. `resolveDisableThinking`, `resolveSilentRewrite`) implement a CLI > env > settings-DB > default precedence and re-read the DB on every change, so the UI can take effect without restart for those few keys. For everything else the user must hand-edit `.env` and restart.

The json method (`providers.json`) bypasses the env/CLI chain entirely: it is a curated file edited through the admin UI (`GET/PUT /admin/api/generic-providers`) and loaded fresh at every request via `locateProvidersFile()` + `readSpecsFromFile()`. The env-synthesized `generic` provider is a third path that lives in `loadGenericProviders()` (`src/providers/genericLoader.ts:259-280`).

The proposed change widens the env surface that the UI can edit, while keeping the env file as the operator's source of truth and the settings DB as a UI-friendly override layer.

## Goals / Non-Goals

**Goals**

- Every non-secret `MIMO2CODEX_*` env var, plus `HTTPS_PROXY` / `HTTP_PROXY` / `NO_PROXY`, is editable from a new **Settings -> Environment** page in the admin SPA, with the currently-resolved value, the value source, and inline validation.
- The `GENERIC_*` single-instance shortcut is rendered as a `generic` provider entry in the existing Providers page, with an "from env" badge when the entry is env-synthesized.
- The CLI > env > settings-DB > default precedence is implemented uniformly for every new key, not just the three that already have it.
- The secret policy (`MIMO_API_KEY`, `DS_API_KEY`, `DEEPSEEK_API_KEY`, `MIMO2CODEX_MASTER_KEY`) stays closed: the UI never renders an input for them and writes are rejected by `forbidden_setting`.
- A consolidated "restart required" banner tracks every env-backed key the user has changed, since most of these cannot take effect without a process restart.

**Non-Goals**

- Writing back to the user's `.env` file. The UI persists to the settings DB only; editing the env file remains a manual operator step. (Rationale: env-file editing has sharp edges -- escaping, atomicity, permissions, BOMs, comments; the existing settings-DB-with-env-override pattern is already proven and matches the three toggles that already work this way.)
- Exposing the desktop-launcher's internal env (`MIMO2CODEX_DESKTOP_PARENT=1`, etc.) -- those are framework concerns, not operator config.
- Live (no-restart) reload of port, host, or proxy. The proxy agent and the listening socket are initialized once; flipping these requires a restart, and the UI says so.
- Removing or replacing the json method. The Providers page keeps its current shape; we only add the env-synthesized `generic` row.

## Decisions

### 1. Settings DB as the override layer, env as the operator source of truth

- **Decision**: UI writes go to the `settings` table; the boot path reads `process.env` first and falls back to the settings DB. The settings DB never silently shadows a real env var; env wins.
- **Rationale**: this matches the three resolvers that already work (`disableThinking`, `silentRewrite`, `logBodyMode`). It also keeps the operator's `.env` file authoritative: an admin who rotates the `MIMO2CODEX_PORT` env var and restarts gets the env value, not a stale UI value.
- **Alternative considered**: write UI changes back to the `.env` file. Rejected: file editing is fragile (CRLF, BOM, quoting, comments), and the user explicitly chose env mode -- the file is their source of truth, not ours.

### 2. Allowlist, not blocklist, for new keys

- **Decision**: introduce a typed `ENVSETTINGS_KEYS` allowlist in `src/config.ts` (e.g. `["mimo2codex.host", "mimo2codex.port", ...]`). The new endpoints only accept keys in this allowlist. `isForbiddenSettingKey` continues to reject the secret keys as defense-in-depth.
- **Rationale**: an allowlist is the only way to keep the `MIMO2CODEX_MASTER_KEY` / `MIMO_API_KEY` / `DS_API_KEY` / `DEEPSEEK_API_KEY` story bulletproof. A blocklist would silently let through any new env var we forget to enumerate.
- **Alternative considered**: extend `isForbiddenSettingKey` to be the only gate. Rejected: it inverts the safety property; the secret set is much smaller than the env set, so an allowlist is the right shape.

### 3. Unified resolver `resolveEnvSetting(key)`

- **Decision**: a single `resolveEnvSetting(key: string, parsed: ParsedArgs, env: NodeJS.ProcessEnv, settings: SettingsStore): { value: string | null; source: "cli" | "env" | "settings" | "default" | "unset" }` helper, consumed at boot by `loadConfig()`. The three existing bespoke resolvers (`resolveDisableThinking` etc.) are rewritten on top of it.
- **Rationale**: today the precedence is copy-pasted three times. Unifying it gives every new key the same semantics for free and keeps the precedence auditable in one place.
- **Alternative considered**: per-key resolvers. Rejected for the reasons above; kept as a thin wrapper for the keys whose value needs coercion (e.g. `MIMO2CODEX_PORT` -> number).

### 4. `synthesizedFromEnv` flag on the generic provider runtime

- **Decision**: extend `ProviderRuntime` with `synthesizedFromEnv: boolean`; `loadGenericProviders()` sets it to `true` for the env shortcut and `false` for json-declared providers. The Providers page renders a small antd `<Tag color="blue">from env</Tag>` next to the id when true.
- **Rationale**: keeps the boot path single-sourced (the env shortcut is still loaded by the same loader) but gives the UI a one-bit signal to render the badge and to offer the right "delete" semantics (clearing the env override, not deleting a json entry).
- **Alternative considered**: re-implement env loading in the UI. Rejected: the UI is a thin client; the loader is the source of truth.

### 5. Consolidated restart banner, per-key restarts

- **Decision**: the Settings page lists every key whose UI value differs from the boot-time resolved value. The existing `RestartRequiredBanner` component gains a `source: "env-settings"` variant and a "show details" affordance that links into the Settings page. A per-key "Restart needed" pill is shown next to each modified row.
- **Rationale**: keeps the user from losing track of pending changes. Most keys are not hot-reloadable (port, host, proxy, dataDir), so a single banner is more honest than pretending every save takes effect immediately.
- **Alternative considered**: auto-trigger a restart on save. Rejected: restart is destructive (in-flight requests dropped), and the operator may want to bundle several edits.

### 6. No new dependencies

- **Decision**: implement the Settings page with antd Form (already in use across the SPA), reuse `ProviderFormModal` for the env-synthesized `generic` entry, reuse the existing `AppConfigContext` for state, reuse `setSetting` / `deleteSetting` from the existing API.
- **Rationale**: zero new dependencies matches the rest of the SPA. The only net-new file is `Settings.tsx`; everything else is glue.

## Risks / Trade-offs

- **[Risk]** Admin sets a settings-DB value that silently loses to a real env var on restart. -> **Mitigation**: the Settings page renders the precedence for every key (small "source" badge: `cli` / `env` / `settings` / `default`); tooltip on each input explains that env wins at boot.
- **[Risk]** A `MIMO2CODEX_PROVIDERS_FILE` mis-edit could break providers.json loading. -> **Mitigation**: `MIMO2CODEX_PROVIDERS_FILE` is **excluded** from the allowlist. It is a structural key, not an operator setting; it stays a CLI / env-only thing.
- **[Risk]** A user edits `MIMO2CODEX_COOKIE_SECURE` while HTTPS is not in front, locks themselves out. -> **Mitigation**: the field has a confirm modal with a "I have HTTPS termination in front of mimo2codex" checkbox, matching the `README.zh.md` warning.
- **[Risk]** A user edits `HTTPS_PROXY` and the running process keeps using the old undici ProxyAgent. -> **Mitigation**: the proxy group banner is highlighted and lists proxy keys under "restart required". The Settings page tooltip says "the outbound proxy agent is initialized once at startup".
- **[Risk]** `MIMO2CODEX_MASTER_KEY` rotation via the UI would invalidate every stored BYOK ciphertext. -> **Mitigation**: `MIMO2CODEX_MASTER_KEY` is `forbidden_setting`; it is **excluded** from the allowlist. The UI never renders an input for it; a tooltip explains why.
- **[Risk]** Env-synthesized `generic` provider collides with a json-declared provider with the same id. -> **Mitigation**: boot behavior is unchanged (env wins, json entry is dropped with a startup warning that we already log). The UI surfaces a single warning badge on the `generic` row when both are present, pointing at the boot log.
- **[Risk]** Restart banner becomes noisy if a user toggles several keys. -> **Mitigation**: the banner groups keys by section, mirrors the page sections, and offers "dismiss until next change". The count is shown in the AppHeader alongside any existing restart indicators.

## Migration Plan

- **Deploy**: no data migration; the new endpoints are additive. The settings DB schema is unchanged. Existing `.env` deployments behave exactly as before.
- **First-time use**: the Settings page lazily initializes its allowlist rows in the `settings` table only when the user actually edits a value. Empty-state shows the current resolved values read from env, with an "Edit" affordance that materializes the settings-DB row.
- **Rollback**: removing the new endpoints and the page route restores prior behavior. The settings DB rows they wrote are inert keys that the boot resolvers ignore.
- **Upgrade prompt**: a single `ReleaseHighlight` on the next version's "What's new" modal in `web/src/release-notes.tsx` linking to the new Settings page; bilingual (`en` + `zh`) entries.

## Open Questions

- Should `MIMO2CODEX_NO_PROXY_FROM_ENV` (boolean, currently CLI-only via `--no-load-env` and `MIMO2CODEX_NO_LOAD_ENV`) be in the allowlist? It's a kill-switch for the proxy layer; the safest default is **yes**, with a confirm modal. -> Confirm during design.
- Should the Settings page offer a "copy as .env" button that renders the current settings-DB values as a `.env` snippet for the operator to paste into their file? Useful for hand-off to ops; deferred unless the user asks.
- The existing `/admin/api/setup-snippets` endpoint already prints ready-to-paste TOML / auth.json. Could the new Settings page link to it for "export to Codex"? Deferred.
