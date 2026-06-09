# Codex 启用 — webui 一键切模型（替代 cc-switch）

> 新增于 **2026-05-14**，起始支持版本 **v0.2.6**。
> webui 入口：`http://127.0.0.1:8988/admin/` → 侧栏「**Codex 启用**」。

## 它能干嘛

把切换 Codex 真正使用的 provider/model 这件事，从「装个 cc-switch / 手动改 `~/.codex/*` / 翻命令行 snippet」收敛成 webui 里点一下「启用」。同时提供两套机制，按你的场景选：

| 机制 | 改的是 | 需重启 Codex | 用场景 |
|---|---|---|---|
| **A. 写入文件并启用** | `~/.codex/auth.json` + `~/.codex/config.toml` | ✅ 需要 | 首次接入、给 Codex 看的"默认 model"换掉、彻底替代 cc-switch |
| **B. 仅运行时覆盖** | mimo2codex 内部 `settings` DB | ❌ 不需要 | 已接入后想临时换上游 model 做实验、A/B 对比 |

两种可以并存。语义上：A 改"Codex 发什么"，B 改"mimo2codex 上游转什么"。Pass 0 路由优先级最高——只要设了 B，Codex 发什么都被代理替换成你指定的组合。

## 替代 cc-switch 的方面

cc-switch 是一个独立桌面应用，最大价值是按按钮就把 `~/.codex/auth.json` + `config.toml` 换掉。对 mimo2codex 用户来说，「Codex 启用」页直接做这事，少一个外部依赖：

| 能力 | cc-switch | Codex 启用 |
|---|---|---|
| 一键写 `auth.json` + `config.toml` | ✅ | ✅ |
| 模型列表（含 mimo / deepseek / 通用 provider） | 需手动添加 | 自动来自 PROVIDER_LIST + custom models |
| 自动备份原文件 | ✅ | ✅（成对 ts 后缀） |
| **保护原始 Codex 配置不被自动清理** | ❌（只滚动保留若干份） | ✅（`.preserve` 标记，永远不 prune） |
| 显示每份备份当时的 `provider/model` | ❌ | ✅（regex 嗅探 toml） |
| 半残 pair 的对称恢复 | N/A | ✅（删除当时不存在的那个文件） |
| 运行时覆盖（不重启 Codex） | ❌ | ✅ |
| 跨 provider/model 路由历史可视化 | ❌ | ✅（admin 日志页配套） |

> **不替代的部分**：cc-switch 还能管 Claude Code / OpenCode 等其他 CLI 工具的 provider profile，那不是 mimo2codex 的范畴。这两个工具完全可以共存——「Codex 启用」操作 `~/.codex/`，cc-switch 操作的也是同一目录，关键差异是 mimo2codex 知道哪份备份是你的真原始配置并永久保留它。

## 备份与恢复

### 自动备份

每次「写入文件并启用」**都**会备份 `auth.json` + `config.toml`，文件名格式：

```
auth.json.bak.<毫秒时间戳>.<pid>[.preserve]
config.toml.bak.<毫秒时间戳>.<pid>[.preserve]
```

同一次 apply 产生的两个文件**共享同一个 ts**——这是「成对备份」，恢复时一起回退，避免出现 `requires_openai_auth = true` + 老 OpenAI key 的混合脏状态。

### `.preserve` 永久保留 ⭐

当被覆盖的 `auth.json` **不是 mimo2codex 写的**（你的真 OpenAI 登录、cc-switch 留下的、手写的等等）时，那一对备份会自动加 `.preserve` 后缀。pruneBackups 永远跳过 `.preserve` 文件——**即使你切换 100 次模型，最早那份"真 Codex 配置"也不会丢**。

判定依据是 `auth.json` 里 `OPENAI_API_KEY` 的值：
- `== "mimo2codex-local"` → 是我们写的 → **不**保留（属于中间快照）
- 其他任何值 / JSON 解析失败 / 文件不存在 → 视为外部 → **保留**

UI 会把这些备份标成 🔒「原始 Codex 配置」，删除时弹强警告，后端必须 `?force=1` 才放行。

### 普通快照轮转

非 `.preserve` 的备份按"keep 最近 10 份"轮转，最老的被自动清理。这样切换历史不会无限膨胀。

### 恢复

webui 备份表里点「恢复」即可。后端语义：

- **完整 pair**（auth + toml 都有备份）→ 原子写回两个文件
- **半残 pair**（只备份了其中一个）→ 把有备份的那半写回，**把另一半当前文件删掉**

第二种是为了让磁盘真正回到 apply 前的状态。典型场景：用户原本只有 OpenAI 登录（`auth.json`），从不自定义 `config.toml`。我们启用 mimo2codex 时只能备份 `auth.json`（toml 不存在）。恢复时如果只写回 `auth.json` 留着我们的 `config.toml`，就是脏状态。

### 手动删除

每行有「删除」按钮：
- 普通快照：二次确认后 `DELETE /admin/api/codex-backups/:ts`
- 🔒 保留备份：二次确认 + 后端 `?force=1`，提示"删除后将无法再一键恢复回你最初的真 Codex 配置"

## 运行时覆盖（机制 B）的细节

当你点某行「仅运行时覆盖」，会调 `PUT /admin/api/active-override`，把 `{providerId, modelId}` 存到 `settings` 表。`selectProvider` 在路由前先看这个覆盖：

```
Pass 0：运行时 override（若 provider 存在且有 runtime / api key）→ 命中
Pass 1：用户声明的通用 provider（providers.json 里 models[] 非空 + 有 key）
Pass 2：内置 provider（mimo / deepseek + 有 key）
Fallback：默认 provider 的 defaultModel
```

边界行为：

- **Override 指向不存在的 providerId** → 静默 fall through 到 Pass 1（不抛错，避免坏配置导致全部请求 500）
- **Override 指向没有 api key 的 provider** → 也 fall through。UI 在 PUT 时就会用 400 + `provider_has_no_key` 拒收，不让你设置一个无效的覆盖
- **Override 的 modelId 不在 provider catalog 里** → 仍然路由到该 provider，但携带 `rewriteNotice`，admin 日志页会标 `client_model_rewritten`
- **`--no-admin` 模式** → 没有 DB，`getActiveOverride` 静默返回 null，等效"未设置"

清除覆盖：UI 上「清除覆盖」按钮，或 `DELETE /admin/api/active-override`。

## REST API 速查

全部挂在 `/admin/api/` 下：

| Method | Path | 作用 |
|---|---|---|
| GET | `/codex-state` | 当前 `~/.codex/` 状态（owner、文件存在、备份列表、运行时 override） |
| GET | `/codex-targets` | 可启用的 (provider × model) 平铺列表 + `hasKey` / `isCurrentOverride` |
| POST | `/codex-apply` | body `{providerId, modelId}`，写入文件 + 自动备份 |
| POST | `/codex-restore` | body `{ts}`，对称恢复（含半残 pair） |
| DELETE | `/codex-backups/:ts[?force=1]` | 删除某 ts 的成对备份（保留型需 force） |
| GET | `/active-override` | 当前覆盖 |
| PUT | `/active-override` | body `{providerId, modelId}`，设置覆盖（要求该 provider 有 key） |
| DELETE | `/active-override` | 清除覆盖 |

## 注意事项 / 已知限制

- **`config.toml` 全量覆盖**：当前实现和 cc-switch 一样，写入时整体替换文件，你原来在 toml 里的其他 `[model_providers.X]` / `[notice.xxx]` / `[mcp_servers.X]` 段会被覆盖。**但都在备份里**，可一键还原。后续可能改成只替换相关键的合并写入，目前为了与 cc-switch 行为对齐先全量覆盖。
- **Symlink**：如果 `~/.codex/config.toml` 是 symlink，`renameSync` 会让 symlink 断开（指向被换到真文件）。和 cc-switch 行为一致。
- **并发**：备份后缀 `<ts>.<pid>`，并发点击不同 ts 产生的备份不会撞名；同一 pid 同一毫秒撞名概率极低，撞了 `copyFileSync` 会覆盖（无害）。
- **Windows**：路径用 `os.homedir()`，自动解析到 `%USERPROFILE%\.codex\`。
- **`--no-admin`**：本页所有功能依赖 admin DB；启动时加 `--no-admin` 则页签不可用（routes 仍存在但 settings 操作会失败）。

## 故障排查

- **「provider 没有 api key 配置」** → 该 provider 没有匹配的 env key 被设置。先 `export ...` 再重启 mimo2codex。
- **「成对备份不完整」**（旧版才会出现，v0.2.6 后改为对称恢复，不再报这个） → 升级版本或直接手动删除遗留的半残备份文件。
- **启用后 Codex 还是用旧 model** → Codex 启动时读 config.toml，**必须完全退出**（系统托盘 / 菜单栏 Quit）后再启动。
- **运行时覆盖设了但没生效** → 检查 `GET /admin/api/active-override` 是否返回你设的；如果返回 null，可能 db 没 open（`--no-admin` 或 dataDir 没权限）。

## 相关文档

- [通用 Provider](./generic-providers.zh.md) — 在 `providers.json` 里声明的非内置 provider
- [README — admin 控制台](../README.zh.md#admin-控制台) — Admin UI 总览
