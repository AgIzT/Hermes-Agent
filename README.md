# AgIzT Hermes 自用记录

个人自用优化版。主要记录当前代码基线、合并过的社区修复，以及我自己这边的网关兼容改动，方便以后更新时别忘。

## 当前基线

- 上游项目：`NousResearch/hermes-agent`
- 基线提交：`9fb40e6a3d63`
- 基线版本：`Hermes Agent v0.13.0 (2026.5.7)`
- 当前个人修改提交：`300755b7e951 Apply community patches on 9fb40e6a`
- 当前分支：`hermes-9fb40e6a-patches`

## 合并的社区修复

社区补丁来源：

- https://github.com/Cyrene963/hermes-patches

已应用：

- `combined-final.patch`
- `agent/disclosure_router.py`
- `agent/memory_policy.default.yaml`
- `agent/memory_metacognition.py`
- `agent/skill_db.py`
- `agent/hybrid_skill_selector.py`
- `plugins/skill-enforcer/`

包括：

- 安全边界增强：secret redaction、SSRF、防敏感文件读写、防路径穿越等。
- memory/session 相关修复：多用户隔离、CJK/session search 等。
- skill 相关优化：SkillDB、skill pre-selection、skill-enforcer。
- memory metacognition / preflight policy。
- 一些 custom provider、credential pool、gateway 兼容修复。

## 网关配置

实际运行配置在本机 Hermes home，不提交到这个 repo：

```text
\hermes\config.yaml
```

## /v1/responses 兼容记录

之前 `/v1/responses` 报错主要有两类原因：

- 请求体要按 Responses API 的形状来，不能直接拿普通 chat/completions 的参数硬套。
- 这个网关会拦 OpenAI Python SDK 默认的 `User-Agent`，表现为 `Your request was blocked.`

所以这里保留了一个小改动：

- `agent/auxiliary_client.py`
- `run_agent.py`

重新部署或升级后，如果 custom `codex_responses` 又挂了，优先检查这两处：当 `provider == "custom"` 且 `api_mode == "codex_responses"` 时，要给 OpenAI client 显式传一个 Hermes 自己的 `User-Agent`。

## 本机额外文件

这些文件在本机 Hermes home 里，不属于源码 repo：

```text
\hermes\LOCAL_CHANGES.md
\hermes\memory_policy.yaml
\hermes\config.yaml
\hermes\auth.json
```

其中：

- `LOCAL_CHANGES.md` 是更详细的本机维护笔记。
- `memory_policy.yaml` 是从社区补丁复制出来的运行配置。

## 端口记录

- `9177`：memory metacognition / hindsight API，配置在 `memory_policy.yaml`
- `9119`：Hermes web dashboard 默认端口
- `8642`：Hermes OpenAI-compatible API server 默认端口
- `8644`：Webhook 平台默认端口
- `8645`：BlueBubbles webhook 默认端口

当前 `config.yaml` 没有显式写 API server/webhook 端口，上面这些主要是 Hermes 默认值和社区补丁配置值。


## Sub2API GPT-5.5 reasoning_effort 修复

本次新增一个窄门控兼容修复：当 Hermes 走 OpenAI-compatible `chat_completions`，且满足以下条件时：

- `provider == "custom"`
- `base_url` 的 host 精确为 `sub2api.codeva.top`
- 当前模型为 `gpt-5.x` 系列，例如 `gpt-5.5`

会把 Hermes 当前动态 reasoning 配置映射到最终 HTTP JSON 的顶层字段：

```json
"reasoning_effort": "<当前 /reasoning 或 agent.reasoning_effort>"
```

也就是说，`low` / `medium` / `high` / `xhigh` 等值不会硬编码，会随当前会话的 `/reasoning` 设置变化。

修改位置：

- `agent/transports/chat_completions.py`
- `tests/run_agent/test_provider_parity.py`

验证记录：

- `_build_api_kwargs()` 已分别验证 `low`、`medium`、`xhigh`，最终 kwargs 均包含顶层 `reasoning_effort`，且不放入 `extra_body.reasoning`。
- 已实际向 `https://sub2api.codeva.top/v1/chat/completions` 发送 `low` 与 `xhigh` 两条请求，均返回 200，Sub2API 后台可按请求记录确认 reasoning 强度不同。

禁用策略：

- `reasoning_config.enabled is False` 时省略 `reasoning_effort`，表示明确关闭 reasoning。
- `effort == "none"` 但未显式 disabled 时按接口枚举原样传 `"none"`。

这个修复只作用于 Sub2API 的 custom GPT-5.x 路径，不影响 Ollama、OpenRouter、Nous、Kimi、LM Studio 等既有逻辑。

## 更新前

- 重新套补丁前先做 `git apply --check`。
- 更新后确认 custom `codex_responses` 的 `User-Agent` 修复还在。
- `memory_policy.yaml` 在本机 Hermes home 里。
