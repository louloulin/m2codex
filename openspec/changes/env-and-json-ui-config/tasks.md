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
