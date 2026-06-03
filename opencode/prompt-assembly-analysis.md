# OpenCode 提示词拼装机制：AGENTS.md 与 Build 智能体提示词的协调逻辑

## 概述

本文分析 OpenCode 如何将 AGENTS.md 等指令文件与 Build 智能体自身的提示词拼装为最终发送给 LLM 的 system prompt。分析基于 `packages/opencode/src/` 下的源码。

---

## 1. 全链路数据流

```
用户发送消息
    │
    ▼
SessionPrompt.prompt()                    ← prompt.ts:158
    │  解析 agent（默认 build）
    │  创建 user message
    ▼
loop()                                    ← prompt.ts:274
    │  加载消息历史
    │  解析 agent 定义
    │  插入 reminder（plan/build 切换等）
    ▼
构建 system 数组                          ← prompt.ts:651
    │  system = [
    │    ...SystemPrompt.environment(model),    ← 环境上下文
    │    ...InstructionPrompt.system()          ← AGENTS.md 等指令文件
    │  ]
    ▼
LLM.stream()                              ← llm.ts:46
    │  最终拼装 system prompt
    │  发送给 AI SDK → Provider API
    ▼
API 请求
```

---

## 2. Build 智能体自身的提示词从哪来

### 2.1 Agent 定义

Build 智能体在 `agent/agent.ts:77-91` 中定义：

```ts
build: {
  name: "build",
  description: "The default agent. Executes tools based on configured permissions.",
  options: {},
  permission: PermissionNext.merge(defaults, ..., user),
  mode: "primary",
  native: true,
  // 注意：没有 prompt 字段！
}
```

**关键点：Build 智能体没有 `prompt` 字段。** 这决定了它使用 provider 模板而非自有提示词。

### 2.2 Provider 模板选择

当 agent 没有 `prompt` 字段时，`SystemPrompt.provider(model)` 根据 model ID 选择模板（`system.ts:19-27`）：

```ts
export function provider(model: Provider.Model) {
  if (model.api.id.includes("gpt-5")) return [PROMPT_CODEX]; // codex_header.txt
  if (
    model.api.id.includes("gpt-") ||
    model.api.id.includes("o1") ||
    model.api.id.includes("o3")
  )
    return [PROMPT_BEAST]; // beast.txt
  if (model.api.id.includes("gemini-")) return [PROMPT_GEMINI]; // gemini.txt
  if (model.api.id.includes("claude")) return [PROMPT_ANTHROPIC]; // anthropic.txt
  if (model.api.id.toLowerCase().includes("trinity")) return [PROMPT_TRINITY]; // trinity.txt
  return [PROMPT_ANTHROPIC_WITHOUT_TODO]; // qwen.txt（兜底）
}
```

**模板选择规则汇总：**

| Model ID 包含      | 模板文件           | 说明                     |
| ------------------ | ------------------ | ------------------------ |
| `gpt-5`            | `codex_header.txt` | GPT-5 / Codex 专用       |
| `gpt-`、`o1`、`o3` | `beast.txt`        | OpenAI GPT/o 系列        |
| `gemini-`          | `gemini.txt`       | Google Gemini            |
| `claude`           | `anthropic.txt`    | Anthropic Claude（默认） |
| `trinity`          | `trinity.txt`      | Trinity 模型             |
| 其他               | `qwen.txt`         | 兜底模板                 |

### 2.3 各模板的核心差异

| 模板            | 核心风格      | 关键特征                                                             |
| --------------- | ------------- | -------------------------------------------------------------------- |
| `anthropic.txt` | 简洁 CLI 助手 | 强调 TodoWrite 工具使用、Task 子智能体委派、代码引用格式 `file:line` |
| `beast.txt`     | 自主迭代型    | 强调"必须持续工作直到问题解决"、互联网研究、严格测试验证             |
| `gemini.txt`    | 结构化工作流  | 5 步工作流（理解→计划→实现→验证→反馈）、安全优先                     |
| `qwen.txt`      | 极简 CLI 助手 | 与 anthropic.txt 类似但更精简，无 TodoWrite 示例                     |

---

## 3. AGENTS.md 的发现与加载

### 3.1 发现路径优先级

`InstructionPrompt.systemPaths()`（`instruction.ts:72-115`）按以下顺序发现指令文件：

```
┌─────────────────────────────────────────────────────────┐
│ 第 1 层：项目目录向上遍历                                   │
│                                                          │
│ 从 Instance.directory 向上遍历到 Instance.worktree        │
│ 按优先级查找：AGENTS.md → CLAUDE.md → CONTEXT.md         │
│ ⚠️ 首次匹配即停止（break），不会同时加载多个                │
│                                                          │
│ 例：/project/packages/opencode/src/                      │
│     → 查找 AGENTS.md → 无 → CLAUDE.md → 无 → CONTEXT.md │
│     → 向上到 /project/packages/opencode/                  │
│     → 查找 AGENTS.md → 找到！停止                         │
├─────────────────────────────────────────────────────────┤
│ 第 2 层：全局配置目录                                      │
│                                                          │
│ ① $OPENCODE_CONFIG_DIR/AGENTS.md（如果设置了环境变量）     │
│ ② Global.Path.config/AGENTS.md（opencode 全局配置目录）   │
│ ③ ~/.claude/CLAUDE.md（除非设置了                         │
│    OPENCODE_DISABLE_CLAUDE_CODE_PROMPT）                  │
│ ⚠️ 三者中只加载第一个存在的（break）                       │
├─────────────────────────────────────────────────────────┤
│ 第 3 层：config.instructions                             │
│                                                          │
│ opencode.json 中的 instructions 数组                      │
│ - 本地文件：glob 模式向上遍历匹配                          │
│ - URL（http/https）：跳过，在 system() 中单独 fetch       │
│ - ~/ 开头：展开为 home 目录路径                            │
│ - 绝对路径：在路径所在目录执行 glob 匹配                    │
└─────────────────────────────────────────────────────────┘
```

**关键细节：**

1. **第 1 层的 `findUp` 行为**（`filesystem.ts:138-150`）：从 `Instance.directory` 开始，逐级向上到 `Instance.worktree`，在每一级查找目标文件。`findUp` 会收集**所有层级**的匹配结果（不会 break），但 `systemPaths()` 中对 `FILES` 数组的遍历在**首次有匹配时 break**——这意味着如果 `/project/AGENTS.md` 存在，就不会再查找 `CLAUDE.md`。

2. **第 2 层的 break 逻辑**：`globalFiles()` 返回最多 3 个候选路径，`systemPaths()` 对它们逐一检查 `exists()`，**第一个存在的即加入并 break**。

3. **去重**：所有路径存入 `Set<string>`，自动去重。

### 3.2 内容加载与格式化

`InstructionPrompt.system()`（`instruction.ts:117-142`）读取所有发现的文件：

```ts
const files = Array.from(paths).map(async (p) => {
  const content = await Filesystem.readText(p).catch(() => "");
  return content ? "Instructions from: " + p + "\n" + content : "";
});
```

**输出格式**：每个文件的内容前缀 `"Instructions from: <绝对路径>\n"`，空文件被过滤。

URL 类型的 instructions 单独 fetch（5 秒超时），格式相同：`"Instructions from: <url>\n<content>"`。

---

## 4. 最终拼装：Build 智能体提示词 + AGENTS.md 如何合并

### 4.1 核心拼装逻辑

拼装发生在 `LLM.stream()` 中（`llm.ts:67-80`）：

```ts
const system = [];
system.push(
  [
    // ① agent 提示词 或 provider 模板
    ...(input.agent.prompt
      ? [input.agent.prompt]
      : isCodex
        ? []
        : SystemPrompt.provider(input.model)),
    // ② 自定义 system（来自 loop 构建的 system 数组）
    ...input.system,
    // ③ 用户消息中的 system 字段
    ...(input.user.system ? [input.user.system] : []),
  ]
    .filter((x) => x)
    .join("\n"),
);
```

### 4.2 Build 智能体的拼装路径

对于 Build 智能体，`input.agent.prompt` 为 `undefined`，因此走 `SystemPrompt.provider(input.model)` 分支。

**完整拼装顺序：**

```
┌──────────────────────────────────────────────────────────────┐
│ 最终 system prompt（单个字符串，用 \n 连接）                    │
│                                                               │
│ ┌──────────────────────────────────────────────────────────┐ │
│ │ ① Provider 模板                                          │ │
│ │    SystemPrompt.provider(model) 的返回值                  │ │
│ │    例如：anthropic.txt 的完整内容                          │ │
│ └──────────────────────────────────────────────────────────┘ │
│                          \n                                   │
│ ┌──────────────────────────────────────────────────────────┐ │
│ │ ② 环境上下文                                              │ │
│ │    SystemPrompt.environment(model) 的返回值                │ │
│ │    包含：model ID、工作目录、git 状态、平台、日期           │ │
│ │    格式：                                                 │ │
│ │    You are powered by the model named <id>...             │ │
│ │    <env>                                                  │ │
│ │      Working directory: /path/to/project                  │ │
│ │      Is directory a git repo: yes                         │ │
│ │      Platform: win32                                      │ │
│ │      Today's date: Wed Jun 03 2026                        │ │
│ │    </env>                                                 │ │
│ └──────────────────────────────────────────────────────────┘ │
│                          \n                                   │
│ ┌──────────────────────────────────────────────────────────┐ │
│ │ ③ AGENTS.md / CLAUDE.md 等指令文件内容                    │ │
│ │    InstructionPrompt.system() 的返回值                     │ │
│ │    每个文件格式：                                          │ │
│ │    Instructions from: /path/to/AGENTS.md                  │ │
│ │    <AGENTS.md 的完整内容>                                  │ │
│ │    （多个文件依次排列，用 \n 分隔）                         │ │
│ └──────────────────────────────────────────────────────────┘ │
│                          \n                                   │
│ ┌──────────────────────────────────────────────────────────┐ │
│ │ ④ 用户消息的 system 字段（可选）                           │ │
│ │    input.user.system                                      │ │
│ │    通过 PromptInput.system 传入                           │ │
│ └──────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

### 4.3 具体示例

假设使用 Claude 模型，项目根目录有 `AGENTS.md`，全局有 `~/.claude/CLAUDE.md`：

```
[anthropic.txt 完整内容]

You are powered by the model named claude-sonnet-4-20250514. The exact model ID is anthropic/claude-sonnet-4-20250514
Here is some useful information about the environment you are running in:
<env>
  Working directory: F:\projects\opencode
  Is directory a git repo: yes
  Platform: win32
  Today's date: Wed Jun 03 2026
</env>
<directories>

</directories>

Instructions from: F:\projects\opencode\AGENTS.md
# Agent Guidelines for OpenCode Repository
...（AGENTS.md 完整内容）...

Instructions from: C:\Users\Admin\.claude\CLAUDE.md
<!-- CODEGRAPH_START -->
...（CLAUDE.md 完整内容）...
```

---

## 5. Agent 提示词的替换 vs 追加机制

### 5.1 决策树

```
agent.prompt 是否存在？
    │
    ├─ 是 → 使用 agent.prompt 替换 provider 模板
    │        （explore、compaction、title、summary、自定义 agent）
    │
    └─ 否 → 是否为 Codex 会话？
             │
             ├─ 是 → 不注入任何 provider 模板
             │        （通过 options.instructions 单独发送 codex_header.txt）
             │
             └─ 否 → 使用 SystemPrompt.provider(model) 选择模板
                      （build、plan 智能体走此路径）
```

### 5.2 各 Agent 的提示词来源

| Agent        | `prompt` 字段       | 提示词来源                       | AGENTS.md 是否追加 |
| ------------ | ------------------- | -------------------------------- | ------------------ |
| **build**    | 无                  | `SystemPrompt.provider(model)`   | 是                 |
| **plan**     | 无                  | `SystemPrompt.provider(model)`   | 是                 |
| explore      | `PROMPT_EXPLORE`    | 自有提示词（替换 provider 模板） | 是                 |
| compaction   | `PROMPT_COMPACTION` | 自有提示词（替换 provider 模板） | 是                 |
| title        | `PROMPT_TITLE`      | 自有提示词（替换 provider 模板） | 是                 |
| summary      | `PROMPT_SUMMARY`    | 自有提示词（替换 provider 模板） | 是                 |
| 自定义 agent | markdown 文件内容   | 自有提示词（替换 provider 模板） | 是                 |

**核心结论：无论哪个 agent，AGENTS.md 等指令文件始终追加在提示词之后。** `agent.prompt` 只决定"基础提示词"是 provider 模板还是自有提示词，不影响指令文件的追加。

---

## 6. 特殊情况：Codex / GPT-5 会话

当 provider 为 OpenAI 且认证方式为 OAuth 时（`llm.ts:65`）：

```ts
const isCodex = provider.id === "openai" && auth?.type === "oauth";
```

Codex 会话的处理有两处不同：

1. **系统消息中不注入 provider 模板**（`llm.ts:72`）：

   ```ts
   ...(input.agent.prompt ? [input.agent.prompt] : isCodex ? [] : SystemPrompt.provider(input.model))
   //                                                    ^^^^^^ 空数组
   ```

2. **通过 `options.instructions` 发送**（`llm.ts:110-112`）：
   ```ts
   if (isCodex) {
     options.instructions = SystemPrompt.instructions(); // codex_header.txt
   }
   ```

这是因为 Codex API 有自己的 `instructions` 传递机制，不走标准 system message。

---

## 7. Read 工具中的 AGENTS.md 二次注入

除了系统提示词中的全局注入，`InstructionPrompt.resolve()`（`instruction.ts:168-191`）还会在 Read 工具读取文件时，发现该文件父目录中的 AGENTS.md/CLAUDE.md 并注入到工具输出中。

```
Read 工具读取 /project/src/session/prompt.ts
    │
    ▼
InstructionPrompt.resolve(messages, filepath, messageID)
    │  从 filepath 的父目录向上遍历到 Instance.directory
    │  在每一级查找 AGENTS.md / CLAUDE.md / CONTEXT.md
    │
    │  过滤条件（全部满足才注入）：
    │  ① found !== target（不是正在读取的文件本身）
    │  ② !system.has(found)（不在系统提示词已加载的集合中）
    │  ③ !already.has(found)（不在已加载历史中）
    │  ④ !isClaimed(messageID, found)（不在当前消息已声明集合中）
    │
    ▼
注入格式："Instructions from: <path>\n<content>"
```

**去重机制**：`InstructionPrompt.loaded()`（`instruction.ts:144-159`）扫描历史消息中所有已完成的 Read 工具调用，收集其 `metadata.loaded` 字段，避免同一文件被重复注入。

---

## 8. Plugin 钩子对提示词的修改

### 8.1 系统提示词变换

```ts
// llm.ts:83-87
await Plugin.trigger(
  "experimental.chat.system.transform",
  { sessionID, model },
  { system },
);
```

插件可以在最终拼装后修改 `system` 数组。变换后，如果 `system[0]`（主提示词）未变且数组超过 2 项，会重新合并为 `[header, rest.join("\n")]` 的两段结构，以优化 Anthropic 的 prompt caching。

### 8.2 消息变换

```ts
// prompt.ts:648
await Plugin.trigger(
  "experimental.chat.messages.transform",
  {},
  { messages: msgs },
);
```

插件可以在消息发送前修改消息历史。

---

## 9. 完整拼装时序图（Build 智能体 + Claude 模型）

```
时间 ──────────────────────────────────────────────────────────►

SessionPrompt.prompt()
  │
  ├─ Agent.get("build") → { name: "build", prompt: undefined, ... }
  │
  ├─ createUserMessage() → 保存用户消息
  │
  └─ loop()
      │
      ├─ Agent.get(lastUser.agent) → build agent
      │
      ├─ insertReminders() → 注入 plan/build 切换提醒
      │
      ├─ 构建系统提示词数组 ──────────────────────────────────┐
      │   │                                                   │
      │   ├─ SystemPrompt.environment(model)                  │
      │   │   → ["You are powered by...<env>...</env>..."]    │
      │   │                                                   │
      │   └─ InstructionPrompt.system()                       │
      │       ├─ systemPaths()                                │
      │       │   ├─ 项目目录 findUp AGENTS.md → 找到         │
      │       │   ├─ 全局目录 → ~/.claude/CLAUDE.md → 找到    │
      │       │   └─ config.instructions → 无额外文件         │
      │       ├─ 读取文件内容                                  │
      │       └─ ["Instructions from: ...AGENTS.md\n...",     │
      │          "Instructions from: ...CLAUDE.md\n..."]      │
      │                                                       │
      │   system = [环境上下文, AGENTS.md内容, CLAUDE.md内容]  │
      └───────────────────────────────────────────────────────┘
      │
      └─ LLM.stream()
          │
          ├─ 最终拼装 system[0] ──────────────────────────────┐
          │   [                                               │
          │     SystemPrompt.provider(model),  → anthropic.txt │
          │     ...input.system,              → 环境+指令文件  │
          │     input.user.system,            → undefined      │
          │   ].filter(x => x).join("\n")                     │
          └───────────────────────────────────────────────────┘
          │
          ├─ Plugin.trigger("experimental.chat.system.transform")
          │   → 插件可修改 system 数组
          │
          ├─ 构造 API 消息
          │   messages = [
          │     { role: "system", content: system[0] },  ← 主提示词
          │     { role: "system", content: system[1] },  ← 插件追加（如有）
          │     ...conversation history
          │   ]
          │
          └─ streamText() → Provider API
```

---

## 10. 关键代码位置索引

| 功能                                 | 文件                     | 行号        |
| ------------------------------------ | ------------------------ | ----------- |
| Agent 定义（build 无 prompt）        | `agent/agent.ts`         | 77-91       |
| Provider 模板选择                    | `session/system.ts`      | 19-27       |
| 环境上下文生成                       | `session/system.ts`      | 29-53       |
| AGENTS.md 发现路径                   | `session/instruction.ts` | 72-115      |
| AGENTS.md 内容加载                   | `session/instruction.ts` | 117-142     |
| Read 工具中的 AGENTS.md 注入         | `session/instruction.ts` | 168-191     |
| 已加载去重                           | `session/instruction.ts` | 144-159     |
| 系统提示词数组构建                   | `session/prompt.ts`      | 651         |
| 最终拼装（agent.prompt vs provider） | `session/llm.ts`         | 67-80       |
| Codex 特殊处理                       | `session/llm.ts`         | 65, 110-112 |
| Plugin 系统提示词变换                | `session/llm.ts`         | 83-93       |
| API 消息构造                         | `session/llm.ts`         | 225-233     |
| findUp 向上遍历                      | `util/filesystem.ts`     | 138-150     |
| globUp 向上遍历                      | `util/filesystem.ts`     | 167-188     |
