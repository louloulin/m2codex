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
- **AND** the banner hides itself when no items remain
