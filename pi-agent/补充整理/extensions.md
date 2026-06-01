# Pi 扩展提供的各种能力详解

扩展是 Pi 的“神经系统”，通过 TypeScript 模块深度嵌入 Pi 的运行生命周期。它几乎可以修改或增强 Pi 的所有行为——从拦截危险的 Shell 命令，到为模型添加全新工具，再到构建复杂的终端交互界面。以下将 Pi 扩展的核心能力分为十类，每类都配有一个具体示例，展示实际代码和效果。

---

## 1. 自定义工具：为模型提供新“手”

扩展可以注册自定义工具，使 LLM 能够执行原本无法完成的操作。这是扩展最基础也最常用的能力。

**场景**：让模型能够查询企业内部的任务追踪系统（例如 Jira）。

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { Type } from "typebox";

export default function (pi: ExtensionAPI) {
  pi.registerTool({
    name: "jira_search",
    label: "Jira Search",
    description: "通过关键词搜索当前项目的 Jira 工单",
    parameters: Type.Object({
      keyword: Type.String({ description: "搜索关键词" }),
    }),
    async execute(toolCallId, params, signal, onUpdate, ctx) {
      // 实际调用 Jira API
      const response = await fetch(`https://jira.company.com/rest/api/2/search?jql=text~"${params.keyword}"`);
      const data = await response.json();
      const issues = data.issues.map((i: any) => `- ${i.key}: ${i.fields.summary}`).join("\n");
      return {
        content: [{ type: "text", text: issues }],
        details: { count: data.issues.length },
      };
    },
  });
}
```

**效果**：用户可以直接说“搜索一下登录页面的相关 bug”，模型便会调用 `jira_search` 工具，返回实际的工单列表。工具本身由扩展提供，完全可控，无需依赖第三方插件。

---

## 2. 事件拦截与安全门控：为操作加上保险

扩展可以监听 Pi 的事件流，并在关键节点阻止或修改操作。最常见的安全用例是拦截危险的 `bash` 命令。

**场景**：防止模型执行 `rm -rf` 或 `sudo` 命令，必须经用户确认。

```typescript
pi.on("tool_call", async (event, ctx) => {
  if (event.toolName === "bash" && event.input.command) {
    if (event.input.command.includes("rm -rf") || event.input.command.startsWith("sudo")) {
      const ok = await ctx.ui.confirm("危险操作", `确定执行: ${event.input.command}？`);
      if (!ok) {
        return { block: true, reason: "用户取消了危险操作" };
      }
    }
  }
});
```

**效果**：每当模型尝试执行高危命令时，终端会弹出确认对话框。用户按 Escape 或选择“取消”即可阻止执行，同时告知模型该操作已被阻止。

---

## 3. 用户交互：与用户对话

扩展可以通过 `ctx.ui` 提供的一系列对话框与用户交互：选择、确认、文本输入、甚至多行编辑器。

**场景**：一个部署命令，先让用户选择目标环境，然后确认执行。

```typescript
pi.registerCommand("deploy", {
  description: "部署应用到指定环境",
  handler: async (args, ctx) => {
    const env = await ctx.ui.select("选择环境:", ["staging", "production"]);
    if (!env) return;

    const confirmed = await ctx.ui.confirm("确认部署", `确定要部署到 ${env} 吗？`);
    if (confirmed) {
      ctx.ui.notify(`正在部署到 ${env}...`, "info");
      // 执行部署逻辑
    }
  },
});
```

**效果**：用户可以输入 `/deploy` 触发交互式向导，终端内出现列表选择器和确认弹窗，完全不需要跳出 Pi 界面。

---

## 4. 自定义 UI 组件：构建完整的终端交互

当简单的对话框不够用时，扩展可以通过 `ctx.ui.custom()` 创建复杂的 TUI 组件，临时接管编辑器区域。

**场景**：一个 mini 任务看板，用户可以用方向键选择任务，按回车查看详情。

```typescript
import { Text, Container } from "@earendil-works/pi-tui";

pi.registerCommand("taskboard", {
  description: "打开任务看板",
  handler: async (args, ctx) => {
    await ctx.ui.custom<string | undefined>((tui, theme, keybindings, done) => {
      const items = ["修复登录页 bug", "重构用户模块", "编写单元测试"];
      let selected = 0;

      const list = new Container(0, 0);
      const render = () => {
        list.clear();
        items.forEach((item, i) => {
          const prefix = i === selected ? "> " : "  ";
          const color = i === selected ? "accent" : "muted";
          list.add(new Text(theme.fg(color, `${prefix}${item}`), 0, 0));
        });
      };
      render();

      list.onKey = (key) => {
        if (key === "up") { selected = Math.max(0, selected - 1); render(); return true; }
        if (key === "down") { selected = Math.min(items.length - 1, selected + 1); render(); return true; }
        if (key === "return") { done(items[selected]); return true; }
        if (key === "escape") { done(undefined); return true; }
        return false;
      };
      return list;
    });
  },
});
```

**效果**：`/taskboard` 打开后，编辑器区域会渲染一个可交互的任务列表，体验接近一个轻量级的终端应用。

---

## 5. 自定义命令：为 Pi 添加新的斜杠指令

扩展可以注册类似 `/mycommand` 的新命令，模型和用户都可以调用它们。

**场景**：一个 `/summarize-session` 命令，让模型总结当前会话的讨论要点。

```typescript
pi.registerCommand("summarize-session", {
  description: "总结当前会话内容",
  handler: async (args, ctx) => {
    const entries = ctx.sessionManager.getEntries();
    const userMessages = entries.filter(e => e.type === "message" && e.message.role === "user");
    // 将用户消息发送给模型进行总结
    pi.sendUserMessage(`请总结我们到目前为止讨论的要点。历史消息：${JSON.stringify(userMessages)}`);
  },
});
```

**效果**：用户可以输入 `/summarize-session`，或者模型也可以在需要时自行调用该命令，来获取对话摘要。

---

## 6. 会话持久化：跨重启保留状态

扩展可以使用 `pi.appendEntry()` 在会话文件中存储自定义数据，下次启动时恢复。

**场景**：一个“已完成任务”追踪器，即使重启 Pi 也能保留已勾选的项目。

```typescript
let completedTasks: string[] = [];

pi.on("session_start", async (_event, ctx) => {
  completedTasks = [];
  for (const entry of ctx.sessionManager.getEntries()) {
    if (entry.type === "custom" && entry.customType === "task-completed") {
      completedTasks = entry.data.items;
    }
  }
});

pi.registerTool({
  name: "complete_task",
  label: "Complete Task",
  description: "标记一个任务为已完成",
  parameters: Type.Object({ task: Type.String() }),
  async execute(toolCallId, params) {
    completedTasks.push(params.task);
    pi.appendEntry("task-completed", { items: [...completedTasks] });
    return { content: [{ type: "text", text: `已标记完成: ${params.task}` }], details: {} };
  },
});
```

**效果**：即使 Pi 关闭后重新打开，之前的已完成任务列表也会从会话文件中恢复，模型可以继续使用这些状态。

---

## 7. 自定义渲染：控制工具调用/结果的显示

默认情况下，工具调用只显示工具名称和输出文本。扩展可以为自己的工具提供精美的渲染效果，例如高亮差异、图标等。

**场景**：一个“文件差异”工具，用绿色和红色高亮增删行。

```typescript
pi.registerTool({
  name: "show_diff",
  label: "Show Diff",
  description: "展示文件差异",
  parameters: Type.Object({ diff: Type.String() }),
  async execute(toolCallId, params) {
    return { content: [{ type: "text", text: params.diff }], details: {} };
  },
  renderResult(result, options, theme, context) {
    const lines = result.content[0].text.split("\n");
    const colored = lines.map(line => {
      if (line.startsWith("+")) return theme.fg("success", line);
      if (line.startsWith("-")) return theme.fg("error", line);
      return theme.fg("muted", line);
    }).join("\n");
    return new Text(colored, 0, 0);
  },
});
```

**效果**：当模型返回差异内容时，终端中会以绿色显示新增行、红色显示删除行，视觉效果清晰。

---

## 8. 远程执行：将工具操作代理到另一台机器

Pi 的内置工具（如 `read`、`bash`）支持可插拔的操作接口。扩展可以通过提供自定义操作对象，将这些工具的执行透明地转发到远程服务器（例如通过 SSH）。

**场景**：所有 `bash` 命令都在远程开发机上执行。

```typescript
import { createBashTool } from "@earendil-works/pi-coding-agent";
import { exec } from "child_process";

const remoteBash = createBashTool(cwd, {
  operations: {
    exec: (command, cwd, options) => {
      return new Promise((resolve) => {
        exec(`ssh dev-server "cd ${cwd} && ${command}"`, (error, stdout, stderr) => {
          resolve({ output: stdout + stderr, exitCode: error ? 1 : 0, cancelled: false, truncated: false });
        });
      });
    },
  },
});
```

**效果**：用户完全感受不到区别，但所有文件读写和命令执行实际上都发生在远程服务器上，Pi 本地环境保持隔离。

---

## 9. 输入转换：重写或拦截用户消息

扩展可以在用户输入被发送给模型之前进行修改，实现自定义的快捷语法。

**场景**：将 `?bug` 开头的消息自动扩展为完整的 bug 报告模板。

```typescript
pi.on("input", async (event, ctx) => {
  if (event.text.startsWith("?bug")) {
    return {
      action: "transform",
      text: `请生成一个 Bug 报告，包含以下部分：\n1. 问题描述\n2. 复现步骤\n3. 期望行为\n4. 实际行为\n5. 环境信息\n\n原始描述：${event.text.slice(4)}`,
    };
  }
});
```

**效果**：用户只需输入 `?bug 登录页面崩溃`，Pi 会自动将其替换为完整的结构化提示，提高了 Bug 报告的规范性。

---

## 10. 模型选择响应与思考级别指示

扩展可以监听模型变化或思考级别变化，并更新 UI 组件，提供清晰的视觉反馈。

**场景**：当模型切换时，在页脚显示当前模型名称；当思考级别为“高”时，显示一个警示图标。

```typescript
pi.on("model_select", async (event, ctx) => {
  ctx.ui.setStatus("current-model", `当前模型: ${event.model.id}`);
});

pi.on("thinking_level_select", async (event, ctx) => {
  if (event.level === "xhigh") {
    ctx.ui.notify("⚠️ 已启用最高思考级别，消耗较大", "warning");
  }
});
```

**效果**：用户能随时在界面底部看到当前使用的模型，并在成本较高时收到提醒，帮助控制 API 开销。

---

## 总结

Pi 扩展的能力体系覆盖了从**底层工具**到**顶层 UI** 的完整栈：

| 能力类别 | 可以做什么 | 典型场景 |
| :--- | :--- | :--- |
| 自定义工具 | 为 LLM 注册新工具 | 集成内部 API、数据库查询 |
| 事件拦截 | 阻止/修改操作 | 安全门控、路径保护 |
| 用户交互 | 对话框、选择器、向导 | 部署确认、环境选择 |
| 自定义 UI | 复杂终端组件 | 任务看板、自定义编辑器 |
| 自定义命令 | 注册斜杠命令 | 摘要生成、工作流触发 |
| 会话持久化 | 跨重启保留数据 | 已完成任务列表、缓存 |
| 自定义渲染 | 富文本显示工具结果 | 差异高亮、语法着色 |
| 远程执行 | 代理到远程环境 | SSH、容器、沙箱 |
| 输入转换 | 扩展快捷语法 | 模板缩写、自定义指令 |
| 模型状态监听 | 响应模型/思考级别变化 | 状态栏更新、成本警示 |

这些能力共同将 Pi 从一个单纯的终端 AI 对话客户端，转变为完全可编程的**AI 工程工作站**。每一个能力都是独立且可组合的，开发者可以根据自身需求精准装配，构建出最适合自己工作流的智能开发环境。