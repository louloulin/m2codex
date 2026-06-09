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

`generic.baseUrl`, `generic.defaultModel`, `generic.shortcut`, `generic.displayName`, `generic.wireApi`, and `generic.forceDefaultModel` SHALL be in the `ENVSETTINGS_KEYS` allowlist. `generic.apiKey` SHALL NOT be in the allowlist; it is a secret and SHALL remain `forbidden_setting`.

#### Scenario: Operator edits generic.baseUrl from the Settings page
- **WHEN** the operator edits the `generic.baseUrl` row in the Settings page
- **THEN** the endpoint writes the row and responds with `200`
- **AND** the Providers page reflects the new value (after restart) on the env-synthesized `generic` row

#### Scenario: Operator attempts to edit generic.apiKey from the Settings page
- **WHEN** the operator types into the `generic.apiKey` row
- **THEN** the row is rendered as read-only with a tooltip explaining that the key is set via the env file
