# Pi Packages（Pi 包）

**版本：最新**  
[查看源码] [在 GitHub 上编辑]

Pi 可以帮助你创建 Pi 包。你可以要求它将你的扩展、技能、提示模板或主题打包起来。

Pi 包将扩展、技能、提示模板和主题打包在一起，以便你通过 npm 或 git 分享它们。一个包可以在 `package.json` 的 `pi` 键下声明资源，或者使用约定的目录结构。

## 目录

- 安装与管理
- 包来源
- 创建一个 Pi 包
- 包结构
- 依赖
- 包过滤
- 启用和禁用资源
- 作用域和去重

## 安装与管理

**安全提示**：Pi 包以完整的系统访问权限运行。扩展会执行任意代码，技能可以指示模型执行任何操作，包括运行可执行文件。在安装第三方包之前，请审查源代码。

```bash
pi install npm:@foo/bar@1.0.0
pi install git:github.com/user/repo@v1
pi install https://github.com/user/repo  # 原始 URL 同样有效
pi install /absolute/path/to/package
pi install ./relative/path/to/package

pi remove npm:@foo/bar
pi list                     # 显示已安装的包（来自设置）
pi update                   # 更新 pi、更新包，并协调固定的 git 引用
pi update --extensions      # 仅更新包并协调固定的 git 引用
pi update --self            # 仅更新 pi
pi update --self --force    # 即使已是最新也重装 pi
pi update npm:@foo/bar      # 更新单个包
pi update --extension npm:@foo/bar
```

这些命令管理的是 pi 包，而不是 pi CLI 本身的安装。要卸载 pi 本身，请参阅 **Quickstart**。

默认情况下，`install` 和 `remove` 会写入用户设置 (`~/.pi/agent/settings.json`)。使用 `-l` 标志可以改为写入项目设置 (`.pi/settings.json`)。项目设置可以与你的团队共享，Pi 会在启动时自动安装任何缺失的包。

如果想试用一个包而不正式安装它，请使用 `--extension` 或 `-e` 标志。这会将包安装到一个临时目录，仅在当前运行时有效：

```bash
pi -e npm:@foo/bar
pi -e git:github.com/user/repo
```

## 包来源

Pi 在设置和 `pi install` 命令中接受三种来源类型。

### npm

```
npm:@scope/pkg@1.2.3
npm:pkg
```

- 带版本的规格会被固定，并在包更新（`pi update`、`pi update --extensions`）时被跳过。
- 用户安装的包存放在 `~/.pi/agent/npm/` 下。
- 项目安装的包存放在 `.pi/npm/` 下。
- 在 `settings.json` 中设置 `npmCommand`，可以将 npm 包的查找和安装操作固定到特定的包装命令，例如 `mise` 或 `asdf`。

示例：

```json
{
  "npmCommand": ["mise", "exec", "node@20", "--", "npm"]
}
```

### git

```
git:github.com/user/repo@v1
git:git@github.com:user/repo@v1
https://github.com/user/repo@v1
ssh://git@github.com/user/repo@v1
```

- 没有 `git:` 前缀时，只接受带协议的 URL（`https://`、`http://`、`ssh://`、`git://`）。
- 有 `git:` 前缀时，接受简写格式，包括 `github.com/user/repo` 和 `git@github.com:user/repo`。
- 同时支持 HTTPS 和 SSH URL。
- SSH URL 会自动使用你配置的 SSH 密钥（遵循 `~/.ssh/config`）。
- 对于非交互式运行（例如 CI），可以设置 `GIT_TERMINAL_PROMPT=0` 禁用凭据提示，并设置 `GIT_SSH_COMMAND`（例如 `ssh -o BatchMode=yes -o ConnectTimeout=5`）以快速失败。
- 引用（refs）是固定的标签或提交。`pi update` 和 `pi update --extensions` 不会将它们移动到更新的引用，但会协调现有克隆到已配置的引用。
- 使用 `pi install git:host/user/repo@new-ref` 来更新设置并将现有包移动到新的固定引用。
- 克隆到 `~/.pi/agent/git/<host>/<path>`（全局）或 `.pi/git/<host>/<path>`（项目）。
- 当协调导致检出更改时，Pi 会重置并清理克隆，然后如果存在 `package.json` 则运行 `npm install`。

SSH 示例：

```bash
# git@host:path 简写（需要 git: 前缀）
pi install git:git@github.com:user/repo

# ssh:// 协议格式
pi install ssh://git@github.com/user/repo

# 带版本引用
pi install git:git@github.com:user/repo@v1.0.0
```

### 本地路径

```
/absolute/path/to/package
./relative/path/to/package
```

本地路径指向磁盘上的文件或目录，添加至设置时不会复制。相对路径会相对于它们所在的设置文件进行解析。如果路径是一个文件，它将作为单个扩展加载。如果是一个目录，Pi 将使用包规则加载资源。

## 创建一个 Pi 包

在 `package.json` 中添加 `pi` 清单，或者使用约定的目录结构。加入 `pi-package` 关键字以提高可发现性。

```json
{
  "name": "my-package",
  "keywords": ["pi-package"],
  "pi": {
    "extensions": ["./extensions"],
    "skills": ["./skills"],
    "prompts": ["./prompts"],
    "themes": ["./themes"]
  }
}
```

路径相对于包的根目录。数组支持 glob 模式和 `!` 排除。

### 画廊元数据

包画廊会展示标记了 `pi-package` 的包。添加 `video` 或 `image` 字段来显示预览：

```json
{
  "name": "my-package",
  "keywords": ["pi-package"],
  "pi": {
    "extensions": ["./extensions"],
    "video": "https://example.com/demo.mp4",
    "image": "https://example.com/screenshot.png"
  }
}
```

- `video`：仅支持 MP4。在桌面端，悬停时自动播放，点击打开全屏播放器。
- `image`：支持 PNG、JPEG、GIF 或 WebP。显示为静态预览。
- 如果两者都设置，`video` 优先。

## 包结构

### 约定目录

如果没有提供 `pi` 清单，Pi 会从以下目录自动发现资源：

- `extensions/` 加载 `.ts` 和 `.js` 文件
- `skills/` 递归查找 `SKILL.md` 文件夹，并将根目录下的 `.md` 文件加载为技能
- `prompts/` 加载 `.md` 文件
- `themes/` 加载 `.json` 文件

## 依赖

第三方运行时依赖应放在 `package.json` 的 `dependencies` 中。那些不注册扩展、技能、提示模板或主题的依赖也应放在 `dependencies` 中。当 Pi 从 npm 或 git 安装一个包时，它会运行 `npm install`，因此这些依赖会被自动安装。

Pi 为扩展和技能捆绑了核心包。如果你导入了以下任何包，请在 `peerDependencies` 中以 `"*"` 范围列出它们，而**不要**将它们打包在一起：`@earendil-works/pi-ai`、`@earendil-works/pi-agent-core`、`@earendil-works/pi-coding-agent`、`@earendil-works/pi-tui`、`typebox`。

其他 pi 包**必须**打包在你的压缩包中。将它们添加到 `dependencies` 和 `bundledDependencies`，然后通过 `node_modules/` 路径引用它们的资源。Pi 使用单独的模块根加载包，因此分开安装不会冲突或共享模块。

示例：

```json
{
  "dependencies": {
    "shitty-extensions": "^1.0.1"
  },
  "bundledDependencies": ["shitty-extensions"],
  "pi": {
    "extensions": ["extensions", "node_modules/shitty-extensions/extensions"],
    "skills": ["skills", "node_modules/shitty-extensions/skills"]
  }
}
```

## 包过滤

使用设置中的对象形式来过滤包加载的内容：

```json
{
  "packages": [
    "npm:simple-pkg",
    {
      "source": "npm:my-package",
      "extensions": ["extensions/*.ts", "!extensions/legacy.ts"],
      "skills": [],
      "prompts": ["prompts/review.md"],
      "themes": ["+themes/legacy.json"]
    }
  ]
}
```

- `+path` 和 `-path` 是相对于包根目录的精确路径。
- 省略某个键表示加载该类型的所有资源。
- 使用 `[]` 表示不加载该类型的任何资源。
- `!pattern` 排除匹配项。
- `+path` 强制包含一个精确路径。
- `-path` 强制排除一个精确路径。

过滤层作用在清单之上。它们会**缩减**已允许加载的内容。

## 启用和禁用资源

使用 `pi config` 来启用或禁用来自已安装包和本地目录的扩展、技能、提示模板和主题。适用于全局 (`~/.pi/agent`) 和项目 (`.pi/`) 作用域。

## 作用域和去重

包可以同时出现在全局和项目设置中。如果同一个包在两者中都出现，**项目条目优先**。身份的判定依据是：

- npm：包名称
- git：不含引用（ref）的仓库 URL
- local：解析后的绝对路径

---

## 💡 Pi 包系统详解与示例

Pi 的包系统不仅仅是一个插件管理器，而是一个**可组合、可分发的能力生态基础**。通过 npm 和 git 分发扩展、技能、提示模板和主题，它允许你将自定义的 AI 工作流打包成可复用的单元，并在团队或社区中共享。

### 1. 核心概念：能力打包与分发

一个 Pi 包本质上是一个 Node.js 包（带有 `package.json`），它通过 `pi` 清单声明自己提供了哪些资源。资源分为四类：

- **extensions**：TypeScript 扩展，注入工具、事件监听器等。
- **skills**：符合 Agent Skills 规范的 `SKILL.md` 目录。
- **prompts**：可复用的 `/` 提示模板。
- **themes**：TUI 主题文件。

通过包机制，你可以将一套完整的“AI 编程环境定制”作为一个整体安装和更新。

### 2. 安装与管理工作流

安装一个包时，Pi 会将其记录到 `settings.json` 中，并下载依赖（如果是 npm 包则安装到 `~/.pi/agent/npm/`，git 包则克隆到 `~/.pi/agent/git/`）。项目级的包可随 `.pi/settings.json` 一起提交到 Git，团队成员拉取代码后启动 Pi 时会自动安装缺失的包，实现环境一致性。

#### 举例：安装一个 git 包并过滤资源

假设团队有一个内部 git 仓库 `github.com/myteam/pi-tools`，其中包含许多扩展和技能。某个项目只需要其中的 `code-review` 技能和 `git-helpers` 扩展，不需要其他。团队可以在项目设置 `.pi/settings.json` 中这样配置：

```json
{
  "packages": [
    {
      "source": "git:github.com/myteam/pi-tools@v2.0",
      "skills": ["skills/code-review"],
      "extensions": ["extensions/git-helpers.ts"],
      "prompts": [],
      "themes": []
    }
  ]
}
```

这样启动 Pi 时，只会加载指定的技能和扩展，其他资源被忽略。

### 3. 创建自己的包：从约定到清单

**最小约定包**：如果你只是简单地将几个文件放到约定的目录中，比如：

```
my-tools/
├── extensions/
│   └── hello.ts
├── skills/
│   └── my-skill/
│       └── SKILL.md
└── prompts/
    └── review.md
```

然后在项目设置中添加本地路径：

```bash
pi install ./my-tools
```

Pi 会自动发现 `extensions/`、`skills/`、`prompts/` 并加载它们。

**带清单的包**：如果要发布到 npm 或 git，则创建一个 `package.json`：

```json
{
  "name": "my-pi-tools",
  "version": "1.0.0",
  "keywords": ["pi-package"],
  "pi": {
    "extensions": ["./extensions"],
    "skills": ["./skills"],
    "prompts": ["./prompts"],
    "video": "https://example.com/demo.mp4"
  }
}
```

发布后，其他人可以通过 `pi install npm:my-pi-tools` 安装。

### 4. 依赖处理与隔离

Pi 包的依赖有两种：

- **运行时依赖**：你的扩展或技能脚本需要用到的第三方库（如 `zod`），列在 `dependencies` 中，安装时 Pi 会执行 `npm install`。
- **核心库**：如 `@earendil-works/pi-coding-agent` 等 Pi 自身提供的 API，应该列为 `peerDependencies`，不要打包到你的包中。这避免了多个包携带不同版本的核心库导致冲突。
- **复用其他 pi 包**：如果你的包需要使用另一个 pi 包的扩展或技能，必须将它放到 `bundledDependencies` 中，然后在 `pi` 清单中通过 `node_modules/` 路径引用。例如：

```json
{
  "dependencies": {
    "another-pi-pkg": "^1.0.0"
  },
  "bundledDependencies": ["another-pi-pkg"],
  "pi": {
    "extensions": [
      "extensions",
      "node_modules/another-pi-pkg/extensions"
    ]
  }
}
```

这保证了每个包有独立的依赖树，避免冲突。

### 5. 包过滤的高级控制

过滤机制让你可以在项目级别精确控制每个包加载的内容。假设你安装了一个庞大的“万能工具包”，但只需要其中一两个功能，可以使用 `!` 排除不需要的，或者使用 `+` 强制包含某些路径。

**举例：排除某个不稳定扩展**

```json
{
  "packages": [
    {
      "source": "npm:my-suite",
      "extensions": ["extensions/*.ts", "!extensions/experimental.ts"]
    }
  ]
}
```

这样 `experimental.ts` 就不会被加载，其他扩展正常加载。

**举例：强制加载一个不在清单中的主题**

有时候包作者可能遗漏了某个主题文件，你可以用 `+path` 强制加载它（前提是文件存在）：

```json
{
  "packages": [
    {
      "source": "npm:my-suite",
      "themes": ["+themes/secret-theme.json"]
    }
  ]
}
```

### 6. 去重与作用域

如果你在全局安装了一个包，又在项目中安装了同一个包的不同版本，项目版本优先。这允许项目覆盖全局设置，确保项目依赖的版本一致性。去重基于 npm 包名、git 仓库 URL 或本地绝对路径，逻辑清晰。

### 7. 示例：构建一个完整的团队效率包

假设你想为团队创建一个包，包含：

- 一个扩展，在每次工具调用前自动记录日志。
- 一个技能，提供部署到 Kubernetes 的标准化流程。
- 一个提示模板，用于生成变更日志。

目录结构：

```
team-workflow/
├── package.json
├── extensions/
│   └── audit-log.ts
├── skills/
│   └── k8s-deploy/
│       └── SKILL.md
├── prompts/
│   └── changelog.md
```

`package.json`：

```json
{
  "name": "@myteam/workflow",
  "version": "1.0.0",
  "keywords": ["pi-package"],
  "pi": {
    "extensions": ["./extensions"],
    "skills": ["./skills"],
    "prompts": ["./prompts"]
  }
}
```

发布到公司内部 npm registry 后，团队成员只需执行：

```bash
pi install npm:@myteam/workflow@1.0.0
```

启动 Pi 后，他们就能使用统一的审计日志、K8s 部署技能和变更日志模板。当包更新时，通过 `pi update` 即可同步，而项目设置文件可以确保所有人使用相同版本。

### 8. 总结

Pi 的包系统通过标准化的 `package.json` 清单和约定目录，将扩展、技能、提示模板和主题统一为可分发、可版本化的单元。它与 npm/git 生态无缝集成，提供精细化过滤和去重机制，使得团队可以像管理代码依赖一样管理 AI 辅助开发环境。这种能力生态正在将“AI 工程最佳实践”转化为可安装的软件包，推动整个领域的工程化进程。