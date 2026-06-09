# Comet Design Handoff

- Change: env-and-json-ui-config
- Phase: design
- Mode: compact
- Context hash: e84d75f26d95d35134ee4a4b44bbdf4f75bf9e5827b6431a4452b4d9cd5b251e

Generated-by: comet-handoff.sh

OpenSpec remains the canonical capability spec. This handoff is a deterministic, source-traceable context pack, not an agent-authored summary.

## openspec/changes/env-and-json-ui-config/proposal.md

- Source: openspec/changes/env-and-json-ui-config/proposal.md
- Lines: 1-55
- SHA256: 2c09632d826c7efffc277e8ebe11b21db39f787c9e6e6bf64f5bf59ad695fdb9

```md
## Why

mimo2codex today has two parallel ways to declare provider configuration:

- **env method** -- values live in the shell / `.env` file (MIMO_API_KEY, DS_API_KEY, GENERIC_*, MIMO2CODEX_*).
- **json method** -- values live in `providers.json`, edited through the admin UI at `/admin/providers` (`GET/PUT /admin/api/generic-providers`).

The json method has full UI parity: a Provider form, validation, atomic write, restart banner. The env method does not. Out of roughly 25 env vars the proxy recognizes, only six (`ui.theme`, `ui.lang`, `disableThinking`, `silentRewrite`, `logBodyMode`, `logRetentionDays`) can be set from the admin UI, and the rest -- including the `GENERIC_*` single-instance shortcut, `MIMO2CODEX_PORT`, `MIMO2CODEX_HOST`, `MIMO2CODEX_DATA_DIR`, `MIMO2CODEX_DEFAULT_PROVIDER`, `MIMO2CODEX_CONTEXT_OVERFLOW_MODE`, `MIMO2CODEX_COOKIE_SECURE`, `MIMO2CODEX_MASTER_KEY`, `MIMO2CODEX_NO_PROXY_FROM_ENV`, `HTTPS_PROXY` / `HTTP_PROXY` / `NO_PROXY` -- must still be hand-edited in `.env` and require a manual restart.

Secrets (`MIMO_API_KEY`, `DS_API_KEY`, `DEEPSEEK_API_KEY`) are intentionally rejected by the current `/admin/api/settings/:key` endpoint (`isForbiddenSettingKey` in `src/admin/router.ts`). That policy stays.

The opportunity: bring the env method up to json-method parity, so operators can do "configure everything from one UI" without losing the existing secret-handling guarantees.

## What Changes

- Add a new **Settings -> Environment** page in the admin SPA that lists every recognized env var (grouped by section: Server, Auth, Logging, Proxy, Advanced), shows its currently-resolved value, and lets the user edit non-secret ones inline. The page surfaces the value source (CLI flag > process env > settings DB > default) for every key.
- Extend the existing **Providers** page to render the `generic` single-instance shortcut the same way it renders a json-declared generic provider: when `GENERIC_API_KEY` / `GENERIC_BASE_URL` / `GENERIC_DEFAULT_MODEL` are present in the process env, the Providers page shows one entry with id `generic` and a small badge "from env". Editing it writes back to the settings DB (env-overriding path), and the on-disk `.env` is left untouched.
- Add a new admin API surface that walks CLI > env > settings DB > default for any of the recognized env-backed keys, so the UI can render a single "current value" per key. The settings DB row continues to take effect at boot via the existing `resolveDisableThinking`-style helpers; for the new keys we add equivalent resolvers in `src/config.ts`.
- Persist writes through the existing `setSetting` / `deleteSetting` path; add the new keys to a per-key allowlist so we keep the secret-rejection policy tight. Save still requires a restart -- the page surfaces a single "restart required" banner that consolidates all pending changes.
- Keep secret handling exactly as today: `MIMO_API_KEY` / `DS_API_KEY` / `DEEPSEEK_API_KEY` and the per-user BYOK ciphertexts remain `forbidden_setting`; the UI never renders an input for them and explains why in a tooltip.

This is **not** a breaking change. Existing `.env`-driven deployments keep working unchanged; deployments that already edit settings via UI keep their current settings rows; the new env method UI is strictly additive.

## Capabilities

### New Capabilities

- `env-runtime-settings`: read/write non-secret env-backed runtime configuration (`MIMO2CODEX_*` and the proxy family) from the admin UI, with a unified CLI > env > settings-DB > default resolver, restart-required gating, and an explicit deny-list for secret keys.
- `generic-env-provider`: surface the `GENERIC_*` single-instance env shortcut as a first-class provider in the Providers UI, alongside json-declared generic providers, with edit + delete (where delete = clear the env override and fall back to a json entry, if one exists, or remove the provider).

### Modified Capabilities

None. The existing `providers-json-management` surface (already supported by the Providers page) keeps its current shape; this change only adds the `generic` entry rendering when the env shortcut is in use.

## Impact

- **Backend** (`src/`):
  - `src/admin/router.ts` -- new endpoints `GET /admin/api/env-settings` (resolved view), `PUT /admin/api/env-settings/:key`, `DELETE /admin/api/env-settings/:key`, plus extending `GET /admin/api/generic-providers` to include the env-synthesized `generic` entry.
  - `src/config.ts` -- new resolver helpers `resolveEnvSetting(key)` and `resolveGenericProviderFromEnv()` consumed at boot.
  - `src/providers/genericLoader.ts` -- `loadGenericProviders()` already produces a generic provider from `GENERIC_*`; add a `synthesizedFromEnv` flag on the runtime so the UI can badge it.
  - `src/db/settings.ts` -- no schema change; add a typed allowlist helper for the new keys.
- **Frontend** (`web/`):
  - `web/src/pages/Settings.tsx` (new) -- the env settings page.
  - `web/src/pages/providers/` -- extend the providers list + form to render the `generic` env provider; the existing `ProviderFormModal` is reused for editing.
  - `web/src/api/client.ts` -- new typed client methods.
  - `web/src/i18n/locales/{en-US,zh-CN}/settings.json` (new) -- bilingual labels.
  - `web/src/release-notes.tsx` -- new `ReleaseHighlight` once shipped.
- **Tests** (`test/`):
  - New unit tests for `resolveEnvSetting` precedence and for the `forbidden_setting` allowlist.
  - New API tests for the three new endpoints.
  - New UI test for the env-badged `generic` provider entry.
- **Docs**:
  - `README.md` / `README.zh.md` -- short section pointing at the new UI page; no env-var tables duplicated.
  - `doc/tag-log.md` + `doc/tag-log.zh.md` -- `[new]` entry under the upcoming version block.
- **Out of scope**: writing back to the user's `.env` file (the UI only writes to the settings DB; env-file editing remains a deliberate boundary).
```

## openspec/changes/env-and-json-ui-config/design.md

- Source: openspec/changes/env-and-json-ui-config/design.md
- Lines: 1-90
- SHA256: 5ee3940904e2cde7059738f93f2e0dc2ce4bb0c4c8121c5189deadc6d22e9f4d

[TRUNCATED]

```md
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

```

Full source: openspec/changes/env-and-json-ui-config/design.md

## openspec/changes/env-and-json-ui-config/tasks.md

- Source: openspec/changes/env-and-json-ui-config/tasks.md
- Lines: 1-54
- SHA256: 37961dbd0676066a57e2d2dc97c7e6fce0ceddf4567f688c6313884c10cc8304

```md
## 1. Backend: allowlist and unified resolver

- [ ] 1.1 Add `ENVSETTINGS_KEYS` typed allowlist in `src/config.ts` covering `MIMO2CODEX_HOST`, `MIMO2CODEX_PORT`, `MIMO2CODEX_DATA_DIR`, `MIMO2CODEX_DEFAULT_PROVIDER`, `MIMO2CODEX_NO_REASONING`, `MIMO2CODEX_VERBOSE`, `MIMO2CODEX_NO_ADMIN`, `MIMO2CODEX_CONTEXT_OVERFLOW_MODE`, `MIMO2CODEX_COOKIE_SECURE`, `MIMO2CODEX_NO_PROXY_FROM_ENV`, `HTTPS_PROXY`, `HTTP_PROXY`, `NO_PROXY`, plus the `generic.*` namespace (excluding `generic.apiKey`). Document each key with its kebab-case settings-DB id, its env-var name, its type, and a short description sourced from `.env.example`.
- [ ] 1.2 Implement `resolveEnvSetting(key, parsed, env, settings): { value, source }` in `src/config.ts`. The helper walks `CLI flag > process env > settings DB > default` and returns the source tag.
- [ ] 1.3 Rewrite the three existing bespoke resolvers in `server.ts` (`resolveDisableThinking`, `resolveSilentRewrite`, plus the `logBodyMode` / `logRetentionDays` equivalents) on top of `resolveEnvSetting`. Existing observable behavior must be unchanged; add a regression test for each.
- [ ] 1.4 Add `ENVSETTINGS_KEYS` to a deny-list check that runs **after** the allowlist. The deny-list contains `MIMO_API_KEY`, `DS_API_KEY`, `DEEPSEEK_API_KEY`, `MIMO2CODEX_MASTER_KEY`, `MIMO2CODEX_PROVIDERS_FILE`. A key in BOTH lists SHALL be rejected as `forbidden_setting` (deny-list wins).
- [ ] 1.5 Wire `cookieSecure`, `noProxyFromEnv`, `contextOverflowMode`, and `verbose` through `loadConfig()` using `resolveEnvSetting`, and surface them on `Config` so `server.ts` and the banner code see the resolved values.

## 2. Backend: env-settings API endpoints

- [ ] 2.1 In `src/admin/router.ts`, add `GET /admin/api/env-settings` that returns `[{ key, envVar, section, value, source, description, secret: boolean }, ...]`. The handler iterates `ENVSETTINGS_KEYS`, resolves each value with `resolveEnvSetting`, and serializes the source as a small string.
- [ ] 2.2 Add `PUT /admin/api/env-settings/:key` that accepts `{ value: string }`, validates the key is in `ENVSETTINGS_KEYS` AND not in the deny-list, coerces the value to the right type (number, boolean, enum), and persists via `setSetting`. Return `400 forbidden_setting` with a clear message for deny-list hits.
- [ ] 2.3 Add `DELETE /admin/api/env-settings/:key` that removes the row via `deleteSetting`. Return `404` if the row is absent.
- [ ] 2.4 Extend `GET /admin/api/generic-providers` to include a single synthesized entry when the env shortcut is in use. The entry has `id: "generic"`, `synthesizedFromEnv: true`, and is placed **before** any json-declared entries with a clear `source: "env"` marker.
- [ ] 2.5 When the env shortcut is in use AND a json `generic` entry exists, return a `warning` field on the env-synthesized entry explaining that env wins at boot. The UI uses this to render the warning badge.
- [ ] 2.6 Add a `DELETE /admin/api/generic-providers/env` endpoint that clears the env override path. The handler removes the `generic.*` rows from the settings DB and logs a "generic provider removed; restart to apply" warning. Return `404` if no env-synthesized entry exists.

## 3. Frontend: Settings page

- [ ] 3.1 Add `web/src/pages/Settings.tsx` with an antd `Form` per section (Server, Auth, Logging, Proxy, Advanced). Each row is a labelled input with a small `source` badge and a tooltip that links to the description in `.env.example`.
- [ ] 3.2 Add `web/src/i18n/locales/en-US/settings.json` and `web/src/i18n/locales/zh-CN/settings.json` with bilingual labels for every key in `ENVSETTINGS_KEYS` plus the section headings.
- [ ] 3.3 Wire the page to `GET /admin/api/env-settings` on mount, debounce input changes, and call `PUT /admin/api/env-settings/:key` on save. A "Reset" button per row calls `DELETE`.
- [ ] 3.4 Render a "Secrets (set in `.env`)" group at the bottom of the page listing the four secret keys as read-only rows with a tooltip explaining the policy.
- [ ] 3.5 Add the Settings page to the SPA router (`web/src/App.tsx` or equivalent) and to the navigation (`web/src/components/AppHeader.tsx`) next to "Providers" / "Models".
- [ ] 3.6 Hook the page into `AppConfigContext` so the existing `RestartRequiredBanner` picks up `source: env-settings` items and the AppHeader shows the consolidated count.

## 4. Frontend: Providers page extension for env generic

- [ ] 4.1 In `web/src/pages/providers/`, extend the list to render the env-synthesized `generic` row when present. Use a small antd `<Tag color="blue">from env</Tag>` next to the id.
- [ ] 4.2 When the entry has the `warning` field set (env + json collision), render a `<Tooltip>`-wrapped `<ExclamationCircleOutlined />` next to the badge.
- [ ] 4.3 Reuse the existing `ProviderFormModal` for editing. On save, transform the form values into `generic.*` settings-DB rows and call `PUT /admin/api/env-settings/:key` for each field.
- [ ] 4.4 The "Delete" button on the env-synthesized row calls `DELETE /admin/api/generic-providers/env` after a confirm modal that warns the operator the change requires a restart and does not edit the `.env` file.

## 5. Tests

- [ ] 5.1 In `test/`, add a `resolveEnvSetting.test.ts` that covers CLI > env > settings-DB > default for at least one boolean, one number, one string, and one enum-typed key.
- [ ] 5.2 Add a `forbiddenSettingAllowlist.test.ts` that asserts every deny-list key returns `400 forbidden_setting` from the new endpoints, even if a maintainer accidentally adds the key to the allowlist.
- [ ] 5.3 Add `envSettingsEndpoints.test.ts` that exercises `GET /admin/api/env-settings`, `PUT`, and `DELETE` for a representative key, including type coercion errors and out-of-allowlist rejections.
- [ ] 5.4 Add `genericProvidersEnv.test.ts` that asserts the `synthesizedFromEnv` flag is set for env-synthesized providers and not for json-declared ones, and that the collision `warning` field is set when both are present.
- [ ] 5.5 Add a Playwright UI test (`web/src/__tests__/settingsPage.spec.ts` or equivalent) that opens the Settings page, edits a non-secret key, and asserts the "restart required" banner appears; a second test edits the env-synthesized `generic` row and asserts the badge is shown.

## 6. Docs and release notes

- [ ] 6.1 In `README.md` and `README.zh.md`, add a short subsection under "Configuration" pointing operators at the new **Settings -> Environment** page; do NOT duplicate the env-var table (it stays in `.env.example`).
- [ ] 6.2 In `doc/tag-log.md` and `doc/tag-log.zh.md`, append a `[new]` entry under the current upcoming-version block describing the Settings page and the env-synthesized `generic` row.
- [ ] 6.3 In `web/src/release-notes.tsx`, add a `ReleaseHighlight` with bilingual `title` + `description`, `kind: "new"`, and `location: "Settings"` for the Settings page; add a second `ReleaseHighlight` for the env-synthesized `generic` row in the Providers page (`location: "Providers"`).

## 7. Validation

- [ ] 7.1 Run `npm run build` and `npm run web:build` and confirm both compile clean.
- [ ] 7.2 Run `npm test` and confirm all new and existing tests pass.
- [ ] 7.3 Start the dev server with `npm run dev`, open `/admin/settings`, edit `MIMO2CODEX_PORT`, restart, and confirm the proxy listens on the new port.
- [ ] 7.4 Start the dev server with `GENERIC_API_KEY=... GENERIC_BASE_URL=...`, open `/admin/providers`, and confirm the "from env" badge is shown on the `generic` row.
- [ ] 7.5 Verify the four secret keys (`MIMO_API_KEY`, `DS_API_KEY`, `DEEPSEEK_API_KEY`, `MIMO2CODEX_MASTER_KEY`) are rendered as read-only rows on the Settings page with the policy tooltip.
```

## openspec/changes/env-and-json-ui-config/specs/env-runtime-settings/spec.md

- Source: openspec/changes/env-and-json-ui-config/specs/env-runtime-settings/spec.md
- Lines: 1-81
- SHA256: 3e8bee7e807273f942c3c6d16ef9afea0a5c1ecc02bdb13b531a845553d44e0e

[TRUNCATED]

```md
## ADDED Requirements

### Requirement: Settings page lists every recognized non-secret env-backed key

The admin SPA SHALL render a "Settings -> Environment" page that lists every env-backed runtime key the proxy recognizes, grouped by section (Server, Auth, Logging, Proxy, Advanced). For each key the page SHALL display the currently-resolved value, the value source (`cli`, `env`, `settings`, `default`, or `unset`), and a short description sourced from `.env.example`.

#### Scenario: All recognized keys visible on first load
- **WHEN** the operator opens the Settings -> Environment page with a fresh admin session
- **THEN** the page renders one row per recognized non-secret env-backed key
- **AND** each row shows the resolved value, the source badge, and the description

#### Scenario: Secret keys are not rendered as inputs
- **WHEN** the Settings page is rendered
- **THEN** `MIMO_API_KEY`, `DS_API_KEY`, `DEEPSEEK_API_KEY`, and `MIMO2CODEX_MASTER_KEY` are listed in a read-only "Secrets (set in `.env`)" group
- **AND** none of those rows expose an input, an "Edit" affordance, or a copy-to-clipboard affordance
- **AND** a tooltip explains that secret keys must be set in the env file

### Requirement: Non-secret env-backed keys are editable from the Settings page

The system SHALL persist edits to non-secret env-backed keys through the existing `settings` sqlite table. A successful edit SHALL require a process restart to take full effect for keys whose value is captured at boot (host, port, dataDir, defaultProvider, masterKey, cookieSecure, proxy, no-proxy-from-env, context-overflow-mode).

#### Scenario: Operator changes the listening port
- **WHEN** the operator types `9000` into the `MIMO2CODEX_PORT` row and saves
- **THEN** the system stores the value in the `settings` table under key `mimo2codex.port`
- **AND** the Settings page marks the row as "restart required"
- **AND** the AppHeader shows a consolidated restart-required banner

#### Scenario: Operator reverts a change
- **WHEN** the operator clicks "Reset" on a row whose current settings-DB value is `9000` and the running env value is `8988`
- **THEN** the system deletes the `mimo2codex.port` row from the settings table
- **AND** the row reverts to the resolved value (`8988`) and the source badge becomes `env`
- **AND** the restart-required banner updates to remove the key

### Requirement: Unified CLI > env > settings-DB > default precedence

For every env-backed runtime key the system SHALL resolve the effective value using the precedence `CLI flag > process env > settings DB > built-in default`, and SHALL report the source of the resolved value. The three existing bespoke resolvers (`disableThinking`, `silentRewrite`, `logBodyMode`) SHALL be rewritten on top of the unified resolver; their externally observable behavior SHALL remain unchanged.

#### Scenario: Env value shadows a settings-DB value at boot
- **WHEN** `MIMO2CODEX_PORT=8988` is set in the process env
- **AND** the `settings` table has a row `mimo2codex.port=9000`
- **THEN** the resolved value at boot is `8988` and the source badge is `env`
- **AND** the settings-DB row remains untouched

#### Scenario: Settings-DB value used when env is unset
- **WHEN** `MIMO2CODEX_PORT` is unset in the process env
- **AND** the `settings` table has a row `mimo2codex.port=9000`
- **THEN** the resolved value at boot is `9000` and the source badge is `settings`

#### Scenario: Built-in default used when neither env nor settings has a value
- **WHEN** `MIMO2CODEX_PORT` is unset in the process env
- **AND** the `settings` table has no row for `mimo2codex.port`
- **THEN** the resolved value is the compiled default (`8988`) and the source badge is `default`

### Requirement: Allowlist of UI-editable env-backed keys

The system SHALL maintain a typed `ENVSETTINGS_KEYS` allowlist in `src/config.ts`. The new admin API endpoints SHALL accept only keys from this allowlist. `MIMO_API_KEY`, `DS_API_KEY`, `DEEPSEEK_API_KEY`, `MIMO2CODEX_MASTER_KEY`, and `MIMO2CODEX_PROVIDERS_FILE` SHALL be excluded from the allowlist and SHALL continue to be rejected as `forbidden_setting`.

#### Scenario: Out-of-allowlist key is rejected
- **WHEN** the admin API receives `PUT /admin/api/env-settings/mimo2codex.providers-file` with body `{ "value": "/tmp/x.json" }`
- **THEN** the endpoint responds with `400 forbidden_setting` and a message that this key is operator-only

#### Scenario: Allowed key round-trips
- **WHEN** the admin API receives `PUT /admin/api/env-settings/mimo2codex.port` with body `{ "value": "9000" }`
- **THEN** the endpoint writes the row and responds with `200 { "key": "mimo2codex.port", "value": "9000" }`

#### Scenario: Secret key is rejected even if a future maintainer adds it to the allowlist
- **WHEN** a key is in BOTH the allowlist and the secret deny-list
- **THEN** the secret deny-list wins and the endpoint responds with `400 forbidden_setting`

### Requirement: Restart-required gating for boot-captured keys

The system SHALL mark any env-backed key whose value is captured at boot as "restart required" whenever its UI value diverges from the running resolved value. The existing `RestartRequiredBanner` SHALL be reused with a new `source: env-settings` variant.

#### Scenario: Editing a non-hot-reloadable key triggers the banner
- **WHEN** the operator edits `MIMO2CODEX_PORT` in the Settings page
- **THEN** the banner shows one item: `mimo2codex.port (8988 -> 9000, restart required)`

#### Scenario: Banner clears when the value reverts
- **WHEN** the operator resets a previously edited key and the new resolved value matches the running value
- **THEN** the banner removes that key
```

Full source: openspec/changes/env-and-json-ui-config/specs/env-runtime-settings/spec.md

## openspec/changes/env-and-json-ui-config/specs/generic-env-provider/spec.md

- Source: openspec/changes/env-and-json-ui-config/specs/generic-env-provider/spec.md
- Lines: 1-91
- SHA256: 0538170e9b25084f9279f1c18951a027e3e0440c3f89a1f71451f7a3fa2a2079

[TRUNCATED]

```md
## ADDED Requirements

### Requirement: Env-synthesized generic provider is rendered in the Providers list

When `GENERIC_API_KEY` or `GENERIC_BASE_URL` is present in the process env, the admin SPA SHALL render a single row in the Providers list with `id = "generic"`, marked as env-synthesized. The row SHALL be editable through the existing `ProviderFormModal` form.

#### Scenario: Env shortcut in use, no json entry
- **WHEN** `GENERIC_API_KEY=sk-xxx` and `GENERIC_BASE_URL=https://example.com/v1` are set
- **AND** `providers.json` has no entry with id `generic`
- **THEN** the Providers list shows exactly one row with `id = "generic"`
- **AND** the row shows a "from env" badge
- **AND** the existing runtime model is unchanged (loader already produces this provider at boot)

#### Scenario: Env shortcut in use, json entry also present
- **WHEN** the env shortcut is set
- **AND** `providers.json` contains a `generic` entry
- **THEN** the Providers list shows the env-synthesized row
- **AND** a warning indicator explains that the env value takes precedence at boot
- **AND** the json entry is hidden from the list (env wins)

#### Scenario: Env shortcut absent, json entry present
- **WHEN** no `GENERIC_*` env vars are set
- **AND** `providers.json` contains a `generic` entry
- **THEN** the Providers list shows the json entry as a normal row
- **AND** no "from env" badge is shown

### Requirement: `synthesizedFromEnv` flag on the provider runtime

The system SHALL expose a `synthesizedFromEnv: boolean` flag on each `ProviderRuntime` surfaced through the admin API. The flag SHALL be `true` for the env-synthesized `generic` provider and `false` for every other provider, including json-declared generic providers.

#### Scenario: Generic provider loaded from env has the flag set
- **WHEN** the loader produces a generic provider from `GENERIC_*` env vars
- **THEN** `GET /admin/api/generic-providers` returns that provider with `synthesizedFromEnv: true`

#### Scenario: Generic provider loaded from json does not have the flag set
- **WHEN** the loader produces a generic provider from a `providers.json` entry
- **THEN** the returned provider has `synthesizedFromEnv: false`

### Requirement: Editing the env-synthesized generic provider writes to the settings DB

When the operator edits the env-synthesized `generic` provider in the UI, the system SHALL persist the form values to the settings DB under the `generic.*` namespace, leaving the on-disk `.env` file untouched. The boot path SHALL resolve `GENERIC_*` values with the same CLI > env > settings-DB > default precedence as every other env-backed key.

#### Scenario: Operator edits the generic base URL via the form
- **WHEN** the operator changes `baseUrl` to `https://new.example.com/v1` in the form
- **AND** saves the change
- **THEN** the system writes `generic.baseUrl=https://new.example.com/v1` to the settings DB
- **AND** the running process continues to use the env value until restart
- **AND** on restart, the new value is used (env still wins if set; otherwise settings-DB)

#### Scenario: Operator reverts a generic form value
- **WHEN** the operator clears a `generic.*` row in the settings DB
- **THEN** the env value (or the json value) takes over for that field on the next boot

### Requirement: Deleting the env-synthesized generic provider clears the env override path

When the operator clicks "Delete" on the env-synthesized `generic` row, the system SHALL remove the `generic.*` settings-DB rows AND the env-synthesized entry from the running registry. The `.env` file SHALL be left untouched. If a json-declared `generic` entry exists, it SHALL take over after restart.

#### Scenario: Delete clears the env-synthesized entry from the runtime
- **WHEN** the operator clicks "Delete" on the env-synthesized `generic` row
- **THEN** the system removes the `generic.*` rows from the settings DB
- **AND** the Providers list no longer shows the env-synthesized row
- **AND** the `.env` file is not modified
- **AND** the running proxy logs a "generic provider removed; restart to apply" warning

#### Scenario: Delete with a json fallback leaves the json entry intact
- **WHEN** the operator deletes the env-synthesized `generic` row
- **AND** `providers.json` has a `generic` entry
- **THEN** the next boot loads the json entry as a normal `generic` provider
- **AND** the "from env" badge is not shown on the json-sourced row

### Requirement: Collision warning surfaces the boot-time precedence

The Providers page SHALL surface a single warning indicator on the `generic` row when the env shortcut AND a json entry are both present, explaining that env wins at boot and pointing the operator to the boot log.

#### Scenario: Both env and json declare a generic provider
- **WHEN** the Providers page loads with both sources present
- **THEN** the `generic` row shows the warning indicator next to the "from env" badge
- **AND** a tooltip explains "env wins at boot; the json entry is ignored until the env shortcut is removed"

### Requirement: New `generic.*` keys are in the env-settings allowlist
```

Full source: openspec/changes/env-and-json-ui-config/specs/generic-env-provider/spec.md

