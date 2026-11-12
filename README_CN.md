# CC-Switch × Command Code 配置指南

> 在 [CC-Switch](https://github.com/farion1231/cc-switch) 里接上 [Command Code](https://commandcode.ai)，支持 **Claude Code** 和 **Claude Desktop**，并显示额度 —— 不用再撞 `messages.N.role` 那堵墙。

[English](README.md) · 实测环境：CC-Switch **4.0.29** / Windows 11 · Command Code API（2026-11）

---

## 30 秒版

| 配置项 | 值 |
| --- | --- |
| Base URL | `https://api.commandcode.ai/provider` |
| **接口格式** | **OpenAI Chat** ← 关键就这一条 |
| 模型 | `deepseek/deepseek-v4.1-flash`（或任意 OSS / OpenAI 模型） |
| API Key | https://commandcode.ai/settings/keys |
| 额度查询 | 内置 **Token Plan → Command Code** |

只记一句话：**Command Code 千万别用 Anthropic 格式。**

---

## 为什么必须用 OpenAI Chat？

Command Code 有两个数据面接口：

| 接口 | 协议 | 接受什么 |
| --- | --- | --- |
| `/provider/v1/messages` | Anthropic Messages | **只收 Claude 模型**，且 `role` 只能是 `user` / `assistant` |
| `/provider/v1/chat/completions` | OpenAI Chat Completions | OSS / OpenAI 模型（`deepseek/...`、`gpt-...` 等） |

Claude Code 和 Claude Desktop 会往 `messages[]` 里塞 `role: "system"` 的消息（比如 `<total_tokens>…</total_tokens>` 这种提示）。Anthropic 那个接口用判别联合校验，直接拒掉：

```json
{
  "type": "error",
  "error": {
    "type": "invalid_request_error",
    "message": "Invalid discriminator value. Expected 'user' | 'assistant' at messages.1.role"
  }
}
```

OpenAI 那个接口允许 `system` 出现在数组任意位置，而且只有它能跑 OSS 模型。所以能用的组合是：

> **OpenAI Chat 格式 + OSS 模型**，由 CC-Switch 的本地代理自动完成 Anthropic → OpenAI 的转换。

转换不用你写代码，只要在供应商表单里选对格式。

---

## Claude Code 配置步骤

1. 打开 CC-Switch → **Claude Code** 标签页 → 新增供应商
2. 选内置的 **Command Code** 预设（或手动填）
3. 把**接口格式**改成 **OpenAI Chat**
4. 模型填 `deepseek/deepseek-v4.1-flash`
   - 后缀 `[1M]`（`deepseek/deepseek-v4.1-flash[1M]`）是 CC-Switch 标记 1M 上下文的写法，代理转发前会自动剥掉，带不带都行
5. 粘贴 API Key
6. 保存后**切换**到该供应商
7. CC-Switch 会提示该供应商**需要本地路由**（因为要做格式转换），确认即可
8. 重启 Claude Code

CC-Switch 会把 `~/.claude/settings.json` 里的 `ANTHROPIC_BASE_URL` 指向本地代理（`127.0.0.1:15721`），再由代理转发到 Command Code。

---

## Claude Desktop 配置步骤

1. 打开 CC-Switch → **Claude Desktop** 标签页 → 新增供应商
2. Base URL 填 `https://api.commandcode.ai/provider`，接口格式选 **OpenAI Chat**
3. 把 Claude 的模型档位映射到 Command Code 模型，例如四档全填 `deepseek/deepseek-v4.1-flash`
4. 粘贴 API Key
5. 切换到该供应商，确认进入**路由模式**
6. **重启 Claude Desktop**，让它重新读取被改写的网关配置

---

## 模型与套餐限制

Command Code 按套餐限制 Claude 模型。低档套餐会返回：

```json
{"type":"error","error":{"type":"permission_error",
 "message":"MODEL_NOT_IN_PLAN: Claude Sonnet 5 available in Pro and above plans or extra on demand usage"}}
```

实用结论：

- **OSS / OpenAI 模型**（`deepseek/deepseek-v4.1-flash`、`gpt-6-luna`、`google/gemini-4-argon` 等）→ 走 **OpenAI Chat** 格式，大多数套餐都包含
- **Claude 模型**（`claude-haiku-5-5`、`claude-sonnet-5` 等）→ 只能走 `/provider/v1/messages`，也就是 **Anthropic** 格式；但那样又会撞上 `role: "system"` 的问题，所以 Claude Code / Claude Desktop 这条路径实际不可用
- `claude-haiku-5-5` 是入门套餐里通常能用的那个 Claude 模型，但同样只能走 Anthropic 接口

查看你的 Key 能看到哪些模型：

```bash
curl https://api.commandcode.ai/provider/v1/models \
  -H "Authorization: Bearer $COMMAND_CODE_API_KEY"
```

---

## 额度显示

CC-Switch 4.0.29 **内置**了 Command Code 的额度查询，不需要自己写脚本。

在供应商卡片上：**⋯ → 配置用量查询 → 启用用量查询 → Token Plan → Command Code**。

它会用你的 Key 调四个接口，显示 5 小时窗口、周窗口和剩余额度：

| 接口 | 返回 |
| --- | --- |
| `/alpha/whoami` | 账号 / 组织 |
| `/alpha/billing/credits` | 5h + 周窗口、剩余月度额度 |
| `/alpha/billing/subscriptions` | 套餐与当前周期 |
| `/alpha/usage/summary` | 当前周期消费 |

注意：只有供应商处于**使用中**状态，后台才会自动刷新。

---

## 常见报错对照

| 报错 | 原因 | 解决 |
| --- | --- | --- |
| `400 Invalid discriminator value. Expected 'user' \| 'assistant' at messages.N.role` | 用了 Anthropic 格式，客户端发了 `role: "system"` 消息 | 把供应商改成 **OpenAI Chat** |
| `400 Model "deepseek/..." is not supported on this endpoint. Use /provider/v1/chat/completions` | OSS 模型打到了 Anthropic 接口 | 改成 **OpenAI Chat** |
| `400 Model "claude-..." must be called via /provider/v1/messages` | Claude 模型打到了 OpenAI 接口 | 换 OSS 模型，或改回 Anthropic 格式 |
| `403 MODEL_NOT_IN_PLAN` | 套餐不含该 Claude 模型 | 换 OSS 模型 |
| `401 UNAUTHORIZED` | Key 错误或过期 | 去 https://commandcode.ai/settings/keys 重新复制 |
| 额度一直不显示 | 用量查询没开，或供应商不在使用中 | 在卡片上开启并切换过去 |
| 改完 Claude Desktop 还是报错 | 它缓存了旧的网关配置 | 重启 Claude Desktop |

---

## 一键导入（可选）

CC-Switch 支持 `ccswitch://` 链接。把 `YOUR_API_KEY` 换成你的 Key 后打开：

```
ccswitch://v1/import?resource=provider&app=claude&name=Command%20Code&endpoint=https%3A%2F%2Fapi.commandcode.ai%2Fprovider&apiKey=YOUR_API_KEY&apiFormat=openai_chat&model=deepseek%2Fdeepseek-v4.1-flash&usageEnabled=true
```

Claude Desktop 把 `app=claude` 换成 `app=claude-desktop`。CC-Switch 会先弹确认框；确认接口格式是 **OpenAI Chat**，如果额度查询没自动打开，再手动开一下。

---

## 备选：自定义额度脚本

如果你更想手写查询脚本（比如想自定义展示排版），可以看 [cc-switch-commandcode-usage](https://github.com/Spirit597/cc-switch-commandcode-usage)。它不是必需的 —— 内置模板覆盖的数据更全。

---

## 参考

- [CC-Switch](https://github.com/farion1231/cc-switch) —— Claude Code / Codex / Gemini 等客户端的供应商管理器
- [Command Code](https://commandcode.ai) —— 编码助手
- [CC-Switch 用量查询文档](https://github.com/farion1231/cc-switch/blob/main/docs/user-manual/en/2-providers/2.5-usage-query.md)

## License

MIT
