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
