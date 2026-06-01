# Themes（主题）

**版本：最新**  
[查看源码] [在 GitHub 上编辑]

Pi 可以创建主题。你可以要求它为你构建一个适合你环境的主题。

主题是 JSON 文件，用于定义 TUI（终端用户界面）的颜色。

## 目录
- 位置
- 选择主题
- 创建自定义主题
- 主题格式
- 颜色标记
  - 核心 UI（11 种颜色）
  - 背景与内容（11 种颜色）
  - Markdown（10 种颜色）
  - 工具差异（3 种颜色）
  - 语法高亮（9 种颜色）
  - 思考级别边框（6 种颜色）
  - Bash 模式（1 种颜色）
  - HTML 导出（可选）
- 颜色值
- 提示
- 示例

## 位置
Pi 从以下位置加载主题：
- **内置**：`dark`（暗色）、`light`（亮色）
- **全局**：`~/.pi/agent/themes/*.json`
- **项目**：`.pi/themes/*.json`
- **包**：`package.json` 中的 `themes/` 目录或 `pi.themes` 条目
- **设置**：`settings.json` 中的 `themes` 数组，可包含文件或目录
- **CLI**：`--theme <path>`（可重复使用）

使用 `--no-themes` 可禁用自动发现。

## 选择主题
通过 `/settings` 或 `settings.json` 选择主题：

```json
{
  "theme": "my-theme"
}
```

首次运行时，Pi 会检测你的终端背景色，并默认使用暗色或亮色主题。

## 创建自定义主题
创建一个主题文件：
```bash
mkdir -p ~/.pi/agent/themes
vim ~/.pi/agent/themes/my-theme.json
```

定义主题时需包含所有必需的颜色（见颜色标记）：
```json
{
  "$schema": "https://raw.githubusercontent.com/earendil-works/pi/main/packages/coding-agent/src/modes/interactive/theme/theme-schema.json",
  "name": "my-theme",
  "vars": {
    "primary": "#00aaff",
    "secondary": 242
  },
  "colors": {
    "accent": "primary",
    "border": "primary",
    "borderAccent": "#00ffff",
    "borderMuted": "secondary",
    "success": "#00ff00",
    "error": "#ff0000",
    "warning": "#ffff00",
    "muted": "secondary",
    "dim": 240,
    "text": "",
    "thinkingText": "secondary",
    "selectedBg": "#2d2d30",
    "userMessageBg": "#2d2d30",
    "userMessageText": "",
    "customMessageBg": "#2d2d30",
    "customMessageText": "",
    "customMessageLabel": "primary",
    "toolPendingBg": "#1e1e2e",
    "toolSuccessBg": "#1e2e1e",
    "toolErrorBg": "#2e1e1e",
    "toolTitle": "primary",
    "toolOutput": "",
    "mdHeading": "#ffaa00",
    "mdLink": "primary",
    "mdLinkUrl": "secondary",
    "mdCode": "#00ffff",
    "mdCodeBlock": "",
    "mdCodeBlockBorder": "secondary",
    "mdQuote": "secondary",
    "mdQuoteBorder": "secondary",
    "mdHr": "secondary",
    "mdListBullet": "#00ffff",
    "toolDiffAdded": "#00ff00",
    "toolDiffRemoved": "#ff0000",
    "toolDiffContext": "secondary",
    "syntaxComment": "secondary",
    "syntaxKeyword": "primary",
    "syntaxFunction": "#00aaff",
    "syntaxVariable": "#ffaa00",
    "syntaxString": "#00ff00",
    "syntaxNumber": "#ff00ff",
    "syntaxType": "#00aaff",
    "syntaxOperator": "primary",
    "syntaxPunctuation": "secondary",
    "thinkingOff": "secondary",
    "thinkingMinimal": "primary",
    "thinkingLow": "#00aaff",
    "thinkingMedium": "#00ffff",
    "thinkingHigh": "#ff00ff",
    "thinkingXhigh": "#ff0000",
    "bashMode": "#ffaa00"
  }
}
```
通过 `/settings` 选择该主题。
**热重载**：当你编辑当前活动的自定义主题文件时，Pi 会自动重新加载它，让你立即看到视觉变化。

## 主题格式
```json
{
  "$schema": "...",
  "name": "my-theme",
  "vars": {
    "blue": "#0066cc",
    "gray": 242
  },
  "colors": {
    "accent": "blue",
    "muted": "gray",
    "text": "",
    ...
  }
}
```
- `name` 是必填项，且必须唯一。
- `vars` 是可选的，用于定义可重用的颜色，然后在 `colors` 中引用。
- `colors` 必须定义全部 51 个必需的标记（token）。
- `$schema` 字段可启用编辑器的自动补全和验证。

## 颜色标记
每个主题必须定义全部 51 个颜色标记，没有可选颜色。

### 核心 UI（11 种颜色）
| 标记 | 用途 |
| :--- | :--- |
| `accent` | 主强调色（Logo、选中项、光标） |
| `border` | 普通边框 |
| `borderAccent` | 高亮边框 |
| `borderMuted` | 柔和边框（编辑器） |
| `success` | 成功状态 |
| `error` | 错误状态 |
| `warning` | 警告状态 |
| `muted` | 次要文本 |
| `dim` | 三级文本（更暗淡） |
| `text` | 默认文本（通常为 `""` 以使用终端默认色） |
| `thinkingText` | 思考块文本 |

### 背景与内容（11 种颜色）
| 标记 | 用途 |
| :--- | :--- |
| `selectedBg` | 选中行背景 |
| `userMessageBg` | 用户消息背景 |
| `userMessageText` | 用户消息文本 |
| `customMessageBg` | 扩展消息背景 |
| `customMessageText` | 扩展消息文本 |
| `customMessageLabel` | 扩展消息标签 |
| `toolPendingBg` | 工具框（等待中） |
| `toolSuccessBg` | 工具框（成功） |
| `toolErrorBg` | 工具框（失败） |
| `toolTitle` | 工具标题 |
| `toolOutput` | 工具输出文本 |

### Markdown（10 种颜色）
| 标记 | 用途 |
| :--- | :--- |
| `mdHeading` | 标题 |
| `mdLink` | 链接文本 |
| `mdLinkUrl` | 链接 URL |
| `mdCode` | 行内代码 |
| `mdCodeBlock` | 代码块内容 |
| `mdCodeBlockBorder` | 代码块边框 |
| `mdQuote` | 引用文本 |
| `mdQuoteBorder` | 引用边框 |
| `mdHr` | 水平分割线 |
| `mdListBullet` | 列表项目符号 |

### 工具差异（3 种颜色）
| 标记 | 用途 |
| :--- | :--- |
| `toolDiffAdded` | 新增行 |
| `toolDiffRemoved` | 删除行 |
| `toolDiffContext` | 上下文行 |

### 语法高亮（9 种颜色）
| 标记 | 用途 |
| :--- | :--- |
| `syntaxComment` | 注释 |
| `syntaxKeyword` | 关键字 |
| `syntaxFunction` | 函数名 |
| `syntaxVariable` | 变量 |
| `syntaxString` | 字符串 |
| `syntaxNumber` | 数字 |
| `syntaxType` | 类型 |
| `syntaxOperator` | 运算符 |
| `syntaxPunctuation` | 标点符号 |

### 思考级别边框（6 种颜色）
编辑器边框的颜色指示当前的思考级别（从 subtle 到 prominent 的视觉层次）：
| 标记 | 用途 |
| :--- | :--- |
| `thinkingOff` | 思考关闭 |
| `thinkingMinimal` | 极简思考 |
| `thinkingLow` | 低思考 |
| `thinkingMedium` | 中等思考 |
| `thinkingHigh` | 高思考 |
| `thinkingXhigh` | 超高思考 |

### Bash 模式（1 种颜色）
| 标记 | 用途 |
| :--- | :--- |
| `bashMode` | bash 模式下（`!` 前缀）编辑器边框颜色 |

### HTML 导出（可选）
`export` 部分控制 `/export` 输出 HTML 时的颜色。如果省略，将从 `userMessageBg` 派生。
```json
{
  "export": {
    "pageBg": "#18181e",
    "cardBg": "#1e1e24",
    "infoBg": "#3c3728"
  }
}
```

## 颜色值
支持四种格式：
| 格式 | 示例 | 描述 |
| :--- | :--- | :--- |
| Hex | `"#ff0000"` | 6 位十六进制 RGB |
| 256 色 | `39` | xterm 256 色调色板索引（0-255） |
| 变量 | `"primary"` | 引用 `vars` 中定义的条目 |
| 默认 | `""` | 使用终端默认颜色（通常用于 `text`） |

**256 色调色板**：
- 0-15：基本 ANSI 颜色（取决于终端）
- 16-231：6×6×6 RGB 颜色立方体（16 + 36×R + 6×G + B，其中 R,G,B 为 0-5）
- 232-255：灰度渐变

**终端兼容性**：Pi 使用 24 位 RGB 颜色。大多数现代终端都支持（iTerm2、Kitty、WezTerm、Windows Terminal、VS Code）。对于仅支持 256 色的旧终端，Pi 会回退到最接近的近似颜色。

检查 truecolor 支持：
```bash
echo $COLORTERM  # 应输出 "truecolor" 或 "24bit"
```

## 提示
- **暗色终端**：使用明亮、饱和的颜色，并提高对比度。
- **亮色终端**：使用较暗、柔和的颜色，并降低对比度。
- **色彩和谐**：从一个基础调色板开始（如 Nord、Gruvbox、Tokyo Night），在 `vars` 中定义，然后一致地引用。
- **测试**：使用不同的消息类型、工具状态、Markdown 内容以及长文本换行来检查主题。
- **VS Code**：将 `terminal.integrated.minimumContrastRatio` 设置为 `1` 以获得准确的颜色。

## 示例
参见内置主题：
- `dark.json`
- `light.json`

---

## 💡 Pi 主题系统详细解释与示例

Pi 的主题系统绝不仅仅是“换个颜色”，它是**可编程 TUI 视觉层**的核心，允许用户精确控制每一个界面元素的颜色，从而匹配个人喜好、提升可读性，甚至通过颜色传递状态信息（如思考级别）。下面我们深入解析其设计理念与使用方法，并通过一个具体示例演示如何从零创建一个自定义主题。

### 1. 设计哲学：分层变量引用，全局统一
主题文件使用 `vars`（变量）作为中间层，你可以定义一组基础颜色，然后在 `colors` 中通过名字引用。这带来了几个好处：
- **保持一致性**：比如你定义 `"blue": "#0066cc"`，然后在多个地方引用 `"primary": "blue"`，当你想换一种蓝色时，只需修改 `vars` 中的 `blue` 定义即可，所有引用处自动更新。
- **主题变体**：可以轻松创建亮/暗变体，只改变 `vars` 中的基础色板，而不必逐个重写 51 个颜色标记。
- **热重载**：编辑主题文件时，Pi 会实时重载，所见即所得，极大降低了调色成本。

### 2. 颜色标记的作用域
51 个标记覆盖了 Pi 界面的每一个角落，分为：
- **核心 UI**：边框、强调色、状态色（成功/错误/警告），负责界面的骨架和交互反馈。
- **背景与内容**：用户消息、助手消息、工具框的背景和文本色。工具框又细分为等待中、成功、失败三种状态背景，让工具调用的结果一目了然。
- **Markdown 渲染**：助手回复常包含 Markdown，这些标记控制标题、代码块、引用等的颜色，影响阅读体验。
- **语法高亮**：当助手或用户展示代码时，Pi 使用这些标记进行语法着色，支持多种语言。
- **思考级别边框**：编辑器边框的颜色随模型的思考深度变化，从浅到深形成视觉提示，让用户感知当前模型正在进行的推理量。
- **Bash 模式**：当你在编辑器中使用 `!` 执行 shell 命令时，边框颜色会改变，提醒你正处于命令模式。

### 3. 颜色值机制：灵活且兼容
Pi 支持 hex、256 色索引、变量和空字符串（终端默认色）。空字符串特别重要：当你设置 `"text": ""` 时，Pi 使用终端默认前景色，这保证了主题在终端切换配色方案时仍能保持协调。256 色索引让主题也能在较老的终端上显示，而 hex 值则提供完整的真彩色支持，对于现代终端能呈现出精细的设计。

### 4. 完整创建示例：Gruvbox 风格主题
假设你想创建一个基于 Gruvbox 暗色调色板的主题。首先，你需要知道 Gruvbox 的典型颜色值（以 hex 表示）：
- 背景：`#282828`（dark0），前景：`#ebdbb2`（light0）
- 红色：`#cc241d`，绿色：`#98971a`，黄色：`#d79921`，蓝色：`#458588`，紫色：`#b16286`，青色：`#689d6a`，橙色：`#d65d0e`

我们将这些颜色定义为 `vars`，然后映射到 Pi 的颜色标记。注意，终端默认背景和前景已经由终端设置，所以主题中的背景色通常用于消息块等特定区域，而默认文本颜色可使用空字符串以匹配终端前景。但为了主题完整性，我们可以设置特定背景。

创建文件 `~/.pi/agent/themes/gruvbox-dark.json`：

```json
{
  "$schema": "https://raw.githubusercontent.com/earendil-works/pi/main/packages/coding-agent/src/modes/interactive/theme/theme-schema.json",
  "name": "gruvbox-dark",
  "vars": {
    "bg0": "#282828",
    "bg1": "#3c3836",
    "fg0": "#ebdbb2",
    "fg1": "#d5c4a1",
    "red": "#cc241d",
    "green": "#98971a",
    "yellow": "#d79921",
    "blue": "#458588",
    "purple": "#b16286",
    "aqua": "#689d6a",
    "orange": "#d65d0e",
    "gray": "#928374"
  },
  "colors": {
    "accent": "blue",
    "border": "gray",
    "borderAccent": "aqua",
    "borderMuted": "bg1",
    "success": "green",
    "error": "red",
    "warning": "yellow",
    "muted": "gray",
    "dim": "bg1",
    "text": "",
    "thinkingText": "fg1",
    "selectedBg": "#504945",
    "userMessageBg": "bg1",
    "userMessageText": "fg0",
    "customMessageBg": "bg1",
    "customMessageText": "fg1",
    "customMessageLabel": "blue",
    "toolPendingBg": "#1d2021",
    "toolSuccessBg": "#2e3b2e",
    "toolErrorBg": "#3c1e1e",
    "toolTitle": "aqua",
    "toolOutput": "",
    "mdHeading": "yellow",
    "mdLink": "blue",
    "mdLinkUrl": "aqua",
    "mdCode": "green",
    "mdCodeBlock": "",
    "mdCodeBlockBorder": "gray",
    "mdQuote": "fg1",
    "mdQuoteBorder": "gray",
    "mdHr": "gray",
    "mdListBullet": "orange",
    "toolDiffAdded": "green",
    "toolDiffRemoved": "red",
    "toolDiffContext": "gray",
    "syntaxComment": "gray",
    "syntaxKeyword": "purple",
    "syntaxFunction": "blue",
    "syntaxVariable": "orange",
    "syntaxString": "green",
    "syntaxNumber": "purple",
    "syntaxType": "yellow",
    "syntaxOperator": "aqua",
    "syntaxPunctuation": "fg1",
    "thinkingOff": "gray",
    "thinkingMinimal": "blue",
    "thinkingLow": "aqua",
    "thinkingMedium": "green",
    "thinkingHigh": "yellow",
    "thinkingXhigh": "red",
    "bashMode": "orange"
  }
}
```

保存后，在 Pi 中运行 `/settings`，将主题切换为 `gruvbox-dark`，立即生效。你可以看到用户消息块变为浅灰色背景，工具框状态颜色变为 Gruvbox 风格，语法高亮中的关键字变为紫色，等等。

### 5. 热重载与调试
在编辑上述 JSON 文件时（例如微调 `accent` 的颜色），一旦保存，Pi 的 TUI 会立即刷新，无需重启。你可以在一个终端窗格编辑主题，在另一个窗格观察 Pi 的变化，快速找到满意的配色。如果不小心写错了格式，Pi 会回退到内置的 dark 主题，并在消息区显示错误提示。

### 6. 主题系统的扩展性
- **分享主题**：你可以将 `gruvbox-dark.json` 放到项目的 `.pi/themes/` 目录，通过版本控制与团队共享，或者打包成 pi package 发布。
- **根据环境切换**：通过 `/settings` 随时切换，也可以编写扩展监听 `session_start` 事件，根据时间或外部条件动态设置主题（例如使用 `pi.setTheme()` 方法，虽然 API 中未见但概念上可能通过设置实现）。
- **HTML 导出**：主题中的 `export` 部分确保导出的 HTML 报告也保持一致的视觉风格，适合分享对话记录。

### 7. 为什么主题重要？
在多 Agent 交互和复杂编码任务中，终端是我们与 AI 的主要接触面。一个精心调校的主题可以：
- **降低认知负荷**：通过颜色快速区分用户、助手、工具调用的输出，以及成功/失败状态。
- **传递微妙信息**：编辑器边框的“思考级别”颜色，让你在不看文字的情况下感知模型是否正在进行深度推理。
- **提升长时间工作的舒适度**：护眼的暗色主题或高对比度亮色主题能减少视疲劳。
- **表现个性与项目文化**：团队可以拥有统一的“项目主题”，增强归属感。

总之，Pi 的主题系统以其全面的颜色标记、灵活的变量体系和热重载能力，为终端 AI 交互提供了专业级的视觉定制体验，是 “Hackable” 精神的又一体现。