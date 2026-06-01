# Pi 包提供的能力详解与示例

一个 Pi 包是一个可复用的能力集合，可以通过 npm 或 git 分发。它能够同时打包以下四种资源：**扩展**、**技能**、**提示模板**和**主题**。通过一个包，你可以将一整套 AI 辅助开发的最佳实践和自定义工具，像安装普通软件库一样部署到自己的 Pi 环境中。

下面通过一个虚构的示例包 `@myteam/devtoolkit` 来展示 Pi 包能够提供的全部能力。

## 示例包：`@myteam/devtoolkit`

这个包的目标是为团队提供统一的工作流增强：自动 Git 检查点、PDF 分析、标准化的代码审查提示以及一个护眼的暗色主题。

### 包结构

```
devtoolkit/
├── package.json
├── extensions/
│   └── git-checkpoint.ts
├── skills/
│   └── pdf-analyzer/
│       ├── SKILL.md
│       └── scripts/
│           └── analyze.py
├── prompts/
│   └── review.md
└── themes/
    └── eye-care-dark.json
```

### `package.json` 清单

```json
{
  "name": "@myteam/devtoolkit",
  "version": "1.0.0",
  "keywords": ["pi-package"],
  "pi": {
    "extensions": ["./extensions"],
    "skills": ["./skills"],
    "prompts": ["./prompts"],
    "themes": ["./themes"]
  }
}
```

使用 `pi` 字段明确声明了各类资源的路径。安装后，Pi 会自动加载这些资源。

---

## 能力 1：扩展（Extension）—— 自动 Git 检查点

扩展可以深度定制 Pi 的行为，例如添加新工具、拦截事件、修改系统提示等。本例中的扩展 `git-checkpoint.ts` 会在每个用户回合开始前自动执行 `git stash`，将未提交的修改保存到一个临时存储区，从而为 AI 提供一个干净的工作区。当会话结束或切换分支时，再自动恢复。

**`extensions/git-checkpoint.ts`** 内容：

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function (pi: ExtensionAPI) {
  pi.on("turn_start", async (_event, ctx) => {
    // 在每个 AI 回合开始前，自动 stash 当前修改
    await pi.exec("git", ["stash", "push", "-m", "pi-auto-checkpoint"], {});
    ctx.ui.notify("工作区已自动暂存", "info");
  });

  pi.on("turn_end", async (_event, ctx) => {
    // 回合结束时恢复工作区
    await pi.exec("git", ["stash", "pop"], {});
  });
}
```

**提供的能力**：
- **后台自动化**：无需用户干预，每个回合自动保持工作区整洁。
- **事件驱动**：通过 `pi.on("turn_start", ...)` 和 `pi.on("turn_end", ...)` 精确插入逻辑。
- **通知反馈**：使用 `ctx.ui.notify` 在界面底部显示状态消息。

安装此包后，用户启动 Pi 即可自动获得该功能，无需任何配置。

---

## 能力 2：技能（Skill）—— PDF 分析

技能为模型提供按需加载的专业知识和操作指南。本例中的技能 `pdf-analyzer` 允许 Pi 在用户要求处理 PDF 时，自动加载分析脚本的使用说明。

**`skills/pdf-analyzer/SKILL.md`** 内容：

```markdown
---
name: pdf-analyzer
description: 从 PDF 中提取文本、表格和图像。当用户要求读取、解析或分析 PDF 文件时使用。
---

## 设置
首次使用前运行：`pip install -r requirements.txt`

## 提取文本
```bash
python3 skills/pdf-analyzer/scripts/analyze.py --mode text <input.pdf>
```

## 提取表格
```bash
python3 skills/pdf-analyzer/scripts/analyze.py --mode tables <input.pdf> --output tables.csv
```

## 提取图像
```bash
python3 skills/pdf-analyzer/scripts/analyze.py --mode images <input.pdf> --output-dir ./images
```
```

**提供的能力**：
- **渐进式披露**：技能描述始终在系统提示中，但完整指令仅在需要时由模型读取。
- **可执行脚本**：技能可以携带实际的 Python 脚本，模型可以调用它们来完成精确的操作，而不是仅靠生成文本。
- **跨工具兼容**：遵循 Agent Skills 标准，这个技能也可以被 Claude Code 或 Codex 使用。

当用户输入 “帮我分析这个 PDF 里的表格” 时，模型会根据描述匹配到 `pdf-analyzer`，自动读取 `SKILL.md` 并按照里面的命令执行。

---

## 能力 3：提示模板（Prompt Template）—— 代码审查

提示模板将常用的提示词抽象为可带参数的命令，减少重复输入并保证团队规范一致。本例中的模板 `review.md` 提供了一个标准化的代码审查提示。

**`prompts/review.md`** 内容：

```markdown
---
description: 审查已暂存的变更，检查逻辑、安全和错误处理
argument-hint: "[focus area]"
---

请审查当前暂存的更改 (`git diff --cached`)。

重点检查：
- 逻辑错误和潜在 bug
- 安全漏洞（如注入、不安全依赖）
- 错误处理缺失
- 代码风格与项目一致性
${@:1}

如果发现严重问题，请提供修复建议。
```

**提供的能力**：
- **快速调用**：用户输入 `/review` 即可展开完整提示词，无需每次都手打审查清单。
- **参数化**：`/review "并发安全"` 会将 “并发安全” 作为额外关注点注入到提示中。
- **自动补全**：模板的描述和参数提示会出现在编辑器的自动补全菜单中。

这使团队中的所有成员都能使用相同的审查标准，减少人为差异。

---

## 能力 4：主题（Theme）—— 护眼暗色

主题定义终端的全部颜色，可以完全改变 Pi 的外观。本例中的主题 `eye-care-dark` 采用低对比度、暖色调的配色，适合长时间编码。

**`themes/eye-care-dark.json`** 内容（节选）：

```json
{
  "name": "eye-care-dark",
  "vars": {
    "bg": "#1e1e1e",
    "fg": "#d4d4d4",
    "warm-yellow": "#dcdcaa",
    "soft-blue": "#569cd6",
    "soft-green": "#6a9955"
  },
  "colors": {
    "accent": "soft-blue",
    "text": "fg",
    "userMessageBg": "#2a2a2a",
    "toolSuccessBg": "#1e2e1e",
    "toolTitle": "warm-yellow",
    "mdCode": "soft-green",
    "syntaxKeyword": "soft-blue",
    "thinkingXhigh": "#f48771"
  }
}
```

（注：实际文件需包含全部 51 个颜色标记，此处仅展示部分以说明概念）

**提供的能力**：
- **视觉个性化**：一键切换整个终端的配色方案。
- **热重载**：编辑主题文件后 Pi 自动刷新，无需重启。
- **分享**：可以随包一起发布，团队成员安装后即可在 `/settings` 中选用该主题。

用户可以通过 `/settings` 选择 `eye-care-dark` 主题，立即获得一致的视觉体验。

---

## 安装与使用

要安装这个示例包，执行：

```bash
pi install npm:@myteam/devtoolkit
```

或者从 Git 安装：

```bash
pi install git:github.com/myteam/devtoolkit@v1.0.0
```

安装后，重启 Pi 或执行 `/reload`，以上全部能力将自动生效：

- `git-checkpoint` 扩展在后台运行，自动管理 Git 工作区。
- `pdf-analyzer` 技能出现在可用技能列表中，按需触发。
- `/review` 提示模板出现在命令补全中，可随时调用。
- `eye-care-dark` 主题在 `/settings` 中可选。

如果需要只在当前项目启用，可以使用 `-l` 标志安装到项目本地：

```bash
pi install -l npm:@myteam/devtoolkit
```

## 总结

一个 Pi 包是四种能力的集合体：

| 能力类型 | 作用 | 示例 |
| :--- | :--- | :--- |
| **扩展** | 修改 Pi 运行逻辑，添加后台行为 | 自动 Git stash、权限门控、自定义工具 |
| **技能** | 提供可复用的专业领域知识和操作指南 | PDF 处理、部署脚本、数据库迁移 |
| **提示模板** | 快速生成标准化的提示词 | 代码审查、变更日志、组件生成 |
| **主题** | 定制终端视觉风格 | 暗色/亮色主题、品牌配色 |

通过将这些能力打包成一个可安装的单元，Pi 包系统让团队能够像管理代码库一样管理 AI 辅助开发的工作流，实现真正的“AI 工程化”。