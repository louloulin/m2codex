# Codex Enable — one-click model switching in the webui (cc-switch replacement)

> Added **2026-05-14**, available since **v0.2.6**.
> WebUI entry: `http://127.0.0.1:8988/admin/` → sidebar **"Codex 启用"**.

## What it does

Collapses the "switch which provider/model Codex actually talks to" workflow from "install cc-switch / hand-edit `~/.codex/*` / scroll command-line snippets" into a single **Enable** button in the webui. Two mechanisms ship in parallel — pick by use case:

| Mechanism | Changes | Requires Codex restart | Use case |
|---|---|---|---|
| **A. Write files & enable** | `~/.codex/auth.json` + `~/.codex/config.toml` | ✅ yes | First-time setup, swapping Codex's default `model`, full cc-switch replacement |
| **B. Runtime override only** | mimo2codex's internal `settings` DB | ❌ no | Trying a different upstream model without restarting Codex, A/B comparisons |

They compose. Semantically: A changes "what Codex sends"; B changes "what mimo2codex forwards upstream". Pass 0 of `selectProvider` has the highest priority — once B is set, every incoming request routes to your chosen pair regardless of what Codex sent.

## What it replaces from cc-switch

[cc-switch](https://github.com/farion1231/cc-switch) is a standalone desktop app whose key value is swapping `~/.codex/auth.json` + `config.toml` with one click. For mimo2codex users, the **Codex Enable** page does exactly that, removing one external dependency:

| Capability | cc-switch | Codex Enable |
|---|---|---|
| One-click write of `auth.json` + `config.toml` | ✅ | ✅ |
| Model catalog (mimo / deepseek / generic providers) | manual entry per provider | auto-populated from `PROVIDER_LIST` + custom models |
| Auto-backup the previous files | ✅ | ✅ (paired ts suffix) |
| **Preserve the user's original Codex config from auto-pruning** | ❌ (rolling N) | ✅ (`.preserve` tag, never pruned) |
| Display each backup's captured `provider/model` | ❌ | ✅ (regex-sniffs the toml) |
| Symmetric restore for half-pairs | n/a | ✅ (deletes the file that didn't exist pre-apply) |
| Runtime override (no Codex restart) | ❌ | ✅ |
| Cross-provider routing history | ❌ | ✅ (built into the admin logs page) |

> **What it does not replace**: cc-switch can also manage Claude Code / OpenCode provider profiles — outside mimo2codex's scope. The two can coexist; both touch `~/.codex/`, but mimo2codex is aware of which backup is the user's original config and pins it.

## Backups & restore

### Automatic backup

Every "Write files & enable" backs up both files. Naming:

```
auth.json.bak.<ms-timestamp>.<pid>[.preserve]
config.toml.bak.<ms-timestamp>.<pid>[.preserve]
```

Both halves of the same apply share the **same ts** — that's the "paired backup" guarantee. Restore handles them together so you never end up with `requires_openai_auth = true` over a foreign `OPENAI_API_KEY`.

### `.preserve` permanent retention ⭐

When the `auth.json` being overwritten was **not** written by us (real OpenAI login, leftover from cc-switch, hand-crafted, …), the pair is tagged `.preserve`. `pruneBackups` permanently skips `.preserve` files — **switch models 100 times and your earliest "real Codex config" still survives**.

Detection rule based on `auth.json`'s `OPENAI_API_KEY` value:
- `== "mimo2codex-local"` → our write → **not** preserved (intermediate snapshot)
- anything else / unparseable JSON / file missing → treated as external → **preserved**

The UI labels these as 🔒 "原始 Codex 配置" and requires a stronger confirm + backend `?force=1` to delete.

### Regular snapshot rotation

Non-preserved backups roll with a "keep newest 10" policy. Older ones are pruned at apply time.

### Restore

Click "恢复" in the backups table. Backend semantics:

- **Complete pair** (both halves backed up) → atomic-write both files back
- **Half-pair** (only one half was backed up) → write the existing half back, **delete the current copy of the other half**

The second case is what returns the disk to its real pre-apply state. Typical scenario: user has real OpenAI auth.json but no customized config.toml. Our first apply only backs up auth.json. Restore-only-the-auth-half would leave our config.toml dangling — exactly the mixed state the paired-backup design tried to prevent.

### Manual delete

Each row has a delete button:
- Regular snapshot: confirm → `DELETE /admin/api/codex-backups/:ts`
- 🔒 preserved: extra warning + backend `?force=1`; message: "deleting will permanently lose the one-click path back to your original real Codex config"

## Runtime override (Mechanism B) details

Clicking "仅运行时覆盖" on a row calls `PUT /admin/api/active-override`, storing `{providerId, modelId}` in the `settings` table. `selectProvider` checks this first:

```
Pass 0: runtime override (if provider exists AND has a runtime / api key) → hit
Pass 1: user-declared generic providers (non-empty models[] in providers.json + has key)
Pass 2: built-in providers (mimo / deepseek + has key)
Fallback: defaultProviderId's defaultModel
```

Edge behavior:

- **Override points at an unknown providerId** → silent fall-through to Pass 1 (don't throw — bad config shouldn't 500 every request)
- **Override points at a provider with no api key** → also falls through. The PUT endpoint refuses up-front with `400 provider_has_no_key` so you can't set an invalid override.
- **Override's modelId not in the provider's catalog** → still routes to that provider, attaches a `rewriteNotice`. Admin logs flag this as `client_model_rewritten`.
- **`--no-admin` mode** → no DB, `getActiveOverride` silently returns null → behaves as "no override".

Clear it: "清除覆盖" button or `DELETE /admin/api/active-override`.

## REST API cheat sheet

All under `/admin/api/`:

| Method | Path | What |
|---|---|---|
| GET | `/codex-state` | Current `~/.codex/` snapshot — owner, file existence, backup list, runtime override |
| GET | `/codex-targets` | Flat list of pickable (provider × model) + `hasKey` / `isCurrentOverride` |
| POST | `/codex-apply` | body `{providerId, modelId}` → write files + auto-backup |
| POST | `/codex-restore` | body `{ts}` → symmetric restore (handles half-pairs) |
| DELETE | `/codex-backups/:ts[?force=1]` | Delete the paired backup at this ts (preserved requires force) |
| GET | `/active-override` | Current runtime override |
| PUT | `/active-override` | body `{providerId, modelId}` → set override (requires the provider has a key) |
| DELETE | `/active-override` | Clear override |

## Caveats / known limitations

- **`config.toml` full overwrite**: like cc-switch, apply replaces the file wholesale. Other `[model_providers.X]` / `[notice.xxx]` / `[mcp_servers.X]` sections you had get clobbered — **but they're in the backup**, one-click restore brings them back. May switch to a key-replacement merge in the future; matched to cc-switch's behavior for now.
- **Symlinks**: if `~/.codex/config.toml` is a symlink, `renameSync` breaks the link (target replaced by a real file). Matches cc-switch behavior.
- **Concurrency**: backup names embed `<ts>.<pid>`. Parallel clicks at different ts can't collide; same-ms same-pid collisions are vanishingly rare and `copyFileSync` would just overwrite (no harm).
- **Windows**: `os.homedir()` resolves to `%USERPROFILE%\.codex\` automatically.
- **`--no-admin`**: this feature needs the admin DB. The routes still exist with `--no-admin`, but settings ops fail.

## Troubleshooting

- **"provider has no api key configured"** → no env var matched for that provider. `export ...` and restart mimo2codex.
- **"incomplete backup pair"** (only seen on pre-v0.2.6) → upgrade, or manually remove the orphan `.bak.*` file.
- **Codex still uses the old model after enable** → Codex reads config.toml at startup. **Fully quit** (tray / menu bar Quit) and relaunch.
- **Runtime override set but not in effect** → `GET /admin/api/active-override` to verify. If null, the DB isn't open (`--no-admin` or unwritable dataDir).

## Related

- [Generic Providers](./generic-providers.md) — providers declared in `providers.json` (non-built-in)
- [README — Admin console](../README.md#admin-console)
