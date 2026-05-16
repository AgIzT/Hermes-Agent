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

## 更新前

- 这个个人仓库刻意移除了 `.github` 自动化，不跑 GitHub Actions / Dependabot。
- 重新套补丁前先做 `git apply --check`。
- 更新后确认 custom `codex_responses` 的 `User-Agent` 修复还在。
- `memory_policy.yaml` 在本机 Hermes home 里。
