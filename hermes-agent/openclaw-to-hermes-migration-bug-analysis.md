# Hermes Agent 从 OpenClaw 迁移配置启动：踩坑与问题定位完整记录

## 概述

本文记录将 OpenClaw 的百炼（bailian）provider 配置迁移到 Hermes Agent 并成功启动的完整过程，涉及配置路径混淆、custom_providers 注册机制、api_mode 注入链路等多个底层问题的定位与修复。

---

## 1. 时间线总览

| 阶段 | 问题 | 根因 | 修复 |
|------|------|------|------|
| 1 | Hermes 用 alibaba/glm-5 发送请求，API key 无效 | `HERMES_HOME` 在 Windows 上是 `%LOCALAPPDATA%\hermes`，而非 `~/.hermes`，之前的配置写入了错误路径 | 将配置写入 `%LOCALAPPDATA%\hermes\config.yaml` |
| 2 | 模型切换到 glm-5，URL 拼接为 `/chat/completions` | config.yaml 写入后 YAML 文件编码损坏（0xa8 字节） | 用 `encoding='utf-8'` 读写 YAML |
| 3 | `model.provider: custom` 时仍走 OpenRouter | `requested_provider` 为 `"custom"` 而非 `"bailian-coding-plan"`，无法匹配 custom_providers 条目 | 设置 `model.provider: bailian-coding-plan` |
| 4 | `bailian-coding-plan` 显示 0 models | custom_providers 缺少 `models` 字段 | 添加完整的 model 列表和 context_length |
| 5 | URL 为 `/v1/chat/completions` 而非 `/v1/messages`（404） | 虽然 `api_mode` 正确解析为 `anthropic_messages`，但 base_url 已有 `/v1` 后缀，且 runtime 中被切换 | 去掉 base_url 中的 `/v1` 后缀，添加 `key_env` 防止 credential pool 冲突 |

---

## 2. 根本原因：Windows 下 Hermes 的 HERMES_HOME 路径与直觉不同

### 现象

编辑了 `C:\Users\Admin\.hermes\config.yaml`，但 Hermes 完全没有读取这些配置。

### 根因

`hermes_constants.get_hermes_home()` 在 Windows 上的默认返回值是 `C:\Users\Admin\AppData\Local\hermes`，而非 `C:\Users\Admin\.hermes`。

```python
# 验证
from hermes_constants import get_hermes_home
print(get_hermes_home())  # C:\Users\Admin\AppData\Local\hermes
```

这意味着：
- 配置路径：`%LOCALAPPDATA%\hermes\config.yaml`
- 环境变量：`%LOCALAPPDATA%\hermes\.env`
- 会话/日志/缓存均在 `%LOCALAPPDATA%\hermes\` 下

**教训**：在 Windows 上操作 Hermes 配置，始终通过 `get_hermes_home()` 获取路径，不要假设是 `~/.hermes`。

---

## 3. YAML 编码损坏问题

### 现象

写完 `config.yaml` 后，Hermes 启动时报错：

```
Failed to parse config.yaml: 'utf-8' codec can't decode byte 0xa8 in position 488
```

### 根因

原始 `config.yaml` 中包含非 UTF-8 字符（如 cursor 配置中的 `\xa8\x81` 字节）。使用 `yaml.dump()` 写入时没有显式指定 `encoding='utf-8'`，导致文件编码不一致。

### 修复

```python
# 错误写法
with open(cf, 'w') as f:
    yaml.dump(cfg, f)

# 正确写法
output = yaml.dump(cfg, default_flow_style=False, allow_unicode=True)
with open(cf, 'w', encoding='utf-8') as f:
    f.write(output)
```

同时从备份文件 `config.yaml.corrupt.*.bak` 恢复原始内容再重新写入。

---

## 4. custom_providers 的 provider 名称匹配机制

### 现象

设置 `model.provider: custom` 和 `custom_providers: [{name: bailian-coding-plan}]` 后，Hermes 仍然走 OpenRouter 默认端点。

### 根因

`_get_named_custom_provider()` (runtime_provider.py:492) 按以下优先级匹配：

1. 先检查是不是合法的内置 provider（通过 `auth_mod.resolve_provider`）
2. 再在 `providers` dict 中匹配 key
3. 最后在 `custom_providers` list 中匹配 `name` 字段

当 `requested_provider = "custom"` 时，它不匹配 `bailian-coding-plan` 这个 name，所以 `_get_named_custom_provider("custom")` 返回 `None`。

只有当 `requested_provider = "bailian-coding-plan"` 时才能正确匹配：

```python
runtime = resolve_runtime_provider(requested='bailian-coding-plan')
# → provider: custom, base_url: .../anthropic, api_mode: anthropic_messages
```

### 修复

将 `model.provider` 设置为完整的 provider 名称：

```yaml
model:
  default: bailian-coding-plan/qwen3.6-plus
  provider: bailian-coding-plan  # 不能是 "custom"，必须是 custom_providers 中声明的名称
```

---

## 5. credential pool 与 provider 自动切换

### 现象

即使 `model.provider: bailian-coding-plan` 设置正确，实际请求仍显示：

```
Provider: alibaba-coding-plan  Model: glm-5
Endpoint: https://coding-intl.dashscope.aliyuncs.com/v1
```

### 根因

1. Hermes 启动时会扫描所有注册的 provider，自动将可用凭证写入 `auth.json` 的 credential_pool
2. 内置 `alibaba-coding-plan` provider 的 env_vars 包含 `DASHSCOPE_API_KEY`
3. 如果 sys env 中恰好存在 `DASHSCOPE_API_KEY`（或之前遗留），alibaba-coding-plan 就会被自动识别为"有凭证可用"
4. 某些代码路径中，Hermes 会从 model 名称推断 provider，如果模型名匹配了内置 catalog，就会切换
5. `key_env` 字段缺失时，custom provider 的凭证优先级低于内置 provider

### 修复

在 `custom_providers` 条目中显式声明 `key_env`：

```yaml
custom_providers:
  - name: bailian-coding-plan
    base_url: https://coding.dashscope.aliyuncs.com/apps/anthropic
    api_key: API-KEY
    key_env: BAILIAN_CODING_PLAN_API_KEY    # 关键字段
    api_mode: anthropic_messages
    models:
      qwen3.6-plus:
        context_length: 1000000
      # ... 更多模型
```

---

## 6. base_url 的 `/v1` 后缀与 Anthropic adapter 路径拼接

### 现象

设置 `base_url: https://coding.dashscope.aliyuncs.com/apps/anthropic/v1` 时，实际请求 URL 为：

```
https://coding.dashscope.aliyuncs.com/apps/anthropic/v1/chat/completions
```

返回 404。

### 根因

1. Hermes 的 Anthropic adapter (`agent/anthropic_adapter.py:697`) 调用 `_normalize_base_url_text(base_url)` 保留完整路径
2. Anthropic SDK 会在 base_url 后追加 `/v1/messages` 作为完整请求路径
3. 如果 base_url 以 `/v1` 结尾，最终路径变成 `/v1/v1/messages`（但实际 404 说明走的不是 Anthropic 路径，而是 OpenAI chat completions 路径 —— 见下一节）
4. 实际上 404 的真正原因是 `api_mode` 在运行时被覆写为 `chat_completions`，导致走 OpenAI 的 `/v1/chat/completions` 路径而非 Anthropic 的 `/v1/messages`

### 修复

去掉 base_url 中的 `/v1`，让 SDK 自己拼接：

```yaml
base_url: https://coding.dashscope.aliyuncs.com/apps/anthropic
# NOT: https://coding.dashscope.aliyuncs.com/apps/anthropic/v1
```

---

## 7. api_mode 在运行时被覆写的机制

### 现象

Runtime 解析中 `api_mode` 正确为 `anthropic_messages`，但实际请求的 URL 是 `/v1/chat/completions`（OpenAI 格式）。

### 根因

`agent/agent_init.py:311-342` 的 api_mode 决策链：

```python
if api_mode in {"chat_completions", "codex_responses", "anthropic_messages", ...}:
    agent.api_mode = api_mode                       # ← 311 行：如果传入合法 api_mode 就直接用
elif agent._base_url_lower.rstrip("/").endswith("/anthropic"):  # ← 329 行
    agent.api_mode = "anthropic_messages"            # URL 包含 /anthropic 时自动检测
else:
    agent.api_mode = "chat_completions"             # ← 342 行：fallback
```

当传入的 api_mode 参数本身就是 `anthropic_messages` 时，311 行会正确处理。但如果调用方没传或传了空值，且 URL 不以 `/anthropic` 结尾（例如包含 `/v1` 后缀破坏了匹配），就会走到 else 分支被设为 `chat_completions`。

同时 `model` 字段也可能在运行时被 `normalize_model_for_provider` 改写。

### 修复

综合前几节的修复（正确的 `api_mode`、`base_url`、`key_env`）确保 `agent_init` 中收到正确的参数。

---

## 8. 最终可用配置

```yaml
model:
  default: bailian-coding-plan/qwen3.6-plus
  provider: bailian-coding-plan

custom_providers:
  - name: bailian-coding-plan
    base_url: https://coding.dashscope.aliyuncs.com/apps/anthropic
    api_key: API-KEY
    key_env: BAILIAN_CODING_PLAN_API_KEY
    api_mode: anthropic_messages
    default_model: qwen3.6-plus
    models:
      qwen3.7-plus:
        context_length: 1000000
      qwen3.6-plus:
        context_length: 1000000
      qwen3.5-plus:
        context_length: 1000000
      qwen3-max-2026-01-23:
        context_length: 262144
      qwen3-coder-next:
        context_length: 262144
      qwen3-coder-plus:
        context_length: 1000000
      glm-5:
        context_length: 202752
      glm-4.7:
        context_length: 202752
      MiniMax-M2.5:
        context_length: 204800
      kimi-k2.5:
        context_length: 262144
```

对应的 `.env`：

```env
BAILIAN_CODING_PLAN_API_KEY=API-KEY
```

---

## 9. 关键文件索引

| 文件 | 作用 |
|------|------|
| `hermes_constants.py` | `get_hermes_home()` — Windows 路径决策 |
| `hermes_cli/config.py:5152` | `_load_config_impl()` — 配置加载 + deep merge |
| `hermes_cli/config.py:3862` | `get_compatible_custom_providers()` — custom_providers 标准化 |
| `hermes_cli/runtime_provider.py:426` | `resolve_requested_provider()` — 从 config 解析 provider 名称 |
| `hermes_cli/runtime_provider.py:492` | `_get_named_custom_provider()` — 按名称匹配 custom provider |
| `hermes_cli/runtime_provider.py:1228` | `resolve_runtime_provider()` — 完整解析链 |
| `agent/agent_init.py:311` | api_mode 决策链 |
| `agent/anthropic_adapter.py:644` | `build_anthropic_client()` — Anthropic 客户端构建 |
| `agent/anthropic_adapter.py:369` | `_is_third_party_anthropic_endpoint()` — 第三方端点检测 |
| `cli.py:3231` | `requested_provider` 初始化 |

---

## 10. 关键教训

1. **Windows 路径陷阱**：Hermes 的 `HERMES_HOME` 在 Windows 上默认是 `%LOCALAPPDATA%\hermes`，不是 `~/.hermes`
2. **YAML 编码**：始终用 `encoding='utf-8'` 读写 config.yaml，原始文件可能含非 UTF-8 字节
3. **provider 名称**：`model.provider` 必须与 `custom_providers[].name` 完全一致，不能设为 `"custom"`
4. **key_env 字段**：自定义 provider 必须声明 `key_env`，否则 credential pool 优先级低于内置 provider
5. **base_url 后缀**：不要手动加 `/v1`，SDK 会自动拼接
6. **api_mode 链路**：即使 runtime 解析正确，`agent_init.py` 中的决策链可能因 URL 特征不匹配而覆写
7. **`hermes claw migrate` 的路径限制**：Windows 上因长路径可能失败，必要时手动写配置
