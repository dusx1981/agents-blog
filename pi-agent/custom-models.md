# Custom Models（自定义模型）

**版本：最新**  
[查看源码] [在 GitHub 上编辑]

通过 `~/.pi/agent/models.json` 文件，你可以添加自定义的提供商和模型（例如 Ollama、vLLM、LM Studio 以及各种代理）。

## 目录
- 最小示例
- 完整示例
- Google AI Studio 示例
- 支持的 API
- 提供商配置
- 模型配置
- 覆盖内置提供商
- 逐模型覆盖
- Anthropic Messages 兼容性
- OpenAI 兼容性

## 最小示例

对于本地模型（Ollama、LM Studio、vLLM 等），每个模型只需要提供 `id` 字段：

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "models": [
        { "id": "llama3.1:8b" },
        { "id": "qwen2.5-coder:7b" }
      ]
    }
  }
}
```

`apiKey` 是必填的，但 Ollama 会忽略它，所以可以填任意值。

一些兼容 OpenAI 的服务器不理解用于具备推理能力的模型的 `developer` 角色。对于此类提供商，可将 `compat.supportsDeveloperRole` 设为 `false`，这样 Pi 会改用 `system` 消息来发送系统提示。如果服务器也不支持 `reasoning_effort`，请同时将 `compat.supportsReasoningEffort` 设为 `false`。

`compat` 可以设置在提供商级别（对所有模型生效），也可以设置在模型级别（覆盖特定模型）。这通常适用于 Ollama、vLLM、SGLang 及类似的兼容 OpenAI 的服务器。

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "compat": {
        "supportsDeveloperRole": false,
        "supportsReasoningEffort": false
      },
      "models": [
        {
          "id": "gpt-oss:20b",
          "reasoning": true
        }
      ]
    }
  }
}
```

## 完整示例

当你需要更具体的配置值时，可以覆盖默认设置：

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "models": [
        {
          "id": "llama3.1:8b",
          "name": "Llama 3.1 8B (Local)",
          "reasoning": false,
          "input": ["text"],
          "contextWindow": 128000,
          "maxTokens": 32000,
          "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 }
        }
      ]
    }
  }
}
```

该文件会在每次打开 `/model` 时重新加载。你可以在会话过程中编辑它，无需重启。

## Google AI Studio 示例

使用 `google-generative-ai` 搭配 `baseUrl`，可以添加来自 Google AI Studio 的模型，包括自定义的 Gemma 4 条目：

```json
{
  "providers": {
    "my-google": {
      "baseUrl": "https://generativelanguage.googleapis.com/v1beta",
      "api": "google-generative-ai",
      "apiKey": "$GEMINI_API_KEY",
      "models": [
        {
          "id": "gemma-4-31b-it",
          "name": "Gemma 4 31B",
          "input": ["text", "image"],
          "contextWindow": 262144,
          "reasoning": true
        }
      ]
    }
  }
}
```

对于 `google-generative-ai` API 类型，添加自定义模型时必须提供 `baseUrl`。

## 支持的 API

| API | 说明 |
| :--- | :--- |
| `openai-completions` | OpenAI Chat Completions（兼容性最广） |
| `openai-responses` | OpenAI Responses API |
| `anthropic-messages` | Anthropic Messages API |
| `google-generative-ai` | Google Generative AI |

`api` 字段可设置在提供商级别（作为所有模型的默认值），也可设置在模型级别（单独覆盖某个模型）。

## 提供商配置

| 字段 | 说明 |
| :--- | :--- |
| `baseUrl` | API 端点 URL |
| `api` | API 类型（见上文） |
| `apiKey` | API 密钥（见下方的值解析） |
| `headers` | 自定义头部（见下方的值解析） |
| `authHeader` | 设为 `true` 后，自动添加 `Authorization: Bearer <apiKey>` 头部 |
| `models` | 模型配置数组 |
| `modelOverrides` | 对该提供商的内置模型进行逐模型覆盖 |

### 值解析

`apiKey` 和 `headers` 字段支持命令执行、环境变量插值和字面量：

- **Shell 命令**：以 `"!command"` 开头的值会将整个值作为命令执行，并使用 stdout 作为结果
  ```json
  "apiKey": "!security find-generic-password -ws 'anthropic'"
  "apiKey": "!op read 'op://vault/item/credential'"
  ```

- **环境变量插值**：`$ENV_VAR` 或 `${ENV_VAR}` 使用指定变量的值。插值可在更大的字面量中使用。
  ```json
  "apiKey": "$MY_API_KEY"
  "apiKey": "${KEY_PREFIX}_${KEY_SUFFIX}"
  ```
  `$FOO_BAR` 会查找变量 `FOO_BAR`；如果想用 `${FOO}_BAR` 表示变量 `FOO` 后接字面量 `_BAR`，请明确使用花括号。缺失的环境变量会导致值无法解析。

- **转义**：`$$` 输出一个字面量 `$`；`$!` 输出一个字面量 `!` 而不触发命令执行。
  ```json
  "apiKey": "$$literal-dollar-prefix"
  "apiKey": "$!literal-bang-prefix"
  ```

- **字面量值**：直接使用
  ```json
  "apiKey": "sk-..."
  ```

遗留的大写环境变量格式（如 `MY_API_KEY`）在启动时会被迁移为 `$MY_API_KEY`。

对于 `models.json`，shell 命令在请求时解析。Pi 有意不为任意命令内置 TTL、过期复用或恢复逻辑。不同命令需要不同的缓存和故障策略，Pi 无法推断出正确的做法。

如果某个命令执行缓慢、昂贵、有速率限制，或者希望在瞬时故障时继续使用之前的值，请将它封装在你自己的脚本或命令中，由脚本实现你需要的缓存或 TTL 行为。

`/model` 的可用性检查仅依赖已配置的认证信息是否存在，不会执行 shell 命令。

### 自定义头部

```json
{
  "providers": {
    "custom-proxy": {
      "baseUrl": "https://proxy.example.com/v1",
      "apiKey": "$MY_API_KEY",
      "api": "anthropic-messages",
      "headers": {
        "x-portkey-api-key": "$PORTKEY_API_KEY",
        "x-secret": "!op read 'op://vault/item/secret'"
      },
      "models": [...]
    }
  }
}
```

## 模型配置

| 字段 | 必填 | 默认值 | 说明 |
| :--- | :--- | :--- | :--- |
| `id` | 是 | — | 模型标识符（发送给 API 的值） |
| `name` | 否 | `id` | 人类可读的模型标签。用于匹配（`--model` 模式）和模型详情/状态文本的显示。 |
| `api` | 否 | 提供商的 `api` | 为该模型覆盖提供商的 API 类型 |
| `reasoning` | 否 | `false` | 是否支持扩展思考 |
| `thinkingLevelMap` | 否 | 省略 | 将 Pi 的思考级别映射到提供商的值，并标记不支持的级别（见下文） |
| `input` | 否 | `["text"]` | 输入类型：`["text"]` 或 `["text", "image"]` |
| `contextWindow` | 否 | `128000` | 上下文窗口大小（以 token 计） |
| `maxTokens` | 否 | `16384` | 最大输出 token 数 |
| `cost` | 否 | 全为 0 | `{"input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0}`（每百万 token） |
| `compat` | 否 | 提供商的 `compat` | 提供商兼容性覆盖。当与提供商级别同时设置时，两者会合并。 |

**当前行为**：

- `/model` 和 `--list-models` 会按模型 `id` 列出条目。
- 已配置的 `name` 用于模型匹配和详情/状态文本。

### 思考级别映射（thinkingLevelMap）

在模型上使用 `thinkingLevelMap` 来描述模型特定的思考控制。键是 Pi 的思考级别：`off`, `minimal`, `low`, `medium`, `high`, `xhigh`。

值为三态：

| 值 | 含义 |
| :--- | :--- |
| 省略 | 该级别受支持，并使用提供商的默认映射 |
| 字符串 | 该级别受支持，并将此字符串值发送给提供商 |
| `null` | 该级别不受支持，会被隐藏/跳过/舍入 |

**示例：一个仅支持 off、high 和 max 推理的模型**：

```json
{
  "id": "deepseek-v4-pro",
  "reasoning": true,
  "thinkingLevelMap": {
    "minimal": null,
    "low": null,
    "medium": null,
    "high": "high",
    "xhigh": "max"
  }
}
```

**示例：一个无法关闭思考的模型**：

```json
{
  "id": "always-thinking-model",
  "reasoning": true,
  "thinkingLevelMap": {
    "off": null
  }
}
```

**迁移**：使用 `compat.reasoningEffortMap` 的旧配置应将映射迁移到模型级别的 `thinkingLevelMap`。对于不应出现在 UI 中的级别，使用 `null`。

## 覆盖内置提供商

将内置提供商路由到代理，而无需重新定义模型：

```json
{
  "providers": {
    "anthropic": {
      "baseUrl": "https://my-proxy.example.com/v1"
    }
  }
}
```

所有内置的 Anthropic 模型仍然可用。现有的 OAuth 或 API 密钥认证仍然有效。

要将自定义模型合并到内置提供商中，需包含 `models` 数组：

```json
{
  "providers": {
    "anthropic": {
      "baseUrl": "https://my-proxy.example.com/v1",
      "apiKey": "$ANTHROPIC_API_KEY",
      "api": "anthropic-messages",
      "models": [...]
    }
  }
}
```

合并语义：
- 内置模型会被保留。
- 自定义模型在提供商内部按 `id` 进行 upsert。
- 如果自定义模型的 `id` 与内置模型的 `id` 匹配，自定义模型将**替换**该内置模型。
- 如果自定义模型的 `id` 是新的，它会被**添加**到内置模型旁边。

## 逐模型覆盖

使用 `modelOverrides` 来自定义特定的内置模型，而无需替换提供商的完整模型列表。

```json
{
  "providers": {
    "openrouter": {
      "modelOverrides": {
        "anthropic/claude-sonnet-4": {
          "name": "Claude Sonnet 4 (Bedrock Route)",
          "compat": {
            "openRouterRouting": {
              "only": ["amazon-bedrock"]
            }
          }
        }
      }
    }
  }
}
```

`modelOverrides` 支持以下逐模型字段：`name`、`reasoning`、`input`、`cost`（部分）、`contextWindow`、`maxTokens`、`headers`、`compat`。

行为说明：
- `modelOverrides` 应用于内置提供商模型。
- 未知的模型 ID 会被忽略。
- 你可以将提供商级别的 `baseUrl`/`headers` 与 `modelOverrides` 结合使用。
- 如果提供商同时还定义了 `models`，自定义模型会在内置覆盖之后进行合并。具有相同 `id` 的自定义模型会替换已被覆盖的内置模型条目。

## Anthropic Messages 兼容性

对于使用 `api: "anthropic-messages"` 的提供商或代理，使用 `compat` 来控制 Anthropic 特定的请求兼容性。

默认情况下，Pi 会为每个工具发送 `eager_input_streaming: true`。如果某个代理或兼容 Anthropic 的后端拒绝该字段，可将 `supportsEagerToolInputStreaming` 设为 `false`。Pi 将省略 `tools[].eager_input_streaming`，并改为在启用工具的请求上发送旧的 `fine-grained-tool-streaming-2025-05-14` beta 头。

一些 Anthropic 模型要求使用自适应思考（`thinking.type: "adaptive"` 加上 `output_config.effort`），而非基于预算的旧版思考负载。内置模型会自动设置。对于路由到这些模型的自定义提供商或别名，可将 `forceAdaptiveThinking` 设为 `true`。

一些兼容 Anthropic 的提供商会发出签名块为空的思考块，并在重放时仍需要这些签名。仅对这类提供商将 `allowEmptySignature` 设为 `true`；真正的 Anthropic 会拒绝空的思考签名。

```json
{
  "providers": {
    "anthropic-proxy": {
      "baseUrl": "https://proxy.example.com",
      "api": "anthropic-messages",
      "apiKey": "$ANTHROPIC_PROXY_KEY",
      "compat": {
        "supportsEagerToolInputStreaming": false,
        "supportsLongCacheRetention": true,
        "forceAdaptiveThinking": true,
        "allowEmptySignature": true
      },
      "models": [
        {
          "id": "claude-opus-4-7",
          "reasoning": true,
          "input": ["text", "image"]
        }
      ]
    }
  }
}
```

| 字段 | 说明 |
| :--- | :--- |
| `supportsEagerToolInputStreaming` | 提供商是否接受逐工具的 `eager_input_streaming`。默认为 `true`。设为 `false` 则省略该字段，并在启用工具的请求上使用旧的细粒度工具流 beta 头。 |
| `supportsLongCacheRetention` | 当缓存保留策略为 long 时，提供商是否接受 Anthropic 的长缓存保留（`cache_control.ttl: "1h"`）。默认为 `true`。 |
| `sendSessionAffinityHeaders` | 启用缓存时是否从会话 ID 发送 `x-session-affinity`。默认为对已知提供商自动检测。 |
| `supportsCacheControlOnTools` | 提供商是否接受在工具定义上使用 Anthropic 风格的 `cache_control` 标记。默认为 `true`。 |
| `forceAdaptiveThinking` | 是否为此模型发送自适应思考（`thinking.type: "adaptive"` 加上 `output_config.effort`）。内置的自适应模型会自动设置。默认为 `false`。 |
| `allowEmptySignature` | 是否将空的思考签名重放为 `signature: ""`，而不是将思考转为文本。默认为 `false`。 |

## OpenAI 兼容性

对于部分兼容 OpenAI 的提供商，使用 `compat` 字段。

- 提供商级别的 `compat` 将默认值应用于该提供商下的所有模型。
- 模型级别的 `compat` 会覆盖该模型的提供商级别值。

```json
{
  "providers": {
    "local-llm": {
      "baseUrl": "http://localhost:8080/v1",
      "api": "openai-completions",
      "compat": {
        "supportsUsageInStreaming": false,
        "maxTokensField": "max_tokens"
      },
      "models": [...]
    }
  }
}
```

| 字段 | 说明 |
| :--- | :--- |
| `supportsStore` | 提供商是否支持 `store` 字段 |
| `supportsDeveloperRole` | 使用 `developer` 角色还是 `system` 角色 |
| `supportsReasoningEffort` | 是否支持 `reasoning_effort` 参数 |
| `supportsUsageInStreaming` | 是否支持 `stream_options: { include_usage: true }`（默认：`true`） |
| `maxTokensField` | 使用 `max_completion_tokens` 还是 `max_tokens` |
| `requiresToolResultName` | 在工具结果消息上包含 `name` |
| `requiresAssistantAfterToolResult` | 在工具结果之后的用户消息前插入一条助手消息 |
| `requiresThinkingAsText` | 将思考块转换为纯文本 |
| `requiresReasoningContentOnAssistantMessages` | 在启用推理时，在所有重放的助手消息上包含空的 `reasoning_content` |
| `thinkingFormat` | 使用 `reasoning_effort`、`openrouter`、`deepseek`、`together`、`zai`、`qwen` 或 `qwen-chat-template` 思考参数 |
| `cacheControlFormat` | 在系统提示、最后一个工具定义以及最后一个用户/助手文本内容上使用 Anthropic 风格的 `cache_control` 标记。目前仅支持 `anthropic`。 |
| `supportsStrictMode` | 在工具定义中包含 `strict` 字段 |
| `supportsLongCacheRetention` | 当缓存保留策略为 long 时，提供商是否接受长缓存保留：对 OpenAI 提示缓存使用 `prompt_cache_retention: "24h"`；当 `cacheControlFormat` 为 `anthropic` 时使用 `cache_control.ttl: "1h"`。默认为 `true`。 |
| `openRouterRouting` | OpenRouter 提供商路由偏好。此对象会原样作为 OpenRouter API 请求中的 `provider` 字段发送。 |
| `vercelGatewayRouting` | Vercel AI Gateway 的路由配置，用于提供商选择（`only`、`order`） |

- `openrouter` 使用 `reasoning: { effort }`。
- `together` 使用 `reasoning: { enabled }`，并在启用 `supportsReasoningEffort` 时同时使用 `reasoning_effort`。
- `qwen` 使用顶层的 `enable_thinking`。
- `qwen-chat-template` 用于需要 `chat_template_kwargs.enable_thinking` 的本地兼容 Qwen 服务器。
- `cacheControlFormat: "anthropic"` 适用于兼容 OpenAI 但通过 `cache_control` 标记在文本内容和工具定义上暴露 Anthropic 风格提示缓存的提供商。

**OpenRouter 示例**：

```json
{
  "providers": {
    "openrouter": {
      "baseUrl": "https://openrouter.ai/api/v1",
      "apiKey": "$OPENROUTER_API_KEY",
      "api": "openai-completions",
      "models": [
        {
          "id": "openrouter/anthropic/claude-3.5-sonnet",
          "name": "OpenRouter Claude 3.5 Sonnet",
          "compat": {
            "openRouterRouting": {
              "allow_fallbacks": true,
              "require_parameters": false,
              "data_collection": "deny",
              "zdr": true,
              "enforce_distillable_text": false,
              "order": ["anthropic", "amazon-bedrock", "google-vertex"],
              "only": ["anthropic", "amazon-bedrock"],
              "ignore": ["gmicloud", "friendli"],
              "quantizations": ["fp16", "bf16"],
              "sort": {
                "by": "price",
                "partition": "model"
              },
              "max_price": {
                "prompt": 10,
                "completion": 20
              },
              "preferred_min_throughput": {
                "p50": 100,
                "p90": 50
              },
              "preferred_max_latency": {
                "p50": 1,
                "p90": 3,
                "p99": 5
              }
            }
          }
        }
      ]
    }
  }
}
```

**Vercel AI Gateway 示例**：

```json
{
  "providers": {
    "vercel-ai-gateway": {
      "baseUrl": "https://ai-gateway.vercel.sh/v1",
      "apiKey": "$AI_GATEWAY_API_KEY",
      "api": "openai-completions",
      "models": [
        {
          "id": "moonshotai/kimi-k2.5",
          "name": "Kimi K2.5 (Fireworks via Vercel)",
          "reasoning": true,
          "input": ["text", "image"],
          "cost": { "input": 0.6, "output": 3, "cacheRead": 0, "cacheWrite": 0 },
          "contextWindow": 262144,
          "maxTokens": 262144,
          "compat": {
            "vercelGatewayRouting": {
              "only": ["fireworks", "novita"],
              "order": ["fireworks", "novita"]
            }
          }
        }
      ]
    }
  }
}
```

---

## 💡 自定义模型系统详解与示例

Pi 的自定义模型系统是连接本地模型、私有化部署、云厂商托管模型以及各种 API 代理的“万能适配器”。通过一个简单的 `models.json` 文件，你无需修改 Pi 的任何源代码，就能接入任何兼容 OpenAI Chat Completions、Anthropic Messages 或 Google Generative AI 协议的模型。这使 Pi 从一个依赖外部 API 的工具，变为可完全自托管的私有化 AI 助手。

### 1. 核心理念：声明式配置、热加载、万能兼容

整个系统基于一个中心配置文件 `~/.pi/agent/models.json`。你只需要按照规范声明提供商的地址、协议类型和模型列表，Pi 在运行时会自动读取并集成。文件在每次打开 `/model` 切换界面时重新加载，因此你可以在会话中编辑配置，无需重启。对于经常调整本地模型参数的开发者，这极其便利。

### 2. 三步接入本地模型（以 Ollama 为例）

假设你在本地使用 Ollama 运行了 `llama3.1:8b` 和 `qwen2.5-coder:7b` 两个模型。典型的接入步骤如下：

**第一步：创建或编辑 `~/.pi/agent/models.json`**

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "models": [
        { "id": "llama3.1:8b" },
        { "id": "qwen2.5-coder:7b" }
      ]
    }
  }
}
```

- `baseUrl` 指向 Ollama 的本地 API 地址。
- `api` 设为 `"openai-completions"`，因为 Ollama 提供了兼容 OpenAI Chat Completions 的端点。
- `apiKey` 是必填字段，但 Ollama 不验证密钥，可填任意值（这里填 `"ollama"` 仅作占位）。

**第二步：在 Pi 中切换模型**

启动 Pi 后，输入 `/model` 打开模型选择器。你会看到新添加的 `llama3.1:8b` 和 `qwen2.5-coder:7b` 出现在列表中，与内置的远程模型并列。选择其中一个即可开始使用本地模型进行对话。

**第三步（可选）：微调兼容性**

某些本地模型（如 Qwen）在推理时可能需要特殊的参数格式。如果你的模型启用了推理（`"reasoning": true`），但 Ollama 不支持 `developer` 角色和 `reasoning_effort` 参数，那么需要加 `compat` 字段：

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "compat": {
        "supportsDeveloperRole": false,
        "supportsReasoningEffort": false
      },
      "models": [
        {
          "id": "gpt-oss:20b",
          "reasoning": true
        }
      ]
    }
  }
}
```

这让 Pi 使用 `system` 角色发送系统提示，并省略 `reasoning_effort`，保证与 Ollama 的兼容。

### 3. 通过代理访问远程模型

许多团队通过 API 代理（如 LiteLLM、Portkey 或自建网关）来统一管理模型访问。Pi 可轻松配置指向代理。

**示例：指向企业内部的 Anthropic 代理**

```json
{
  "providers": {
    "corp-anthropic": {
      "baseUrl": "https://ai-proxy.corp.com/v1",
      "api": "anthropic-messages",
      "apiKey": "$CORP_API_KEY",
      "headers": {
        "x-custom-header": "value"
      },
      "models": [
        {
          "id": "claude-sonnet-4-20250514",
          "reasoning": true,
          "input": ["text", "image"],
          "contextWindow": 200000,
          "maxTokens": 64000,
          "cost": { "input": 3, "output": 15, "cacheRead": 0.3, "cacheWrite": 6 }
        }
      ]
    }
  }
}
```

这里 `apiKey` 使用了环境变量引用 `$CORP_API_KEY`，确保密钥不会硬编码进文件。

### 4. 覆盖内置提供商，添加自定义模型

假设你希望在使用 Anthropic 的同时，增加一个自己微调的 Claude 变体。可以直接覆盖内置的 `anthropic` 提供商：

```json
{
  "providers": {
    "anthropic": {
      "baseUrl": "https://my-fine-tuned-proxy.example.com",
      "apiKey": "$ANTHROPIC_API_KEY",
      "api": "anthropic-messages",
      "models": [
        {
          "id": "my-fine-tuned-claude",
          "name": "My Fine-Tuned Claude",
          "reasoning": false,
          "input": ["text"],
          "contextWindow": 200000,
          "maxTokens": 32000
        }
      ]
    }
  }
}
```

这样内置的 `claude-sonnet-4-5` 等仍然可用，同时列表里会多出一个 `my-fine-tuned-claude`。

### 5. 高级路由控制（OpenRouter / Vercel AI Gateway）

对于使用 OpenRouter 或 Vercel AI Gateway 的用户，Pi 支持通过 `compat` 精细控制路由行为，包括按价格排序、指定允许的提供商、设置延迟上限等。

**OpenRouter 示例：仅使用 Bedrock 上的 Claude**

```json
{
  "providers": {
    "openrouter": {
      "baseUrl": "https://openrouter.ai/api/v1",
      "apiKey": "$OPENROUTER_API_KEY",
      "api": "openai-completions",
      "models": [
        {
          "id": "anthropic/claude-sonnet-4",
          "name": "Claude Sonnet 4 (Bedrock Only)",
          "compat": {
            "openRouterRouting": {
              "only": ["amazon-bedrock"],
              "max_price": { "prompt": 3, "completion": 15 }
            }
          }
        }
      ]
    }
  }
}
```

这确保请求仅路由到 Amazon Bedrock，避免被分发到其他提供商。

### 6. 密钥安全：动态命令执行

对于安全要求较高的环境，`apiKey` 支持从 macOS Keychain 或 1Password CLI 动态获取：

```json
"apiKey": "!security find-generic-password -ws 'anthropic'"
"apiKey": "!op read 'op://vault/item/credential'"
```

这避免了在配置文件中明文写入密钥。`!` 前缀告诉 Pi 在每次请求时执行该命令，并用其标准输出作为 API 密钥。需要注意的是，Pi 不会对命令输出做缓存，如果你需要缓存或重试逻辑，应自行编写包装脚本。

### 7. 思考级别映射：适配不同模型的推理控制

不同厂商对“推理深度”的命名各不相同。Pi 提供了 `thinkingLevelMap` 来适配：

```json
{
  "id": "deepseek-v4-pro",
  "reasoning": true,
  "thinkingLevelMap": {
    "minimal": null,
    "low": null,
    "medium": null,
    "high": "high",
    "xhigh": "max"
  }
}
```

这个配置表示该模型只支持 `high`（映射为 `"high"`）和 `xhigh`（映射为 `"max"`），其他级别被设为 `null` 从而在 UI 中隐藏或不可选。

### 8. 成本追踪

`cost` 字段允许你为自定义模型指定每百万 token 的价格。这样 Pi 在底部状态栏中就能准确显示本次会话的预估费用，这对于需要控制成本的团队非常重要。

### 9. 总结

Pi 的自定义模型系统通过一个声明式的 JSON 配置文件，实现了对几乎所有主流模型服务协议的支持。其核心优势在于：

- **极低接入成本**：Ollama 只需 6 行 JSON。
- **热加载**：编辑配置即可生效，无需重启。
- **深度兼容性适配**：通过 `compat` 字段可以精细处理各厂商的差异。
- **安全性**：支持从系统 Keychain 或密码管理器动态获取密钥。
- **高级路由**：对 OpenRouter 和 Vercel AI Gateway 的一等支持。

这使 Pi 成为一个真正中立的 AI 交互层：无论模型运行在本地、企业代理、还是公有云，你都能用完全相同的终端界面和工作流与它们交互。