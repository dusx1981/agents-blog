# OpenCode 执行策略深度分析报告

> 生成日期：2026-06-02
> 基于 OpenCode 源码逐层追踪，涵盖串行/并行执行、子智能体派发、权限控制、重试与压缩等全链路

---

## 目录

1. [架构总览：三层执行模型](#1-架构总览三层执行模型)
2. [串行执行引擎：SessionPrompt.loop()](#2-串行执行引擎sessionpromptloop)
3. [并行执行机制：AI SDK streamText()](#3-并行执行机制ai-sdk-streamtext)
4. [流处理器：SessionProcessor](#4-流处理器sessionprocessor)
5. [子智能体执行：TaskTool](#5-子智能体执行tasktool)
6. [显式并行：Batch 工具](#6-显式并行batch-工具)
7. [Command 子任务路径：SubtaskPart](#7-command-子任务路径subtaskpart)
8. [Agent 定义与能力矩阵](#8-agent-定义与能力矩阵)
9. [权限系统](#9-权限系统)
10. [步骤限制与 Doom Loop 检测](#10-步骤限制与-doom-loop-检测)
11. [重试机制](#11-重试机制)
12. [上下文压缩（Compaction）](#12-上下文压缩compaction)
13. [串行 vs 并行决策模型](#13-串行-vs-并行决策模型)
14. [完整执行流程图](#14-完整执行流程图)
15. [关键代码路径索引](#15-关键代码路径索引)

---

## 1. 架构总览：三层执行模型

OpenCode 的执行引擎由三个嵌套层组成，每层有不同的串行/并行语义：

```
┌─────────────────────────────────────────────────────────────────┐
│ 第1层：SessionPrompt.loop() — 串行步骤循环                       │
│   while(true) { 逐步推进，每步处理一组工具调用 }                  │
│   源码：src/session/prompt.ts:274-713                            │
├─────────────────────────────────────────────────────────────────┤
│ 第2层：Vercel AI SDK streamText() — 同步并行执行                  │
│   同一 LLM 响应中的多个 tool_calls → 自动 Promise.all 并行       │
│   源码：src/session/llm.ts:172-255                               │
├─────────────────────────────────────────────────────────────────┤
│ 第3层：TaskTool.execute() — 子智能体独立会话                      │
│   每个 Task 创建 child session，拥有自己的 loop()                 │
│   源码：src/tool/task.ts:27-165                                  │
└─────────────────────────────────────────────────────────────────┘
```

**核心原则：**

| 原则             | 说明                                                       |
| ---------------- | ---------------------------------------------------------- |
| **同消息并行**   | 同一条 LLM 响应中的多个 Task/tool 调用 → 并行执行          |
| **跨消息串行**   | 不同 LLM 响应轮次中的 Task/tool 调用 → 串行执行            |
| **决策权在 LLM** | OpenCode 运行时不做智能决策，忠实执行 LLM 生成的调用模式   |
| **上下文隔离**   | 子智能体不继承主代理的会话历史，只接收 prompt 中提供的信息 |
| **权限边界独立** | 子智能体有独立的权限配置，主代理不可越权                   |

---

## 2. 串行执行引擎：SessionPrompt.loop()

**源码位置**：`src/session/prompt.ts:274-713`

这是整个执行引擎的核心，是一个严格的 `while(true)` 串行循环。

### 2.1 入口与状态初始化

```typescript
// prompt.ts:158-185
export const prompt = fn(PromptInput, async (input) => {
  const session = await Session.get(input.sessionID)
  // ... 创建用户消息、处理权限 ...
  return loop({ sessionID: input.sessionID }) // ← 进入主循环
})
```

```typescript
// prompt.ts:274-285
export const loop = fn(LoopInput, async (input) => {
  const { sessionID, resume_existing } = input
  const abort = resume_existing ? resume(sessionID) : start(sessionID)
  // 如果 session 已在运行，排队等待
  if (!abort) {
    return new Promise<MessageV2.WithParts>((resolve, reject) => {
      const callbacks = state()[sessionID].callbacks
      callbacks.push({ resolve, reject })
    })
  }
  using _ = defer(() => cancel(sessionID))
  let step = 0
  // ... 进入 while(true) 循环
})
```

### 2.2 主循环流程

```
loop() 入口
    │
    ▼
┌──────────────────────────────────────────────┐
│ step++                                        │
│ 读取所有消息: msgs = MessageV2.stream()        │
│ 逆向遍历查找:                                 │
│  - lastUser (最后用户消息)                     │
│  - lastAssistant (最后助手消息)                │
│  - lastFinished (最后完成的助手消息)           │
│  - tasks[] (pending 的 subtask/compaction)    │
└──────────┬───────────────────────────────────┘
           │
     ┌─────▼──────────────────────────┐
     │ lastAssistant 已完成 且         │
     │ finish ≠ "tool-calls"/"unknown"│
     │ 且 lastUser.id < lastAssistant.id?
     └──┬──────────────────────┬──────┘
      是│                     │否
        ▼                     ▼
     break 退出循环       继续处理
                               │
           ┌───────────────────▼───────────────────┐
           │ 从 tasks 队列弹出一个 pending task     │
           └──┬──────────────────────────────┬─────┘
              │                              │
     ┌────────▼────────┐           ┌─────────▼─────────┐
     │ type=subtask?   │           │ type=compaction?  │
     └──┬──────────┬───┘           └──┬───────────┬───┘
       是│        │否               是│           │否
         ▼        ▼                   ▼            ▼
   串行执行     检查 context      处理 compaction  检查 overflow
   TaskTool     overflow?         → continue       → 创建 compaction
   → continue                     (prompt.ts:529)   → continue

           ┌───────────────────────────────────────▼───────────────────┐
           │ 正常处理：LLM 调用                                        │
           │ 1. resolveTools() — 根据权限过滤可用工具                   │
           │ 2. SessionProcessor.create() — 创建流处理器                │
           │ 3. processor.process() — 流式处理 LLM 响应                │
           │ 4. 检查返回值: "stop" → break, "compact" → 创建压缩       │
           │ 5. continue → 回到循环顶部                                 │
           └──────────────────────────────────────────────────────────┘
```

### 2.3 退出条件

```typescript
// prompt.ts:318-325
if (
  lastAssistant?.finish &&
  !["tool-calls", "unknown"].includes(lastAssistant.finish) &&
  lastUser.id < lastAssistant.id
) {
  break // 助手已完成且不是工具调用 → 退出循环
}
```

| 退出条件       | finish 值                    | 含义                         |
| -------------- | ---------------------------- | ---------------------------- |
| 模型自然结束   | `"stop"`                     | 模型认为任务完成             |
| 工具调用待处理 | `"tool-calls"`               | 不退出，继续循环处理工具结果 |
| 未知原因       | `"unknown"`                  | 不退出，继续循环             |
| 权限阻断       | processor 返回 `"stop"`      | 用户拒绝或工具被 deny        |
| 结构化输出     | structuredOutput ≠ undefined | 捕获到结构化输出，直接退出   |

### 2.4 关键特性

- **严格串行**：每次循环迭代只处理一个 subtask 或一次 LLM 调用
- **步骤计数**：`step++` 每轮递增，用于步骤限制检查
- **消息重读**：每轮都重新读取所有消息（`MessageV2.stream()`），保证状态最新
- **Abort 支持**：每轮检查 `abort.aborted`，支持用户取消

---

## 3. 并行执行机制：AI SDK streamText()

**源码位置**：`src/session/llm.ts:46-255`

### 3.1 调用链

```
loop() → processor.process() → LLM.stream() → streamText()  [Vercel AI SDK]
```

### 3.2 streamText() 配置

```typescript
// llm.ts:172-255
return streamText({
  tools, // 经权限过滤后的工具集
  toolChoice, // "auto" | "required" | "none"
  activeTools, // 激活的工具列表（排除 "invalid"）
  messages, // 包含 system + 历史消息
  model, // 包装后的语言模型
  temperature, // 温度参数
  maxOutputTokens, // 最大输出 token
  abortSignal, // 取消信号
  maxRetries, // SDK 层重试次数
  experimental_repairToolCall, // 工具调用修复（大小写等）
  experimental_telemetry, // OpenTelemetry 遥测
  onError, // 错误回调
})
```

### 3.3 并行执行原理

Vercel AI SDK 的 `streamText()` 在收到 LLM 返回的多个 tool_calls 时，**自动并行执行所有工具**。这不是 opencode 的代码，而是 AI SDK 的内置行为。

```
LLM 返回响应
    │
    ├── tool_call_1: Task("Fix auth bug")
    ├── tool_call_2: Task("Fix cart bug")
    ├── tool_call_3: bash("npm test")
    │
    ▼
AI SDK 自动并行执行:
    ┌─────────────────┐
    │ Promise.all([   │
    │   execute(1),   │  ← TaskTool → child_session_A.loop()
    │   execute(2),   │  ← TaskTool → child_session_B.loop()
    │   execute(3),   │  ← bash 直接执行
    │ ])              │
    └─────────────────┘
    │
    ▼ 所有工具完成后
    结果写入 assistant message
    loop() 进入下一轮
```

### 3.4 工具解析与权限过滤

```typescript
// llm.ts:258-266
async function resolveTools(input: Pick<StreamInput, "tools" | "agent" | "user">) {
  const disabled = PermissionNext.disabled(Object.keys(input.tools), input.agent.permission)
  for (const tool of Object.keys(input.tools)) {
    if (input.user.tools?.[tool] === false || disabled.has(tool)) {
      delete input.tools[tool] // ← 被 deny 的工具从工具集中移除
    }
  }
  return input.tools
}
```

工具被 `deny` 后，LLM 根本看不到该工具的描述，无法调用。

---

## 4. 流处理器：SessionProcessor

**源码位置**：`src/session/processor.ts:26-421`

### 4.1 创建与状态

```typescript
// processor.ts:26-37
export function create(input: {
  assistantMessage: MessageV2.Assistant
  sessionID: string
  model: Provider.Model
  abort: AbortSignal
}) {
  const toolcalls: Record<string, MessageV2.ToolPart> = {} // 跟踪工具调用
  let snapshot: string | undefined // 文件系统快照
  let blocked = false // 是否被权限阻断
  let attempt = 0 // 重试计数
  let needsCompaction = false // 是否需要压缩
  // ...
}
```

### 4.2 process() 内部循环

```typescript
// processor.ts:45-417
async process(streamInput: LLM.StreamInput) {
  const shouldBreak = (await Config.get()).experimental?.continue_loop_on_deny !== true
  while (true) {
    const stream = await LLM.stream(streamInput)
    for await (const value of stream.fullStream) {
      switch (value.type) {
        case "tool-input-start":   // 工具开始接收输入 → 创建 ToolPart(status: pending)
        case "tool-call":          // 工具调用完成 → running + doom loop 检测
        case "tool-result":        // 工具返回结果 → completed
        case "tool-error":         // 工具出错 → error + blocked 检查
        case "text-delta":         // 文本流式输出
        case "reasoning-delta":    // 推理流式输出
        case "start-step":         // 多步开始 → 快照
        case "finish-step":        // 步骤完成 → token 统计 + 压缩检查
      }
      if (needsCompaction) break  // 压缩需要时提前退出流
    }
    // 流结束后的处理...
  }
}
```

### 4.3 工具调用生命周期

```
tool-input-start  →  创建 ToolPart { status: "pending", input: {}, raw: "" }
        │
tool-input-delta  →  流式接收输入参数（不更新 DB）
        │
tool-input-end    →  输入接收完成
        │
tool-call         →  更新 ToolPart { status: "running", input: parsed }
        │              + Doom Loop 检测
        │
tool-result       →  更新 ToolPart { status: "completed", output, title, metadata }
        │
tool-error        →  更新 ToolPart { status: "error", error }
                      + 如果是 RejectedError/DeniedError → blocked = shouldBreak
```

### 4.4 返回值语义

| 返回值       | 触发条件             | loop() 行为            |
| ------------ | -------------------- | ---------------------- |
| `"continue"` | 正常完成，需要继续   | 继续下一轮循环         |
| `"stop"`     | 被权限阻断 或 有错误 | break 退出循环         |
| `"compact"`  | 检测到上下文溢出     | 创建压缩任务，继续循环 |

---

## 5. 子智能体执行：TaskTool

**源码位置**：`src/tool/task.ts:27-165`

### 5.1 工具签名

```typescript
// task.ts:14-25
const parameters = z.object({
  description: z.string().describe("A short (3-5 words) description of the task"),
  prompt: z.string().describe("The task for the agent to perform"),
  subagent_type: z.string().describe("The type of specialized agent to use for this task"),
  task_id: z.string().describe("Resume a previous task session").optional(),
  command: z.string().describe("The command that triggered this task").optional(),
})
```

### 5.2 执行流程

```
TaskTool.execute(params, ctx)
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ 1. 权限检查                                          │
│    if (!ctx.extra?.bypassAgentCheck) {               │
│      ctx.ask({                                      │
│        permission: "task",                           │
│        patterns: [params.subagent_type],             │
│        always: ["*"],                               │
│      })                                             │
│    }                                                │
└────────────────────┬────────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────────┐
│ 2. 获取/创建 child session                           │
│    if (params.task_id) {                             │
│      恢复已有 session                                │
│    } else {                                         │
│      Session.create({                               │
│        parentID: ctx.sessionID,  ← 绑定父会话       │
│        title: params.description + " (@agent)",      │
│        permission: [                                 │
│          todowrite: deny,  ← 子智能体禁用 todo       │
│          todoread: deny,                             │
│          task: deny (除非有 task 权限), ← 默认禁止嵌套│
│          ...primary_tools: allow,                    │
│        ]                                            │
│      })                                             │
│    }                                                │
└────────────────────┬────────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────────┐
│ 3. 调用 SessionPrompt.prompt()                       │
│    → 进入子会话的 loop()                              │
│    → 子会话独立运行，有自己的步骤计数和 LLM 调用      │
│    → 子会话的 abort 绑定到父会话的 abort              │
└────────────────────┬────────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────────┐
│ 4. 提取结果返回                                      │
│    text = result.parts.findLast(x => x.type === "text")?.text
│    output = [                                        │
│      `task_id: ${session.id}`,                       │
│      "<task_result>",                               │
│      text,                                          │
│      "</task_result>",                              │
│    ].join("\n")                                     │
│    return { title, metadata: { sessionId, model }, output }
└─────────────────────────────────────────────────────┘
```

### 5.3 Child Session 权限隔离

```typescript
// task.ts:72-101
permission: [
  { permission: "todowrite", pattern: "*", action: "deny" },
  { permission: "todoread", pattern: "*", action: "deny" },
  // 如果 agent 没有 task 权限：
  ...(hasTaskPermission ? [] : [{ permission: "task", pattern: "*", action: "deny" }]),
  // primary_tools 只给主代理：
  ...(config.experimental?.primary_tools?.map((t) => ({
    pattern: "*",
    action: "allow" as const,
    permission: t,
  })) ?? []),
]
```

| 权限            | 子智能体默认        | 说明                         |
| --------------- | ------------------- | ---------------------------- |
| `todowrite`     | deny                | 子智能体不管理 todo 列表     |
| `todoread`      | deny                | 子智能体不读取 todo          |
| `task`          | deny (除非显式允许) | 默认禁止子智能体嵌套派发     |
| `primary_tools` | allow               | 主代理独占工具对子智能体开放 |

### 5.4 Abort 传播

```typescript
// task.ts:121-125
function cancel() {
  SessionPrompt.cancel(session.id)
}
ctx.abort.addEventListener("abort", cancel) // 父会话取消 → 子会话也取消
using _ = defer(() => ctx.abort.removeEventListener("abort", cancel))
```

父会话的 abort 信号会传播到子会话，确保取消操作级联生效。

### 5.5 并行子智能体执行时序

```
父会话 loop() — Step N
    │
    ├── LLM 返回: Task(A) + Task(B) [同一消息]
    │
    │   AI SDK 并行执行:
    │   ┌─────────────────────────────────────────────┐
    │   │ Task(A).execute()                           │
    │   │   → child_session_A.loop()                  │
    │   │     while(true) {                           │
    │   │       LLM → tool_calls → 执行 → ...        │
    │   │       break (完成)                          │
    │   │     }                                       │
    │   │                                            │
    │   │ Task(B).execute()  [与 A 同时运行]          │
    │   │   → child_session_B.loop()                  │
    │   │     while(true) {                           │
    │   │       LLM → tool_calls → 执行 → ...        │
    │   │       break (完成)                          │
    │   │     }                                       │
    │   └─────────────────────────────────────────────┘
    │
    │   两个 Task 都完成，结果写入 assistant message
    │   finish = "tool-calls"
    │
    ├── Step N+1: loop() 继续下一轮
    │   检测到 lastAssistant.finish === "tool-calls"
    │   → 不退出，继续循环
    │   → LLM 看到两个 Task 的结果后继续处理
    │
    ├── Step N+2: LLM 可能再发 Task(C) [跨消息，串行]
    │   或 Task(C)+Task(D) [同消息，并行]
    ...
```

---

## 6. 显式并行：Batch 工具

**源码位置**：`src/tool/batch.ts:8-181`

### 6.1 启用条件

需要配置 `config.experimental.batch_tool = true`，否则 Batch 工具不会出现在工具集中。

### 6.2 核心并行逻辑

```typescript
// batch.ts:132
const results = await Promise.all(toolCalls.map((call) => executeCall(call)))
```

### 6.3 约束与限制

| 约束       | 值                       | 说明                     |
| ---------- | ------------------------ | ------------------------ |
| 最大调用数 | 25                       | 超出部分标记为 error     |
| 禁止嵌套   | `DISALLOWED = ["batch"]` | batch 不能调 batch       |
| 仅内置工具 | -                        | MCP 外部工具不能被 batch |
| 部分失败   | 不影响其他               | 每个调用独立跟踪状态     |

### 6.4 执行流程

```
BatchTool.execute({ tool_calls: [call1, call2, call3] })
    │
    ├── 截取前 25 个调用
    │
    ├── Promise.all([
    │     executeCall(call1),  // 独立 try/catch
    │     executeCall(call2),  // 独立 try/catch
    │     executeCall(call3),  // 独立 try/catch
    │   ])
    │
    ├── 每个 executeCall:
    │     1. 查找工具注册表
    │     2. 验证参数
    │     3. 创建 ToolPart { status: "running" }
    │     4. 执行 tool.execute()
    │     5. 更新 ToolPart { status: "completed" / "error" }
    │
    └── 返回 { title, output, metadata: { totalCalls, successful, failed } }
```

---

## 7. Command 子任务路径：SubtaskPart

**源码位置**：`src/session/prompt.ts:1834-1850`

### 7.1 触发条件

当通过斜杠命令（command）触发子智能体时，走的是与 LLM 主动调用 Task 工具不同的路径：

```typescript
// prompt.ts:1834
const isSubtask = (agent.mode === "subagent" && command.subtask !== false) || command.subtask === true
```

### 7.2 两条路径对比

```
路径 A：LLM 主动调用 Task 工具
    LLM 生成 tool_call: Task(...)
    → AI SDK 执行 TaskTool.execute()
    → 可并行（同消息多个 Task）
    → 决策权在 LLM

路径 B：Command 触发 SubtaskPart
    command() 判断 isSubtask = true
    → 创建 SubtaskPart { type: "subtask", agent, prompt, ... }
    → 添加到用户消息的 parts 中
    → loop() 在下一轮检测到 pending subtask
    → 直接调用 TaskTool.execute()（不经过 LLM）
    → 严格串行（loop 每次只弹出一个 pending subtask）
    → 决策权在 command 配置（确定性）
```

### 7.3 SubtaskPart 数据结构

```typescript
// message-v2.ts:204-219
export const SubtaskPart = PartBase.extend({
  type: z.literal("subtask"),
  prompt: z.string(),
  description: z.string(),
  agent: z.string(),
  model: z
    .object({
      providerID: z.string(),
      modelID: z.string(),
    })
    .optional(),
  command: z.string().optional(),
})
```

### 7.4 loop() 中的 Subtask 处理

```typescript
// prompt.ts:352-525
if (task?.type === "subtask") {
  const taskTool = await TaskTool.init()
  // ... 创建 assistant message, ToolPart ...
  const result = await taskTool.execute(taskArgs, taskCtx)
  // ... 更新 ToolPart 状态 ...
  continue // ← 回到循环顶部，处理下一个 pending task
}
```

**关键**：每次循环只弹出一个 subtask（`tasks.pop()`），保证串行处理。

---

## 8. Agent 定义与能力矩阵

**源码位置**：`src/agent/agent.ts`

### 8.1 Agent.Info 数据结构

```typescript
// agent.ts:24-49
export const Info = z.object({
  name: z.string(),
  description: z.string().optional(),
  mode: z.enum(["subagent", "primary", "all"]),
  native: z.boolean().optional(),
  hidden: z.boolean().optional(),
  topP: z.number().optional(),
  temperature: z.number().optional(),
  color: z.string().optional(),
  permission: PermissionNext.Ruleset,
  model: z.object({ modelID: z.string(), providerID: z.string() }).optional(),
  variant: z.string().optional(),
  prompt: z.string().optional(),
  options: z.record(z.string(), z.any()),
  steps: z.number().int().positive().optional(),
})
```

### 8.2 内置 Agent 完整配置

#### build — 默认主代理

| 属性   | 值                                                               |
| ------ | ---------------------------------------------------------------- |
| mode   | `"primary"`                                                      |
| native | `true`                                                           |
| steps  | `undefined` (Infinity)                                           |
| 权限   | defaults + `question: "allow"`, `plan_enter: "allow"` + 用户配置 |

#### plan — 规划模式代理

| 属性   | 值                                                                                       |
| ------ | ---------------------------------------------------------------------------------------- |
| mode   | `"primary"`                                                                              |
| native | `true`                                                                                   |
| steps  | `undefined` (Infinity)                                                                   |
| 权限   | defaults + `question: "allow"`, `plan_exit: "allow"`, `edit: { "*": "deny" }` + 用户配置 |
| 限制   | 只能编辑 `.opencode/plans/*.md` 和 `plans/*.md` 文件                                     |

#### general — 通用子智能体

| 属性   | 值                                                            |
| ------ | ------------------------------------------------------------- |
| mode   | `"subagent"`                                                  |
| native | `true`                                                        |
| steps  | `undefined` (Infinity)                                        |
| 权限   | defaults + `todoread: "deny"`, `todowrite: "deny"` + 用户配置 |

#### explore — 代码探索子智能体

| 属性   | 值                                                                                                                 |
| ------ | ------------------------------------------------------------------------------------------------------------------ |
| mode   | `"subagent"`                                                                                                       |
| native | `true`                                                                                                             |
| prompt | `PROMPT_EXPLORE` (来自 `./prompt/explore.txt`)                                                                     |
| 权限   | defaults + `"*": "deny"` + 显式允许: `grep`, `glob`, `list`, `bash`, `webfetch`, `websearch`, `codesearch`, `read` |

#### compaction — 上下文压缩代理（隐藏）

| 属性   | 值                       |
| ------ | ------------------------ |
| mode   | `"primary"`              |
| native | `true`                   |
| hidden | `true`                   |
| 权限   | defaults + `"*": "deny"` |

#### title — 标题生成代理（隐藏）

| 属性        | 值                       |
| ----------- | ------------------------ |
| mode        | `"primary"`              |
| native      | `true`                   |
| hidden      | `true`                   |
| temperature | `0.5`                    |
| 权限        | defaults + `"*": "deny"` |

#### summary — 摘要生成代理（隐藏）

| 属性   | 值                       |
| ------ | ------------------------ |
| mode   | `"primary"`              |
| native | `true`                   |
| hidden | `true`                   |
| 权限   | defaults + `"*": "deny"` |

### 8.3 默认权限基线

所有 agent 继承的 `defaults` 规则集：

```typescript
{
  "*": "allow",                    // 所有工具默认允许
  doom_loop: "ask",                // doom loop 检测需用户确认
  external_directory: {
    "*": "ask",                    // 外部目录需确认
    ...whitelistedDirs: "allow",   // skill 目录 + truncate glob 允许
  },
  question: "deny",                // question 工具默认禁用
  plan_enter: "deny",              // plan_enter 默认禁用
  plan_exit: "deny",               // plan_exit 默认禁用
  read: {
    "*": "allow",                  // 读取默认允许
    "*.env": "ask",                // .env 文件需确认
    "*.env.*": "ask",              // .env.* 文件需确认
    "*.env.example": "allow",      // .env.example 允许
  },
}
```

### 8.4 自定义 Agent 加载

```typescript
// agent.ts:205-232
for (const [key, value] of Object.entries(cfg.agent ?? {})) {
  if (value.disable) {
    delete result[key]
    continue
  } // 可禁用内置 agent
  let item = result[key]
  if (!item)
    item = result[key] = {
      name: key,
      mode: "all", // 自定义默认为 "all"
      permission: PermissionNext.merge(defaults, user),
      options: {},
      native: false,
    }
  // 逐字段覆盖...
  item.mode = value.mode ?? item.mode
  item.steps = value.steps ?? item.steps
  item.permission = PermissionNext.merge(item.permission, PermissionNext.fromConfig(value.permission ?? {}))
}
```

**关键行为**：

- 自定义 agent 默认 `mode: "all"`（既可做主代理也可做子智能体）
- 权限是**合并**（additive），不是替换
- `disable` 可以移除任何内置 agent

### 8.5 能力矩阵总览

| Agent      | 文件读写      | Bash | Grep/Glob | WebFetch | Todo | Task   | 典型用途   |
| ---------- | ------------- | ---- | --------- | -------- | ---- | ------ | ---------- |
| build      | 读+写         | ✅   | ✅        | ✅       | ✅   | ✅     | 默认主代理 |
| plan       | 只读+计划文件 | ❌   | ✅        | ✅       | ✅   | ✅     | 规划模式   |
| general    | 读+写         | ✅   | ✅        | ✅       | ❌   | 视配置 | 多步骤实现 |
| explore    | 只读          | ✅   | ✅        | ✅       | ❌   | ❌     | 代码探索   |
| compaction | ❌            | ❌   | ❌        | ❌       | ❌   | ❌     | 上下文压缩 |
| title      | ❌            | ❌   | ❌        | ❌       | ❌   | ❌     | 标题生成   |
| summary    | ❌            | ❌   | ❌        | ❌       | ❌   | ❌     | 摘要生成   |

---

## 9. 权限系统

**源码位置**：`src/permission/next.ts`

### 9.1 核心数据结构

```typescript
// next.ts:25-44
export const Action = z.enum(["allow", "deny", "ask"])
export const Rule = z.object({
  permission: z.string(), // 工具名或通配符
  pattern: z.string(), // 文件模式或 "*"
  action: Action, // "allow" | "deny" | "ask"
})
export const Ruleset = Rule.array()
```

### 9.2 规则求值：最后匹配胜出

```typescript
// next.ts:236-243
export function evaluate(permission: string, pattern: string, ...rulesets: Ruleset[]): Rule {
  const merged = merge(...rulesets) // 简单拼接所有规则
  const match = merged.findLast(
    // ← 最后匹配的规则胜出
    (rule) => Wildcard.match(permission, rule.permission) && Wildcard.match(pattern, rule.pattern),
  )
  return match ?? { action: "ask", permission, pattern: "*" } // 无匹配默认 ask
}
```

**求值顺序**：`defaults` → `agent 特定覆盖` → `用户配置` → `session 权限`

后配置的规则覆盖先配置的，因为 `findLast` 取最后一个匹配。

### 9.3 权限请求流程

```
ctx.ask({ permission, patterns, ... })
    │
    ▼
对每个 pattern:
    │
    ├── evaluate() = "deny"  → throw DeniedError (立即拒绝，不问用户)
    ├── evaluate() = "ask"   → 发布 Event.Asked → 等待用户响应
    │                           ├── "reject"  → throw RejectedError
    │                           ├── "once"    → 本次允许
    │                           └── "always"  → 添加规则到 approved，自动批准后续匹配
    └── evaluate() = "allow" → 继续
```

### 9.4 用户响应的级联效果

- **reject**：拒绝当前请求，**同时拒绝同一 session 的所有其他 pending 请求**
- **always**：添加批准规则，**自动检查并批准同一 session 中新匹配的 pending 请求**

### 9.5 工具过滤

```typescript
// next.ts:245-257
export function disabled(tools: string[], ruleset: Ruleset): Set<string> {
  // 对每个工具，如果存在 blanket deny 规则 (pattern: "*", action: "deny")
  // 则从工具集中移除，LLM 看不到该工具
}
```

编辑工具映射：`edit`, `write`, `patch`, `multiedit` → 统一映射到 `"edit"` 权限

### 9.6 错误类型与行为

| 错误类型         | 触发条件           | 对 loop 的影响                    |
| ---------------- | ------------------ | --------------------------------- |
| `DeniedError`    | 配置规则自动拒绝   | 阻断执行                          |
| `RejectedError`  | 用户拒绝（无消息） | 根据 `continue_loop_on_deny` 决定 |
| `CorrectedError` | 用户拒绝（带反馈） | 根据 `continue_loop_on_deny` 决定 |

```typescript
// processor.ts:48
const shouldBreak = (await Config.get()).experimental?.continue_loop_on_deny !== true
// processor.ts:220-225
if (value.error instanceof PermissionNext.RejectedError || value.error instanceof Question.RejectedError) {
  blocked = shouldBreak // true = 阻断, false = 继续
}
```

---

## 10. 步骤限制与 Doom Loop 检测

### 10.1 步骤限制

```typescript
// prompt.ts:558-559
const agent = await Agent.get(lastUser.agent)
const maxSteps = agent.steps ?? Infinity
const isLastStep = step >= maxSteps
```

当达到最大步骤时，注入 `MAX_STEPS` 提示：

```typescript
// prompt.ts:665-672
...(isLastStep ? [{ role: "assistant" as const, content: MAX_STEPS }] : [])
```

这迫使模型在下一步只产生文本响应，不再调用工具。

### 10.2 Doom Loop 检测

```typescript
// processor.ts:20
const DOOM_LOOP_THRESHOLD = 3

// processor.ts:152-176
const parts = await MessageV2.parts(input.assistantMessage.id)
const lastThree = parts.slice(-DOOM_LOOP_THRESHOLD)

if (
  lastThree.length === DOOM_LOOP_THRESHOLD &&
  lastThree.every(
    (p) =>
      p.type === "tool" &&
      p.tool === value.toolName &&
      p.state.status !== "pending" &&
      JSON.stringify(p.state.input) === JSON.stringify(value.input),
  )
) {
  // 连续 3 次相同工具 + 相同输入 → 触发 doom_loop 权限请求
  await PermissionNext.ask({
    permission: "doom_loop",
    patterns: [value.toolName],
    // ...
  })
}
```

**检测条件**：最近 3 个 part 都是同一工具的调用，且输入参数完全相同。

---

## 11. 重试机制

**源码位置**：`src/session/retry.ts`

### 11.1 常量

```typescript
// retry.ts:6-9
export const RETRY_INITIAL_DELAY = 2000 // 初始延迟 2 秒
export const RETRY_BACKOFF_FACTOR = 2 // 每次翻倍
export const RETRY_MAX_DELAY_NO_HEADERS = 30_000 // 无 header 时最大 30 秒
export const RETRY_MAX_DELAY = 2_147_483_647 // 有 header 时无上限
```

### 11.2 指数退避序列

无 header 时：`2s → 4s → 8s → 16s → 30s → 30s → 30s → ...`

有 header 时：

1. 优先使用 `retry-after-ms` header（Anthropic 风格）
2. 其次使用 `retry-after` header（标准 HTTP）
3. 无 retry-after 时：指数退避，无上限

### 11.3 可重试错误

```typescript
// retry.ts:61-100
export function retryable(error) {
  if (MessageV2.ContextOverflowError.isInstance(error)) return undefined // 永不重试

  if (MessageV2.APIError.isInstance(error)) {
    if (!error.data.isRetryable) return undefined // API 标记不可重试
    // 特殊处理：FreeUsageLimitError、Overloaded 等
    return error.data.message // 返回状态消息
  }
  // 通用错误解析：too_many_requests, exhausted, unavailable, rate_limit
}
```

### 11.4 重试在 loop 中的位置

```typescript
// processor.ts:350-377
} catch (e: any) {
  const error = MessageV2.fromError(e, { providerID: input.model.providerID })
  const retry = SessionRetry.retryable(error)
  if (retry !== undefined) {
    attempt++
    const delay = SessionRetry.delay(attempt, ...)
    SessionStatus.set(input.sessionID, { type: "retry", attempt, message: retry, next: Date.now() + delay })
    await SessionRetry.sleep(delay, input.abort).catch(() => {})
    continue    // ← 重试同一次 LLM 调用
  }
  // 不可重试：设置错误，停止
  input.assistantMessage.error = error
  SessionStatus.set(input.sessionID, { type: "idle" })
}
```

**关键**：重试发生在 `processor.process()` 内部，重试的是**同一次 LLM 流式调用**。外层 `loop()` 不感知重试。

---

## 12. 上下文压缩（Compaction）

**源码位置**：`src/session/compaction.ts`

### 12.1 溢出检测

```typescript
// compaction.ts:32-48
export async function isOverflow(input: { tokens; model }) {
  if (config.compaction?.auto === false) return false // 可禁用
  const context = input.model.limit.context
  if (context === 0) return false // 未知上下文大小

  const count = tokens.total || tokens.input + tokens.output + tokens.cache.read + tokens.cache.write
  const reserved = config.compaction?.reserved ?? Math.min(20_000, ProviderTransform.maxOutputTokens(input.model))
  const usable = input.model.limit.input
    ? input.model.limit.input - reserved
    : context - ProviderTransform.maxOutputTokens(input.model)
  return count >= usable
}
```

当 token 使用量 ≥ 可用上下文窗口时触发压缩。`reserved` 缓冲区（默认 20K 或最大输出 token）确保模型有足够空间生成响应。

### 12.2 三个触发点

| 触发点                         | 源码位置               | 条件                                               |
| ------------------------------ | ---------------------- | -------------------------------------------------- |
| loop 检测到 pending compaction | `prompt.ts:529-539`    | 用户消息中有 compaction part                       |
| loop 检测到上下文溢出          | `prompt.ts:542-554`    | `lastFinished` 且 `isOverflow()`                   |
| processor 流中检测到溢出       | `processor.ts:282-284` | `finish-step` 时 `isOverflow()` → 返回 `"compact"` |

### 12.3 压缩流程

```
SessionCompaction.process()
    │
    ├── 1. 创建 assistant message (mode: "compaction", summary: true)
    ├── 2. 使用 compaction agent (所有工具禁用)
    ├── 3. 构建压缩 prompt (Goal, Instructions, Discoveries, Accomplished, Relevant files)
    ├── 4. 发送完整消息历史 + 压缩 prompt 到模型
    ├── 5. 模型生成结构化摘要
    └── 6. 返回 "continue" 或 "stop"
```

### 12.4 旧工具输出修剪（Pruning）

```typescript
// compaction.ts 常量
export const PRUNE_MINIMUM = 20_000 // 只修剪 >20K token 可释放时
export const PRUNE_PROTECT = 40_000 // 保护最近 40K token 的工具输出
const PRUNE_PROTECTED_TOOLS = ["skill"] // skill 工具输出永不修剪
```

算法：

1. 从最新消息向最旧消息反向遍历
2. 跳过最近 2 轮（保留近期上下文）
3. 在任何已有压缩摘要处停止
4. 累计已完成工具输出的 token 估计
5. 累计 > `PRUNE_PROTECT` (40K) 后，标记更旧的输出为待修剪
6. 仅当可修剪总量 > `PRUNE_MINIMUM` (20K) 时才执行
7. 修剪设置 `part.state.time.compacted = Date.now()`（标记而非删除）

---

## 13. 串行 vs 并行决策模型

### 13.1 决策流程

```
用户请求到达
    │
    ▼
是否有多个可识别的子任务？
    │
    ├── 否 → 主代理直接执行（不使用 Task）
    │
    └── 是 → 子任务之间是否独立？
              │
              ├── 是 → 是否操作同一文件/共享资源？
              │         │
              │         ├── 否 → ✅ 并行派发
              │         │         同一消息中发出多个 Task
              │         │
              │         └── 是 → ⚠️ 串行派发
              │                   跨消息逐个 Task
              │
              └── 否 → 后续任务是否需要前序结果？
                        │
                        ├── 是 → 串行派发
                        │         等前序 Task 完成后再发
                        │
                        └── 否 → 部分并行
                                  独立组并行，依赖组串行
```

### 13.2 决策因素矩阵

| 因素           | 倾向并行               | 倾向串行                 |
| -------------- | ---------------------- | ------------------------ |
| **文件作用域** | 各任务操作不同文件     | 任务修改同一文件         |
| **数据依赖**   | 无数据流依赖           | B 需要 A 的输出          |
| **副作用**     | 无共享副作用           | 共享数据库、文件系统状态 |
| **错误传播**   | 一个失败不影响其他     | 一个失败导致后续不可行   |
| **用户意图**   | "同时处理"、"一起修复" | "先做A，再做B"           |
| **任务数量**   | ≥3 个独立任务          | ≤2 个任务或强关联任务    |
| **审查需求**   | 不需要逐个审查         | 每个任务需要质量门控     |

### 13.3 执行模式分类

| 模式          | 触发条件               | Task 分布             | 典型场景                      |
| ------------- | ---------------------- | --------------------- | ----------------------------- |
| **纯并行**    | 多个独立任务           | 同一消息，多个 Task   | 3+ 个文件各自有 bug           |
| **纯串行**    | 任务间有依赖           | 跨消息，逐个 Task     | 重构 → 更新调用方             |
| **阶段并行**  | 阶段内独立，阶段间依赖 | 每阶段一条消息        | 阶段1并行修复 → 阶段2集成验证 |
| **混合模式**  | 部分独立，部分依赖     | 按依赖分组            | 独立bug修复 + 依赖修复的测试  |
| **串行+审查** | 每个任务需质量门控     | 串行 Task + 审查 Task | Subagent-Driven Development   |

### 13.4 LLM 决策引导

系统 prompt 中的关键引导（`src/tool/task.txt:19`）：

> "Launch multiple agents concurrently whenever possible, to maximize performance; to do that, use a single message with multiple tool uses"

Plan 模式 prompt（`src/session/prompt/plan-reminder-anthropic.txt`）：

> "Launch up to 3 explore agents IN PARALLEL"

这些 prompt 引导 LLM 在合适场景下选择并行模式。

---

## 14. 完整执行流程图

```
用户发送消息
       │
       ▼
SessionPrompt.prompt()
       │
       ├── createUserMessage() — 创建用户消息 + 处理文件/agent parts
       │
       ▼
SessionPrompt.loop()
       │
       │ while(true) {
       │
       ├── 读取消息 → 检查退出条件
       │
       ├── [pending subtask?]
       │     └── TaskTool.execute() → child session loop() → continue
       │
       ├── [pending compaction?]
       │     └── SessionCompaction.process() → continue
       │
       ├── [context overflow?]
       │     └── SessionCompaction.create() → continue
       │
       ├── [正常处理]
       │     │
       │     ├── resolveTools() — 权限过滤
       │     │
       │     ├── SessionProcessor.create()
       │     │
       │     └── processor.process()
       │           │
       │           ├── LLM.stream() → streamText()
       │           │     │
       │           │     └── LLM 生成响应
       │           │           │
       │           │           ├── 纯文本 → 流式输出
       │           │           │
       │           │           ├── 单个 tool_call → 串行执行
       │           │           │
       │           │           └── 多个 tool_calls → AI SDK 自动并行执行
       │           │                 │
       │           │                 ├── 普通工具 (bash, read, edit...)
       │           │                 │     → 直接执行
       │           │                 │
       │           │                 ├── TaskTool
       │           │                 │     → 创建 child session
       │           │                 │     → child.loop() 独立运行
       │           │                 │
       │           │                 └── BatchTool
       │           │                       → Promise.all() 显式并行
       │           │
       │           ├── Doom Loop 检测 (3x 相同调用)
       │           │
       │           ├── 重试逻辑 (可重试错误 → 指数退避 → 重试)
       │           │
       │           └── 返回 "continue" | "stop" | "compact"
       │
       ├── [result = "stop"] → break
       ├── [result = "compact"] → 创建压缩 → continue
       └── [result = "continue"] → continue
       │
       │ } // end while
       │
       ├── SessionCompaction.prune() — 修剪旧工具输出
       │
       └── 返回最终消息
```

---

## 15. 关键代码路径索引

| 功能                   | 文件                        | 行号      |
| ---------------------- | --------------------------- | --------- |
| 主循环 loop()          | `src/session/prompt.ts`     | 274-713   |
| prompt 入口            | `src/session/prompt.ts`     | 158-185   |
| Subtask 串行处理       | `src/session/prompt.ts`     | 352-525   |
| Compaction 触发        | `src/session/prompt.ts`     | 529-554   |
| 正常处理分支           | `src/session/prompt.ts`     | 556-712   |
| 退出循环判断           | `src/session/prompt.ts`     | 318-325   |
| 步骤限制               | `src/session/prompt.ts`     | 558-559   |
| Command → Subtask 路径 | `src/session/prompt.ts`     | 1834-1850 |
| Processor 创建         | `src/session/processor.ts`  | 26-37     |
| Processor 流处理       | `src/session/processor.ts`  | 45-417    |
| Doom Loop 检测         | `src/session/processor.ts`  | 152-176   |
| 工具调用生命周期       | `src/session/processor.ts`  | 111-229   |
| 重试逻辑               | `src/session/processor.ts`  | 350-377   |
| LLM streamText 调用    | `src/session/llm.ts`        | 172-255   |
| 工具权限过滤           | `src/session/llm.ts`        | 258-266   |
| TaskTool 定义          | `src/tool/task.ts`          | 27-165    |
| Child session 创建     | `src/tool/task.ts`          | 66-101    |
| Abort 传播             | `src/tool/task.ts`          | 121-125   |
| BatchTool 并行执行     | `src/tool/batch.ts`         | 132       |
| BatchTool 约束         | `src/tool/batch.ts`         | 5-6       |
| Agent.Info 定义        | `src/agent/agent.ts`        | 24-49     |
| 内置 Agent 配置        | `src/agent/agent.ts`        | 76-203    |
| 自定义 Agent 加载      | `src/agent/agent.ts`        | 205-232   |
| 默认权限基线           | `src/agent/agent.ts`        | 56-73     |
| Permission.evaluate    | `src/permission/next.ts`    | 236-243   |
| Permission.ask         | `src/permission/next.ts`    | 131-161   |
| Permission.merge       | `src/permission/next.ts`    | 64-66     |
| Permission.disabled    | `src/permission/next.ts`    | 245-257   |
| Session.create         | `src/session/index.ts`      | 212-228   |
| Session parentID       | `src/session/index.ts`      | 122       |
| 重试延迟计算           | `src/session/retry.ts`      | 28-59     |
| 可重试错误判断         | `src/session/retry.ts`      | 61-100    |
| 压缩溢出检测           | `src/session/compaction.ts` | 32-48     |
| 压缩修剪               | `src/session/compaction.ts` | 50-99     |
| SubtaskPart 定义       | `src/session/message-v2.ts` | 204-219   |
| Agent 配置 schema      | `src/config/config.ts`      | 665-752   |
| 实验性特性             | `src/config/config.ts`      | 1148-1168 |
| 并行提示 (task.txt)    | `src/tool/task.txt`         | 19        |

---

## 附录 A：配置参考

### Agent 配置 schema

```typescript
{
  model: "provider/model",           // 可选，覆盖模型
  variant: "string",                 // 可选，模型变体
  temperature: 0.7,                  // 可选，温度
  top_p: 0.9,                        // 可选，top_p
  prompt: "string",                  // 可选，系统 prompt
  disable: true,                     // 可选，禁用 agent
  description: "string",             // 可选，描述
  mode: "subagent" | "primary" | "all",  // 可选，模式
  hidden: true,                      // 可选，隐藏
  steps: 20,                         // 可选，最大步骤数
  permission: {                      // 可选，权限
    read: "allow",
    edit: { "*.env": "ask", "*": "allow" },
    bash: "ask",
    task: { "*": "deny", "explore": "allow" },
  },
}
```

### 实验性特性

```typescript
{
  experimental: {
    batch_tool: true,                // 启用 Batch 工具
    continue_loop_on_deny: true,     // 权限拒绝后继续循环
    openTelemetry: true,             // 启用 OpenTelemetry
    primary_tools: ["tool1"],        // 主代理独占工具
    mcp_timeout: 30000,              // MCP 请求超时(ms)
  },
  compaction: {
    auto: true,                      // 自动压缩(默认 true)
    prune: true,                     // 自动修剪(默认 true)
    reserved: 20000,                 // 压缩预留 token
  },
}
```

### 配置优先级（从低到高）

1. Remote `.well-known/opencode` (组织默认)
2. Global `~/.config/opencode/opencode.json`
3. Custom `OPENCODE_CONFIG` 环境变量
4. Project `opencode.json`
5. `.opencode` 目录 (agents, commands, plugins)
6. Inline `OPENCODE_CONFIG_CONTENT` 环境变量
7. Managed config (企业级，最高优先级)

---

## 附录 B：术语表

| 术语               | 定义                                              |
| ------------------ | ------------------------------------------------- |
| **Primary Agent**  | 主代理，用户直接交互的代理（build/plan）          |
| **Subagent**       | 子智能体，由主代理通过 Task 工具调用的辅助代理    |
| **Task**           | OpenCode 内置工具，用于派发子智能体               |
| **Child Session**  | Task 创建的独立会话，拥有独立上下文和 loop()      |
| **Parent Session** | 主会话，通过 parentID 关联子会话                  |
| **SubtaskPart**    | Command 触发的确定性子任务，在 loop 中串行处理    |
| **Batch**          | 显式并行工具，通过 Promise.all 执行多个工具调用   |
| **Doom Loop**      | 连续 3 次相同工具+相同输入的循环，需用户确认      |
| **Compaction**     | 上下文压缩，当 token 超限时自动触发               |
| **Pruning**        | 旧工具输出修剪，释放 token 空间                   |
| **Ruleset**        | 权限规则集，由 Rule 数组组成                      |
| **finish**         | LLM 响应的结束原因："stop"/"tool-calls"/"unknown" |
| **step**           | loop() 的迭代计数，受 agent.steps 限制            |
