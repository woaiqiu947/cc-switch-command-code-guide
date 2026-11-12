# CC-Switch × Command Code

> Get [Command Code](https://commandcode.ai) working in [CC-Switch](https://github.com/farion1231/cc-switch) — for **Claude Code** and **Claude Desktop** — including the quota display, without hitting the `messages.N.role` wall.

[中文说明](README_CN.md) · Tested with CC-Switch **4.0.29** on Windows 11 · Command Code API as of 2026-11

---

## The 30-second version

| Setting | Value |
| --- | --- |
| Base URL | `https://api.commandcode.ai/provider` |
| **API format** | **OpenAI Chat** ← the one that matters |
| Model | `deepseek/deepseek-v4.1-flash` (or any OSS / OpenAI model) |
| API key | https://commandcode.ai/settings/keys |
| Usage query | Built-in **Token Plan → Command Code** |

If you only remember one thing: **do not use the Anthropic format with Command Code.**

---

## Why "OpenAI Chat" and not "Anthropic"?

Command Code exposes two data-plane endpoints:

| Endpoint | Shape | Accepts |
| --- | --- | --- |
| `/provider/v1/messages` | Anthropic Messages | **Claude models only**, and `role` must be `user` / `assistant` |
| `/provider/v1/chat/completions` | OpenAI Chat Completions | OSS / OpenAI models (`deepseek/...`, `gpt-...`, ...) |

Claude Code and Claude Desktop inject `role: "system"` entries into `messages[]` — for example the `<total_tokens>…</total_tokens>` notice. The Anthropic endpoint validates the array with a discriminated union and rejects them outright:

```json
{
  "type": "error",
  "error": {
    "type": "invalid_request_error",
    "message": "Invalid discriminator value. Expected 'user' | 'assistant' at messages.1.role"
  }
}
```

The OpenAI endpoint accepts `system` messages anywhere in the array, and it is also the only endpoint that serves OSS models. So the working combination is:

> **OpenAI Chat format + an OSS model**, with CC-Switch's local proxy doing the Anthropic → OpenAI conversion.

You do not write any conversion code — you just have to pick the right format in the provider form.

---

## Claude Code setup

1. Open CC-Switch → **Claude Code** tab → **Add provider**
2. Pick the built-in **Command Code** preset (or fill the fields manually)
3. Set **API format** to **OpenAI Chat**
4. Set the model to `deepseek/deepseek-v4.1-flash`
   - The `[1M]` suffix (`deepseek/deepseek-v4.1-flash[1M]`) is a CC-Switch marker for a 1M context window. The proxy strips it before forwarding, so either form works.
5. Paste your API key
6. Save, then **switch** to the provider
7. CC-Switch will tell you the provider **needs local routing** (because of the format conversion) — accept it
8. Restart Claude Code

CC-Switch writes `~/.claude/settings.json` with `ANTHROPIC_BASE_URL` pointing at its local proxy (`127.0.0.1:15721`), and the proxy forwards to Command Code.

---

## Claude Desktop setup

1. Open CC-Switch → **Claude Desktop** tab → add a provider
2. Base URL `https://api.commandcode.ai/provider`, API format **OpenAI Chat**
3. Map the Claude model slots to a Command Code model, e.g. all four slots → `deepseek/deepseek-v4.1-flash`
4. Paste your API key
5. Switch to the provider and accept **routing mode**
6. **Restart Claude Desktop** so it re-reads the rewritten gateway config

---

## Models and plan limits

Command Code gates Claude models by plan. On a lower tier you will see:

```json
{"type":"error","error":{"type":"permission_error",
 "message":"MODEL_NOT_IN_PLAN: Claude Sonnet 5 available in Pro and above plans or extra on demand usage"}}
```

Practical rules:

- **OSS / OpenAI models** (`deepseek/deepseek-v4.1-flash`, `gpt-6-luna`, `google/gemini-4-argon`, …) → use the **OpenAI Chat** format. These are what most plans include.
- **Claude models** (`claude-haiku-5-5`, `claude-sonnet-5`, …) → only on `/provider/v1/messages`, i.e. the **Anthropic** format. But then the `role: "system"` problem comes back, so this path is not usable from Claude Code / Claude Desktop.
- `claude-haiku-5-5` is the one Claude model that tends to work on entry plans — still only via the Anthropic endpoint.

List the models your key can see:

```bash
curl https://api.commandcode.ai/provider/v1/models \
  -H "Authorization: Bearer $COMMAND_CODE_API_KEY"
```

---

## Quota display

CC-Switch 4.0.29 has a **built-in** Command Code quota query — you do not need a custom script.

On the provider card: **⋯ → Configure usage query → Enable usage query → Token Plan → Command Code**.

It calls four endpoints with your key and shows the 5-hour window, the weekly window and the remaining credits:

| Endpoint | What it returns |
| --- | --- |
| `/alpha/whoami` | account / org |
| `/alpha/billing/credits` | 5h + weekly windows, remaining monthly credits |
| `/alpha/billing/subscriptions` | plan and current period |
| `/alpha/usage/summary` | spend for the current period |

The provider must be **in use** for the background refresh to run.

---

## Troubleshooting

| Error | Cause | Fix |
| --- | --- | --- |
| `400 Invalid discriminator value. Expected 'user' \| 'assistant' at messages.N.role` | Anthropic format; the client sent a `role: "system"` message | Switch the provider to **OpenAI Chat** |
| `400 Model "deepseek/..." is not supported on this endpoint. Use /provider/v1/chat/completions` | OSS model sent to the Anthropic endpoint | Switch to **OpenAI Chat** |
| `400 Model "claude-..." must be called via /provider/v1/messages` | Claude model sent to the OpenAI endpoint | Use an OSS model, or switch to Anthropic format |
| `403 MODEL_NOT_IN_PLAN` | Your plan does not include that Claude model | Use an OSS model |
| `401 UNAUTHORIZED` | Wrong / expired key | Re-copy from https://commandcode.ai/settings/keys |
| Quota never appears | Usage query off, or provider not in use | Enable it on the card and switch to the provider |
| Claude Desktop still fails after a fix | It cached the old gateway config | Restart Claude Desktop |

---

## One-click import (optional)

CC-Switch understands `ccswitch://` links. Replace `YOUR_API_KEY` and open it:

```
ccswitch://v1/import?resource=provider&app=claude&name=Command%20Code&endpoint=https%3A%2F%2Fapi.commandcode.ai%2Fprovider&apiKey=YOUR_API_KEY&apiFormat=openai_chat&model=deepseek%2Fdeepseek-v4.1-flash&usageEnabled=true
```

Use `app=claude-desktop` for Claude Desktop. CC-Switch shows a confirmation dialog before importing; check that the format is **OpenAI Chat** and enable the usage query afterwards if it is not already on.

---

## Alternative: custom usage script

If you prefer a hand-written query script (for example to show a different layout), see [cc-switch-commandcode-usage](https://github.com/Spirit597/cc-switch-commandcode-usage). It is not required — the built-in template covers the same data and more.

---

## References

- [CC-Switch](https://github.com/farion1231/cc-switch) — provider manager for Claude Code, Codex, Gemini and more
- [Command Code](https://commandcode.ai) — the coding agent
- [CC-Switch usage query docs](https://github.com/farion1231/cc-switch/blob/main/docs/user-manual/en/2-providers/2.5-usage-query.md)

## License

MIT
