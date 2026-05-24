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

### 问题现象

Hermes 会话里 `/reasoning` 或 `agent.reasoning_effort` 已经能动态设置：

```text
none / minimal / low / medium / high / xhigh
```

但在当前 custom provider + Sub2API + `gpt-5.5` 路径下，主对话最终发出的 OpenAI-compatible `chat/completions` 请求体里**没有任何 reasoning 参数**。结果是：

- Hermes UI / 配置层看起来已经切到了 `xhigh`。
- 但 Sub2API 后台实际看到的强度并没有跟随变化。
- 手动用真实 HTTP 请求加顶层字段：

```json
"reasoning_effort": "xhigh"
```

Sub2API 后台可以正确显示为 `xhigh`，说明网关和模型本身支持这个字段，问题在 Hermes 请求构造层。

### 根因

Hermes 的 reasoning 参数在不同 provider 路径下有不同映射方式：

- OpenRouter / Nous 等路径主要走 `extra_body.reasoning`。
- Kimi、TokenHub、LM Studio 等路径有各自的顶层 `reasoning_effort` 或 thinking 字段处理。
- custom provider 是 OpenAI-compatible 的通用兜底路径，默认不会假设某个中转支持哪种 reasoning 字段。

因此在 `api_mode == "chat_completions"` 且 `provider == "custom"` 时，`_build_api_kwargs()` 虽然拿到了：

```python
self.reasoning_config = {"enabled": True, "effort": "xhigh"}
```

但最终 `ChatCompletionsTransport.build_kwargs()` 没有把它映射成 Sub2API 需要的顶层字段：

```json
"reasoning_effort": "xhigh"
```

也就是说，问题不是配置没生效，而是**配置到最终 HTTP JSON 的 provider-specific 映射缺失**。

### 修复策略

本修复采用“窄门控、可回滚”的方式，只针对已验证支持该字段的路径生效：

必须同时满足：

- 当前 API 模式是 OpenAI-compatible `chat_completions`
- 当前 provider 是 `custom`
- `base_url` 的 host 精确匹配：`sub2api.codeva.top`
- 当前模型属于 GPT-5.x 系列，例如：
  - `gpt-5.5`
  - `gpt-5`
  - `openai/gpt-5.5` 这类带 provider 前缀的模型名也会先取最后一段再判断

满足条件后，把 Hermes 当前动态 reasoning 配置：

```python
reasoning_config["effort"]
```

映射为最终请求体的顶层字段：

```json
"reasoning_effort": "<当前 effort>"
```

例如：

```json
"reasoning_effort": "low"
```

或：

```json
"reasoning_effort": "xhigh"
```

这里没有硬编码 `xhigh`。它会随当前 `/reasoning` 或 `agent.reasoning_effort` 动态变化。

### 修改位置

核心代码：

- `agent/transports/chat_completions.py`

新增 helper：

- `_base_url_host_matches(...)`
- `_is_sub2api_gpt5_reasoning_route(...)`
- `_resolve_sub2api_reasoning_effort(...)`

在 `ChatCompletionsTransport.build_kwargs()` 中增加：

```python
if _is_sub2api_gpt5_reasoning_route(base_url, model, is_custom_provider):
    _sub2api_effort = _resolve_sub2api_reasoning_effort(reasoning_config)
    if _sub2api_effort is not None:
        api_kwargs["reasoning_effort"] = _sub2api_effort
```

回归测试：

- `tests/run_agent/test_provider_parity.py`

新增测试覆盖：

- `low` / `medium` / `xhigh` 会进入顶层 `reasoning_effort`
- disabled 时不发送 `reasoning_effort`
- Sub2API 上的非 GPT-5.x 模型不发送
- 其他 custom endpoint 的 GPT-5.x 模型不发送

### disabled / none 策略

这里区分两个语义：

1. 明确关闭 reasoning：

```python
{"enabled": False}
```

这种情况省略 `reasoning_effort`。理由是 Hermes 语义里这是“不要请求 reasoning”，省略字段是对 OpenAI-compatible 中转最保守的关闭方式。

2. 用户明确选择枚举值 `none`：

```python
{"enabled": True, "effort": "none"}
```

这种情况按接口支持的枚举原样传：

```json
"reasoning_effort": "none"
```

### 验证记录

`_build_api_kwargs()` 验证结果：

```json
{"effort":"low","reasoning_effort":"low","top_level_keys":["messages","model","reasoning_effort","timeout"],"extra_body":{}}
{"effort":"medium","reasoning_effort":"medium","top_level_keys":["messages","model","reasoning_effort","timeout"],"extra_body":{}}
{"effort":"xhigh","reasoning_effort":"xhigh","top_level_keys":["messages","model","reasoning_effort","timeout"],"extra_body":{}}
```

实际通过修复后的 `_build_api_kwargs()` + OpenAI client 向 Sub2API 发送过两条请求：

```json
{
  "effort": "low",
  "built_top_level_reasoning_effort": "low",
  "status": "ok",
  "id": "resp_0805ce4331c1d1b9016a135bb8847881918e532bde5dbf139c",
  "content": "low",
  "usage": {"completion_tokens": 17, "prompt_tokens": 31, "total_tokens": 48}
}
```

```json
{
  "effort": "xhigh",
  "built_top_level_reasoning_effort": "xhigh",
  "status": "ok",
  "id": "resp_010ad7d8bfe4e508016a135bbd17b48191b3cea623cfa5c873",
  "content": "xhigh",
  "usage": {"completion_tokens": 50, "prompt_tokens": 32, "total_tokens": 82}
}
```

定向回归测试通过：

```text
6 passed
```

### 影响范围

这个修复只影响：

```text
custom provider + sub2api.codeva.top + chat_completions + gpt-5.x
```

不会改变这些路径：

- Ollama
- OpenRouter
- Nous
- Kimi / Moonshot
- LM Studio
- Gemini
- GitHub Models / Copilot
- 其他 custom endpoint
- Sub2API 上非 GPT-5.x 模型

### 回滚方式

如果之后 Sub2API 改了字段协议，或者 Hermes 上游实现了更通用的 custom reasoning 映射，可以直接回滚相关提交，或移除 `agent/transports/chat_completions.py` 里的 Sub2API helper 与注入块。

## 更新前

- 重新套补丁前先做 `git apply --check`。
- 更新后确认 custom `codex_responses` 的 `User-Agent` 修复还在。
- `memory_policy.yaml` 在本机 Hermes home 里。
