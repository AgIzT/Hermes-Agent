# AgIzT Hermes 自用记录

这份仓库是个人自用优化版，主要目标是：保留一个能直接跑的 Hermes 基线，加上社区修复，以及我自己这台机器上验证过的 custom Responses 网关兼容改动。

## 当前基线

- 上游项目：`NousResearch/hermes-agent`
- 当前基线提交：`9fb40e6a3d63`
- 基线版本：`Hermes Agent v0.13.0 (2026.5.7)`
- 本仓库本地补丁提交：`300755b7e951 Apply community patches on 9fb40e6a`
- 当前开发分支：`hermes-9fb40e6a-patches`

为什么选这个基线：

- 旧的 `v0.12.0` 上直接套社区补丁会冲突。
- `9fb40e6a3d63` 这个点已经验证可以干净应用社区补丁。

## 合并的社区修复

社区补丁来源：

- https://github.com/Cyrene963/hermes-patches

已应用内容：

- `combined-final.patch`
- `agent/disclosure_router.py`
- `agent/memory_policy.default.yaml`
- `agent/memory_metacognition.py`
- `agent/skill_db.py`
- `agent/hybrid_skill_selector.py`
- `plugins/skill-enforcer/`
- 一批安全、memory、session、skill、gateway、custom provider 相关修复

简单理解：

- 修了一堆安全边界，比如 secret redaction、SSRF、敏感文件读写、防路径穿越。
- 补了 memory/session 多用户隔离、CJK/session search 相关内容。
- 加了 SkillDB / skill pre-selection / skill-enforcer 之类的 token 和技能选择优化。
- 加了 memory metacognition / preflight policy 这套更主动的记忆检查框架。
- 修了部分 custom provider / credential pool / max_tokens / gateway 兼容问题。

## 我自己的网关配置

实际机器上的运行配置在 Hermes home，不直接提交到这个 repo：

```text
D:\program\hermes\config.yaml
```

当前关键配置是：

```yaml
model:
  default: gpt-5.5
  provider: custom
  base_url: https://sub2api.codeva.top/v1
  api_key: <本机已配置，仓库里不要写明文>
  api_mode: codex_responses

agent:
  max_turns: 60
  verbose: false
  reasoning_effort: xhigh
```

注意：

- `base_url` 写到 `/v1` 就够了，不要写成 `/v1/responses`。
- `/responses` 是 SDK/Hermes 自己拼出来的。
- `api_mode` 必须是 `codex_responses`。
- 当前模型是 `gpt-5.5`。
- 思考深度改成了 `xhigh`，Hermes 默认一般是 `medium`。

## /v1/responses 踩坑记录

这个 custom 网关一开始不能直接跑，主要不是模型问题，而是请求格式和请求头问题。

网关要求：

- `instructions` 必须存在。
- `input` 必须是 list，不能是普通字符串。
- `max_output_tokens` 会被这个网关拒绝。
- OpenAI Python SDK 默认的 `User-Agent: OpenAI/Python ...` 会被网关拦截，报：

```text
Your request was blocked.
```

所以我额外保留了一个很小的兼容修复：

- `agent/auxiliary_client.py`
- `run_agent.py`

核心逻辑是：当走 `provider == "custom"` 且 `api_mode == "codex_responses"` 时，显式加 Hermes 自己的 UA：

```python
default_headers = {"User-Agent": "HermesAgent/<version>"}
```

这个改动很重要。以后升级、rebase、重新套社区补丁后，第一件事就是检查它有没有丢。

## 本机额外文件

这些文件在本机 Hermes home 里，不属于源码 repo：

```text
D:\program\hermes\LOCAL_CHANGES.md
D:\program\hermes\memory_policy.yaml
D:\program\hermes\config.yaml
D:\program\hermes\auth.json
```

其中：

- `LOCAL_CHANGES.md` 是更详细的本机维护笔记。
- `memory_policy.yaml` 是从社区补丁复制出来的运行配置。
- `config.yaml` 里有真实 API key，不能提交。
- `auth.json` 里有认证信息，不能提交。

## 端口记录

当前需要记住的端口：

- `9177`：memory metacognition / hindsight API，配置在 `memory_policy.yaml`
- `9119`：Hermes web dashboard 默认端口
- `8642`：Hermes OpenAI-compatible API server 默认端口
- `8644`：Webhook 平台默认端口
- `8645`：BlueBubbles webhook 默认端口

当前 `config.yaml` 没有显式写 API server/webhook 端口，上面这些主要是 Hermes 默认值和社区补丁配置值。

## Windows gateway 重启方式

本机是 Windows，gateway 一般这样起：

```powershell
$env:HERMES_HOME = 'D:\program\hermes'
Start-Process `
  -FilePath 'D:\program\hermes\hermes-agent\venv\Scripts\hermes.exe' `
  -ArgumentList @('gateway', 'run') `
  -WorkingDirectory 'D:\program\hermes' `
  -WindowStyle Hidden `
  -RedirectStandardOutput 'D:\program\hermes\logs\gateway-restart-YYYYMMDD-HHMMSS.out.log' `
  -RedirectStandardError  'D:\program\hermes\logs\gateway-restart-YYYYMMDD-HHMMSS.err.log'
```

PID 文件：

```text
D:\program\hermes\gateway.pid
```

日志：

```text
D:\program\hermes\logs\gateway-restart-*.out.log
D:\program\hermes\logs\gateway-restart-*.err.log
```

## 更新前检查

以后每次 `hermes update`、切上游分支、重新套社区补丁之前，先看这些：

1. 不要把 `config.yaml`、`auth.json`、`.env`、真实 key 之类东西提交上来。
2. 确认当前基线是不是还想停在 `9fb40e6a3d63`，还是要换新基线。
3. 社区补丁最好先 `git apply --check`，别直接跑一键脚本。
4. 更新后确认 `custom + codex_responses` 的 User-Agent 修复还在。
5. 跑一次最小 Responses 测试，能返回 `OK` 再重启 gateway。

最小测试大概长这样：

```powershell
$env:HERMES_HOME = 'D:\program\hermes'
$env:PYTHONPATH = 'D:\program\hermes\hermes-agent'
@'
from hermes_cli.runtime_provider import resolve_runtime_provider
from agent.auxiliary_client import resolve_provider_client

rt = resolve_runtime_provider(requested="custom")
client, model = resolve_provider_client(
    provider="custom",
    model=rt.get("model") or "gpt-5.5",
    api_mode=rt.get("api_mode"),
    explicit_base_url=rt.get("base_url"),
    explicit_api_key=rt.get("api_key"),
)
resp = client.chat.completions.create(
    model=model,
    messages=[{"role": "user", "content": "Reply OK"}],
    max_tokens=8,
)
print(resp.choices[0].message.content)
'@ | D:\program\hermes\hermes-agent\venv\Scripts\python.exe -
```

预期：

```text
OK
```

