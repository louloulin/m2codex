---
comet_change: env-and-json-ui-config
role: technical-design
canonical_spec: openspec
---

# env-and-json-ui-config -- Technical Design

> Canonical source of truth for **what** and **why** is the OpenSpec change at
> `openspec/changes/env-and-json-ui-config/`. This document is the
> implementation reference for **how**; it deliberately does not restate
> requirements, scenarios, or non-goals -- those live in `proposal.md`,
> `design.md`, and `specs/*/spec.md` under that change.

## Source documents (read first)

- `openspec/changes/env-and-json-ui-config/proposal.md`
- `openspec/changes/env-and-json-ui-config/design.md`
- `openspec/changes/env-and-json-ui-config/specs/env-runtime-settings/spec.md`
- `openspec/changes/env-and-json-ui-config/specs/generic-env-provider/spec.md`
- `openspec/changes/env-and-json-ui-config/tasks.md`

## File layout (after this change)

```
src/
  config.ts                                 # ENVSETTINGS_KEYS + resolveEnvSetting
  admin/router.ts                           # 3 new endpoints + generic-providers extension
  providers/genericLoader.ts                # synthesizedFromEnv flag on runtime
  providers/registry.ts                     # ProviderRuntime.synthesizedFromEnv
  db/settings.ts                            # allowlist helper, no schema change
  util/envDoc.ts                            # NEW: parses .env.example descriptions
web/
  src/pages/Settings.tsx                    # NEW: env settings page
  src/pages/providers/                      # existing; extends list + form
  src/api/client.ts                         # new typed client methods
  src/i18n/locales/en-US/settings.json      # NEW
  src/i18n/locales/zh-CN/settings.json      # NEW
  src/release-notes.tsx                     # add ReleaseHighlight on ship
test/
  resolveEnvSetting.test.ts                 # NEW
  forbiddenSettingAllowlist.test.ts         # NEW
  envSettingsEndpoints.test.ts              # NEW
  genericProvidersEnv.test.ts               # NEW
  envDoc.test.ts                            # NEW
web/src/__tests__/settingsPage.spec.ts      # NEW (Playwright)
docs/superpowers/specs/2026-06-02-env-and-json-ui-config-design.md   # this file
```

## 1. Allowlist data shape (`src/config.ts`)

```ts
export interface EnvSettingDef {
  /** Settings-DB key and UI identifier. */
  key: string;
  /** Process env var name (uppercase, original spelling). */
  envVar: string;
  /** UI grouping for the Settings page. */
  section: "server" | "auth" | "logging" | "proxy" | "advanced" | "generics";
  type: "string" | "number" | "boolean" | "enum";
  /** For type: "enum" only. */
  enumValues?: readonly string[];
  /** Compiled-in default (string form, coerced at runtime). */
  defaultValue?: string;
  /** Whether a settings-DB value takes effect without a restart. */
  restartRequired: boolean;
  /** Always read from process env / settings-DB; never shown as input. */
  secret: boolean;
  /** Description, sourced from .env.example (set by util/envDoc.ts at boot). */
  description: string;
}

export const ENVSETTINGS_KEYS: readonly EnvSettingDef[] = [
  // see table in section 3
] as const;

export const ENVSETTINGS_DENYLIST: ReadonlySet<string> = new Set([
  "mimo.apiKey",
  "deepseek.apiKey",
  "deepseek.apiKeyAlt",
  "mimo2codex.masterKey",
  "mimo2codex.providersFile",
]);
```

The `key` field uses dotted lowercase (`mimo2codex.port` for `MIMO2CODEX_PORT`,
`generic.baseUrl` for `GENERIC_BASE_URL`). The settings-DB rows created by
the new endpoints follow this exact spelling; legacy rows (`disableThinking`,
`silentRewrite`, `logBodyMode`, `logRetentionDays`, `ui.theme`, `ui.lang`) keep
their old spelling and are NOT renamed in this change.

## 2. Unified resolver (`src/config.ts`)

```ts
import { getSetting, type SettingsStore } from "./db/settings.js";
import type { ParsedArgs } from "./config.js";
import { describeEnvVar } from "./util/envDoc.js";

export interface ResolvedEnvSetting {
  def: EnvSettingDef;
  value: string | number | boolean;
  rawValue: string | null;
  source: "cli" | "env" | "settings" | "default" | "unset";
  description: string;
}

export function resolveEnvSetting(
  def: EnvSettingDef,
  parsed: ParsedArgs,
  env: NodeJS.ProcessEnv,
  settings: SettingsStore,
): ResolvedEnvSetting {
  // 1) CLI flag -- exact match on lowercased env var name
  const cliKey = kebabToCliFlag(def.envVar); // e.g. MIMO2CODEX_PORT -> port
  if (parsed[cliKey] !== undefined) {
    const { value } = coerce(def, String(parsed[cliKey]));
    return { def, value, rawValue: String(parsed[cliKey]), source: "cli", description: describeEnvVar(def.envVar) };
  }
  // 2) Process env
  const envRaw = env[def.envVar];
  if (envRaw !== undefined && envRaw !== "") {
    const { value, error } = coerce(def, envRaw);
    if (!error) return { def, value, rawValue: envRaw, source: "env", description: describeEnvVar(def.envVar) };
    // malformed env -- log + fall through; do not throw at boot
  }
  // 3) Settings DB
  const row = settings.get(def.key);
  if (row !== undefined) {
    const { value, error } = coerce(def, row);
    if (!error) return { def, value, rawValue: row, source: "settings", description: describeEnvVar(def.envVar) };
  }
  // 4) Compiled default
  if (def.defaultValue !== undefined) {
    const { value } = coerce(def, def.defaultValue);
    return { def, value, rawValue: def.defaultValue, source: "default", description: describeEnvVar(def.envVar) };
  }
  // 5) Unset
  return { def, value: emptyOf(def.type), rawValue: null, source: "unset", description: describeEnvVar(def.envVar) };
}
```

`coerce(def, raw)` lives next to `resolveEnvSetting` and returns `{ value, error? }`.
- `number`: `Number(raw)`, must be finite, must be in `[0, 65535]` for ports.
- `boolean`: non-empty string `=== "true"`, `"1"`, `"yes"`, `"on"` -> true (case-insensitive);
  any other non-empty string -> false; empty/undefined -> false. (Mirrors the
  `MIMO2CODEX_NO_REASONING=1` convention used today.)
- `enum`: must equal one of `def.enumValues`; error otherwise.
- `string`: passthrough.

The three existing per-request resolvers in `src/server.ts` (`resolveDisableThinking`,
`resolveSilentRewrite`, the `logBodyMode` / `logRetentionDays` equivalents) are
rewritten as thin wrappers over `resolveEnvSetting` with their legacy `key`
(`disableThinking`, `silentRewrite`, `logBodyMode`, `logRetentionDays`) and
`envVar` (`MIMO2CODEX_DISABLE_THINKING`, `MIMO2CODEX_SILENT_REWRITE`, etc.,
which are not actually documented today -- the legacy env-var mapping for
those is a no-op, and the resolver still returns the settings-DB value).
A regression test pins the legacy precedence.

## 3. Allowlist table (final, for the `ENVSETTINGS_KEYS` literal)

| key | env | section | type | restart | default |
|---|---|---|---|---|---|
| `mimo2codex.host` | `MIMO2CODEX_HOST` | server | string | yes | `127.0.0.1` |
| `mimo2codex.port` | `MIMO2CODEX_PORT` | server | number | yes | `8988` |
| `mimo2codex.dataDir` | `MIMO2CODEX_DATA_DIR` | server | string | yes | `~/.mimo2codex` |
| `mimo2codex.defaultProvider` | `MIMO2CODEX_DEFAULT_PROVIDER` | server | enum(mimo\|deepseek\|*generic-id*) | yes | `mimo` |
| `mimo2codex.contextOverflowMode` | `MIMO2CODEX_CONTEXT_OVERFLOW_MODE` | logging | enum(friendly\|passthrough) | no | `friendly` |
| `mimo2codex.cookieSecure` | `MIMO2CODEX_COOKIE_SECURE` | auth | boolean | yes | `false` |
| `mimo2codex.noProxyFromEnv` | `MIMO2CODEX_NO_PROXY_FROM_ENV` | proxy | boolean | yes | `false` |
| `mimo2codex.verbose` | `MIMO2CODEX_VERBOSE` | logging | boolean | no | `false` |
| `mimo2codex.noReasoning` | `MIMO2CODEX_NO_REASONING` | advanced | boolean | no | `false` |
| `proxy.https` | `HTTPS_PROXY` | proxy | string | yes | -- |
| `proxy.http` | `HTTP_PROXY` | proxy | string | yes | -- |
| `proxy.no` | `NO_PROXY` | proxy | string | yes | `localhost,127.0.0.1,::1` |
| `generic.baseUrl` | `GENERIC_BASE_URL` | generics | string | yes | -- |
| `generic.defaultModel` | `GENERIC_DEFAULT_MODEL` | generics | string | yes | -- |
| `generic.shortcut` | `GENERIC_SHORTCUT` | generics | string | yes | `generic` |
| `generic.displayName` | `GENERIC_DISPLAY_NAME` | generics | string | yes | `Generic` |
| `generic.wireApi` | `GENERIC_WIRE_API` | generics | enum(chat\|responses) | yes | `chat` |
| `generic.forceDefaultModel` | `GENERIC_FORCE_DEFAULT_MODEL` | generics | boolean | yes | `false` |

Deny-list (not in `ENVSETTINGS_KEYS`; UI never renders an input):
`MIMO_API_KEY`, `DS_API_KEY`, `DEEPSEEK_API_KEY`, `MIMO2CODEX_MASTER_KEY`,
`MIMO2CODEX_PROVIDERS_FILE`. The current `isForbiddenSettingKey()` check
in `src/admin/router.ts` is extended to a single `isEnvSettingForbidden(key)`
helper that combines `ENVSETTINGS_DENYLIST.has(key) || def.secret`.

## 4. Description source (`src/util/envDoc.ts`)

- At module load, read `<repo>/.env.example` once with `readFileSync` and
  parse with a 4-line state machine: skip blank/`#` lines, capture the most
  recent `# ...` comment, then on the next `KEY=value` line store
  `descriptions.set(KEY, lastComment)`.
- Export `describeEnvVar(envVar: string): string` returning the captured
  comment or a generic "no description" fallback.
- The `ENVSETTINGS_KEYS[*].description` is filled by a one-time
  `ENVSETTINGS_KEYS.forEach(d => d.description = describeEnvVar(d.envVar))`
  call at module load.
- In test mode (env `VITEST=1`), the parser reads `<repo>/.env.example` from
  the absolute repo root resolved via `import.meta.url`; this matches the
  existing `scripts/load-env.sh` convention.

This is a one-time read; the table is frozen for the process lifetime. No
runtime file watching.

## 5. API contracts (`src/admin/router.ts`)

### `GET /admin/api/env-settings`

```json
{
  "sections": {
    "server":     [{ "key", "envVar", "value", "rawValue", "source", "type", "restartRequired", "description" }],
    "auth":       [...],
    "logging":    [...],
    "proxy":      [...],
    "advanced":   [...],
    "generics":   [...]
  },
  "secrets": [
    { "envVar": "MIMO_API_KEY", "isSet": true|false },
    { "envVar": "DS_API_KEY", "isSet": ... },
    { "envVar": "DEEPSEEK_API_KEY", "isSet": ... },
    { "envVar": "MIMO2CODEX_MASTER_KEY", "isSet": ... }
  ],
  "restartRequired": true|false
}
```

`restartRequired` is the OR over every row whose `restartRequired === true`
AND `source === "settings"` (the row exists, will be picked up at next boot).

### `PUT /admin/api/env-settings/:key`

Request:
```json
{ "value": "9000" }
```

Behaviour:
- 404 if `:key` not in `ENVSETTINGS_KEYS` (case-sensitive).
- 400 `forbidden_setting` if `ENVSETTINGS_DENYLIST.has(:key)` or `def.secret`.
- 400 `invalid_value` if `coerce(def, body.value).error` is set; payload
  includes `{ allowed: def.enumValues }` for enums or `{ min, max }` for numbers.
- Otherwise: `setSetting(def.key, body.value)`; respond `200 { key, value, source: "settings" }`.
- Restart-required flag computed by re-running `resolveEnvSetting` for every
  def after the write, and the SPA's banner refreshes on the next `GET`.

### `DELETE /admin/api/env-settings/:key`

- 404 if not in `ENVSETTINGS_KEYS`.
- `deleteSetting(def.key)`; respond `200 { deleted: true, key }`.

### `GET /admin/api/generic-providers` (extended)

Existing payload shape preserved. Each entry gains:
```ts
{
  ...existingFields,
  synthesizedFromEnv: boolean,
  warning?: string,  // only when env + json both declare id "generic"
}
```

The env-synthesized entry is placed **first** in the array, before any
json-declared entries with the same id.

### `DELETE /admin/api/generic-providers/env` (new)

- 404 if no env-synthesized `generic` provider is in the runtime registry.
- Removes every `generic.*` row from the settings table.
- `log.warn("generic provider removed; restart to apply")`.
- 200 `{ deleted: true, envCleared: true, restartRequired: true }`.

## 6. `synthesizedFromEnv` on the runtime

`src/providers/registry.ts`:
```ts
export interface ProviderRuntime {
  // ... existing fields
  synthesizedFromEnv: boolean;
}
```

`src/providers/genericLoader.ts`:
- The env-synthesized `GenericProviderSpec` (built in
  `loadGenericProviders()` around line 260-280 today) gets
  `synthesizedFromEnv: true` before being passed to `createGenericProvider`.
- The existing `loadFromFile()` path sets `synthesizedFromEnv: false` for every
  spec read from disk.
- If both env and json declare id `generic`, the env entry wins at boot
  (existing behavior); the loader logs a warning that the UI will surface as
  the `warning` field on the entry.

## 7. Settings page UI (`web/src/pages/Settings.tsx`)

Component tree:

```
<SettingsPage>
  <Section title="Server"   keys={serverKeys} />
  <Section title="Auth"     keys={authKeys}   />
  <Section title="Logging"  keys={loggingKeys}/>
  <Section title="Proxy"    keys={proxyKeys}  />
  <Section title="Advanced" keys={advancedKeys}/>
  <Section title="Generics" keys={genericsKeys}/>
  <SecretsSection secrets={secrets} />
  <RestartRequiredBanner source="env-settings" />
</SettingsPage>
```

Per-row layout (antd `Form.Item`):

```
[Label "Listening port"]    [Input: 8988]   [env]  [restart required]  [Reset]
```

- Source badge: antd `<Tag>` colored `blue` (env), `purple` (cli),
  `green` (settings), `default` (default), `default` (unset). Empty
  badge hidden.
- "restart required" pill: red `<Tag>` only when `def.restartRequired`
  AND the running value differs from the form value.
- "Reset" button: only shown when `source === "settings"`; calls
  `DELETE /admin/api/env-settings/:key`.
- Save flow: every row is uncontrolled; on blur, debounced 300 ms
  `PUT /admin/api/env-settings/:key`. The page does not have a global
  "Save" button -- the per-row save is the contract.

`SecretsSection`:

- Rendered as a non-editable list of four env vars, each with a
  `<InfoCircleOutlined />` tooltip:
  > "Set this in the .env file on the machine running mimo2codex,
  > then restart. The UI never accepts secret values to avoid
  > leaking them through admin logs and history."
- `isSet` badge: green "set" / grey "unset", read from
  `process.env[envVar] !== undefined` on the server side at GET time.

## 8. Providers page extension (`web/src/pages/providers/`)

- The providers list filters `synthesizedFromEnv === true` rows to the top
  and shows `<Tag color="blue">from env</Tag>` next to the id.
- If `warning` is set on the row, an `<ExclamationCircleOutlined />` icon
  with a `<Tooltip>` follows the badge.
- The existing `ProviderFormModal` is reused for editing. On submit,
  the form fields are mapped to `generic.*` settings-DB writes via
  `PUT /admin/api/env-settings/:key` (one call per changed field).
- A new `<Button danger>Delete</Button>` on the env-synthesized row
  opens a `Modal.confirm` that says:
  > "Remove the env-synthesized generic provider? The .env file is
  > NOT modified. The settings-DB rows for `generic.*` are cleared.
  > If providers.json has its own `generic` entry, it takes over
  > after the next restart. Continue?"
- Confirm -> `DELETE /admin/api/generic-providers/env` -> `message.success`
  with a follow-up "Restart mimo2codex for changes to take effect."

## 9. Restart banner integration

- The existing `RestartRequiredBanner` component gains a new variant:
  `source: "env-settings" | "codex-apply"`.
- The banner is fed by `AppConfigContext`, which now polls
  `GET /admin/api/env-settings` every 30 s and on focus. When
  `response.restartRequired === true`, the banner shows:
  - 1 item: "N env-backed values pending restart"
  - Click "Show details" -> navigates to `/admin/settings`.
- Per-row "restart required" pills in the Settings page (section 7) are
  computed client-side from the same payload: a row needs restart if
  `def.restartRequired` AND the running value (last GET response) differs
  from the form value. A row that was just saved is always flagged.

## 10. i18n keys (`web/src/i18n/locales/{en-US,zh-CN}/settings.json`)

Namespace: `settings`. New file. Keys follow the convention
`settings.<key>.label`, `settings.<key>.help`, `settings.<key>.placeholder`,
`settings.section.<section>`, `settings.secret.<envVar>`, `settings.actions.reset`,
`settings.actions.edit`, `settings.banner.restart`, `settings.banner.details`.

The English and Chinese files are kept in lockstep (same key set, same
ordering). The PR template's translation checklist requires both before merge.

## 11. Test strategy

### Backend (`test/`)

- `resolveEnvSetting.test.ts` -- table-driven; for each of the 4 types, for
  each of 5 source combinations (cli / env / settings / default / unset),
  assert `source` and the coerced `value`. ~20 cases.
- `forbiddenSettingAllowlist.test.ts` -- call the new endpoints with every
  deny-list key; expect 400 `forbidden_setting`. Repeat with the key
  *also* in `ENVSETTINGS_KEYS` (simulated via test-only allowlist override)
  to confirm the deny-list always wins.
- `envSettingsEndpoints.test.ts` -- happy-path GET/PUT/DELETE for `mimo2codex.port`
  and `generic.baseUrl`; type-coercion failures (string for `number`,
  invalid enum value); out-of-allowlist keys.
- `genericProvidersEnv.test.ts` -- boot two fixtures:
  (a) env shortcut only,
  (b) env shortcut + json entry with id `generic`,
  (c) json entry only.
  Assert `synthesizedFromEnv` and `warning` field per case.
- `envDoc.test.ts` -- verify the parser captures comments above `KEY=value`
  pairs, ignores trailing comments, and handles CRLF / LF / BOM. Snapshot
  the captured map against the real `.env.example` to catch silent drift.

### Frontend (`web/src/__tests__/`)

- `settingsPage.spec.ts` (Playwright):
  - Load `/admin/settings`, assert every section renders.
  - Edit `mimo2codex.port` to 9000, save; assert the
    "restart required" banner shows and the row has the red pill.
  - Reset the row; assert the banner clears.
  - Open `/admin/providers` with `GENERIC_*` env vars set in the test
    fixture; assert the "from env" badge.
  - Open `/admin/settings`, assert the four secrets are present in the
    Secrets section as read-only rows.

## 12. File-by-file change list

| File | Change |
|---|---|
| `src/config.ts` | Add `EnvSettingDef`, `ENVSETTINGS_KEYS` literal, `ENVSETTINGS_DENYLIST`, `resolveEnvSetting`, `coerce`, `isEnvSettingForbidden`. Extend `Config` with the resolved fields from the new defs (`cookieSecure`, `noProxyFromEnv`, `contextOverflowMode`, `verbose`, `noReasoning`, `defaultProvider`, `dataDir`, `host`, `port`). |
| `src/admin/router.ts` | Add 3 endpoints + extend `GET /admin/api/generic-providers`. Wire `loadConfig` so `cookieSecure` etc. come from `resolveEnvSetting`. |
| `src/providers/registry.ts` | Add `synthesizedFromEnv: boolean` to `ProviderRuntime`. |
| `src/providers/genericLoader.ts` | Set `synthesizedFromEnv: true` on the env-synthesized spec; set `false` on the json-loaded specs. Log a warning when both env and json declare id `generic`. |
| `src/db/settings.ts` | Add `isEnvSettingForbidden(key: string): boolean` helper combining the deny-list and `def.secret`. No schema change. |
| `src/util/envDoc.ts` | New: parse `.env.example` descriptions. |
| `src/server.ts` | Rewrite the three existing per-request resolvers as thin wrappers over `resolveEnvSetting`. Add `resolve*` for the new defs that need per-request re-read (`contextOverflowMode`, `verbose`, `noReasoning`). |
| `web/src/pages/Settings.tsx` | New. |
| `web/src/pages/providers/index.tsx` | Render env-synthesized row with badge + warning. Add the "Delete" button. |
| `web/src/pages/providers/ProviderFormModal.tsx` | Save handler splits form values into `generic.*` settings writes. |
| `web/src/api/client.ts` | Add `getEnvSettings`, `putEnvSetting`, `deleteEnvSetting`, `deleteGenericEnvProvider` typed methods. |
| `web/src/i18n/locales/en-US/settings.json` | New. |
| `web/src/i18n/locales/zh-CN/settings.json` | New. |
| `web/src/release-notes.tsx` | New `ReleaseHighlight` entries on ship. |
| `web/src/components/AppHeader.tsx` | Link to `/admin/settings`; show restart-required count. |
| `web/src/App.tsx` (or router) | New `/admin/settings` route. |
| `test/resolveEnvSetting.test.ts` | New. |
| `test/forbiddenSettingAllowlist.test.ts` | New. |
| `test/envSettingsEndpoints.test.ts` | New. |
| `test/genericProvidersEnv.test.ts` | New. |
| `test/envDoc.test.ts` | New. |
| `web/src/__tests__/settingsPage.spec.ts` | New. |
| `README.md` / `README.zh.md` | One-paragraph pointer to the new page in the "Configuration" section. |
| `doc/tag-log.md` + `doc/tag-log.zh.md` | `[new]` entry on ship. |
| `openspec/changes/env-and-json-ui-config/` | No further edits in this phase. |

## 13. Open questions deferred

These were surfaced in the OpenSpec `design.md` and confirmed during
brainstorming; they are explicitly NOT addressed in this change and will
land in a follow-up if the operator asks:

- "Copy as .env" button on the Settings page. (Hand-off to ops.)
- Linking the Settings page to `/admin/api/setup-snippets` for export to
  Codex `auth.json` / `config.toml`.
- Whether to make `cookieSecure` hot-reloadable via a session-middleware
  re-read. (Currently classified `restartRequired: true`.)
