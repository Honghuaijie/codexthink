# Codex Skills 官方文档总结

基于 OpenAI 官方文档整理：

- `https://developers.openai.com/codex/skills`
- `https://developers.openai.com/codex/plugins/build`

本文重点回答两个问题：

1. Codex 如何导入、发现、加载 skill
2. 如何自己编写一个可被 Codex 使用的 skill

---

## 1. 先说结论

在 Codex 体系里：

- `skill` 是工作流本体，是一套可复用的任务说明和资源
- `plugin` 是分发单位，用来打包和安装一个或多个 skill，也可以顺带打包 app、MCP 配置等内容
- Codex 会从若干固定目录自动扫描 skill
- 一个 skill 本质上就是一个目录，里面至少要有一个 `SKILL.md`

所以，平时自己写工作流时，重点是“写 skill”；当你想把 skill 分享给别人、复用给多个团队、或者和 app 能力一起发布时，再考虑“打包成 plugin”。

---

## 2. Codex 如何发现和导入 skill

### 2.1 Skill 的发现机制

官方文档说明，Codex 会从以下位置读取 skills：

### 术语说明

- `$CWD`
  - Current Working Directory，当前工作目录
  - 指你启动 Codex 时所在的目录
- `$HOME`
  - 当前用户的家目录
  - 例如你的 macOS 用户目录通常类似 `/Users/hhj`
- `$REPO_ROOT`
  - 当前 Git 仓库的根目录
  - 只有在当前目录处于某个 Git 仓库内部时，这个概念才成立

举例：

- 如果你在 `/Users/hhj/dev/project/service/api` 里启动 Codex
- 且这个项目的 Git 根目录是 `/Users/hhj/dev/project`

那么：

- `$CWD = /Users/hhj/dev/project/service/api`
- `$REPO_ROOT = /Users/hhj/dev/project`
- `$HOME = /Users/hhj`

### `REPO` 级 仓库级别（该 skill 只能在项目中使用）

`$CWD` 是你启动 Codex 时所在的目录，不是命令。

- `$CWD/.agents/skills`
- `$CWD` 到仓库根目录之间每一层目录下的 `.agents/skills`（也就是说项目中的不同层级都可以有自己的 skill）
- `$REPO_ROOT/.agents/skills`

这意味着：如果你在一个 Git 仓库的子目录里启动 Codex，它会沿着当前目录一路向上扫描，直到仓库根目录，查找每一层的 `.agents/skills`。

补充理解：

- 当前目录可以放当前模块专属的 skill
- 父目录可以放多个子模块共享的 skill
- 仓库根目录可以放整个项目通用的 skill

适合场景：

- 某个微服务单独需要一套技能
- 某个仓库全体成员共用一套技能
- 某个父目录下的多个模块共用一套技能

### 2.2 如果当前目录没有 `git init`

如果当前项目不是一个 Git 仓库，没有执行过 `git init`，那通常就没有严格意义上的“仓库根目录”。

这时可以这样理解：

- 当前目录下的 `.agents/skills` 仍然可能是有效的 skill 入口
- 但“从 `$CWD` 一路向上扫描到 `$REPO_ROOT`”这条 repo 级规则，通常不会像 Git 仓库那样完整成立
- 更稳妥的做法是把 skill 放在：
  - 当前目录的 `.agents/skills`
  - 或用户级的 `~/.agents/skills`

简单说：

- 有 Git 仓库时，可以理解为“仓库级 skill 发现”
- 没有 Git 仓库时，更接近“当前目录级 + 用户级 skill 发现”

### `USER` 级

- `$HOME/.agents/skills`

这是用户自己的全局 skill 目录。放在这里的 skill，理论上可以在你所有项目里被 Codex 发现。

### `ADMIN` 级
对这台机器上的所有用户都适用
- `/etc/codex/skills`

这是机器或容器级的共享目录，适合团队统一预装技能。

### `SYSTEM` 级

- OpenAI 随 Codex 内置的 skills

例如 `skill-creator`、`skill-installer` 这类系统技能。

---

## 3. Codex 是怎么“加载” skill 的

官方文档强调，Codex 对 skill 的处理不是一次性把所有内容全塞进上下文，而是采用 **progressive disclosure**。（逐步分析）

### 3.1 第一步：先读取 metadata

Codex 启动后，先只看到每个 skill 的这些信息：

- `name`
- `description`
- 文件路径
- 可选的 `agents/openai.yaml` 元数据

也就是说，最开始进入上下文的不是整个 skill，而只是“技能简介”。

这些 metadata 主要来自两个地方：

- `SKILL.md` 文件头部的 YAML frontmatter
  - 这是最核心、也是必须存在的 metadata 来源
  - 至少要包含 `name` 和 `description`
- `agents/openai.yaml`
  - 这是可选的补充 metadata
  - 这个文件如果存在，是放在“某个具体 skill 自己的目录里”，不是全局共享一个
  - 常用于配置 UI 展示信息、是否允许隐式触发、工具依赖等

可以这样理解：

- `SKILL.md` frontmatter = 基础 metadata
- `agents/openai.yaml` = 补充 metadata

Codex 的读取顺序可以粗略理解为：

```text
扫描到某个 skill 目录
-> 读取 SKILL.md 的 frontmatter
-> 如果这个 skill 目录下还有 agents/openai.yaml，也一起读取
-> 将这些 metadata 纳入可用 skills 列表
-> 只有真正触发 skill 时，才继续加载完整的 SKILL.md 正文
```

### 3.1.1 用 `superpowers` 举一个真实例子

假设你本机已经安装了 `superpowers`，并且目录关系如下：

```text
~/.agents/skills/superpowers
-> ~/.codex/superpowers/skills
```

这里的 `~/.agents/skills/superpowers` 是一个软链接，真正的 skill 内容放在 `~/.codex/superpowers/skills`。

`superpowers` 不是一个单独 skill，而是一组 skills 的集合，例如：

- `using-superpowers`
- `brainstorming`
- `systematic-debugging`

其中某个具体 skill 的真实文件路径可能是：

```text
~/.codex/superpowers/skills/using-superpowers/SKILL.md
```

这个文件开头大致是：

```md
---
name: using-superpowers
description: Use when starting any conversation - establishes how to find and use skills, requiring Skill tool invocation before ANY response including clarifying questions
---
```

那么，Codex 对这套 `superpowers` 的 metadata 读取过程，可以理解为：

```text
Codex 启动
-> 扫描用户级目录 ~/.agents/skills
-> 发现 superpowers 是一个软链接
-> 跟随软链接进入 ~/.codex/superpowers/skills
-> 遍历其中每个子目录
-> 发现 using-superpowers/ 目录下有 SKILL.md
-> 读取 SKILL.md 顶部 frontmatter
-> 提取出：
   - name = using-superpowers
   - description = Use when starting any conversation ...
-> 把这组信息加入可用 skills 列表
-> 当任务匹配 description，或者用户显式点名这个 skill 时
-> 才继续加载 using-superpowers 的完整正文
```

在这套 `superpowers` 里，如果没有额外的 `agents/openai.yaml`，那 Codex 主要就是从每个 `SKILL.md` 的 frontmatter 读取 metadata。

所以，对 `superpowers` 这个例子来说，最准确的理解是：

- Codex 不是把整个 `superpowers` 仓库当成一个 skill 读取
- Codex 是先通过 `~/.agents/skills/superpowers` 找到 skill 集合目录
- 再逐个读取其中每个 skill 子目录里的 `SKILL.md` frontmatter
- 每个子目录才是一个真正的 skill

### 3.2 第二步：决定是否触发

Codex 有两种触发方式：

1. 显式触发
   - 在 prompt 里直接提 skill
   - 在 CLI / IDE 中用 `/skills`
   - 输入 `$skill-name`

2. 隐式触发
   - 当用户任务和 skill 的 `description` 匹配时，Codex 会主动选择这个 skill

因此，`description` 非常重要。它不是普通介绍语，而是“触发规则”的核心。

### 3.3 第三步：只在需要时加载 `SKILL.md`

只有当 Codex 决定使用某个 skill 时，它才会读取这个 skill 的完整 `SKILL.md` 内容。  
这就是为什么官方建议：

- skill 要聚焦
- `SKILL.md` 不要过长
- 详细资料放在 `references/`
- 需要稳定执行的动作放在 `scripts/`

---

## 4. Codex 支持软链接

官方文档明确说明：

- Codex 支持 symlinked skill folders
- 扫描 skill 目录时，会跟随软链接继续读取目标目录

这就是为什么类似下面的做法成立：

```bash
ln -s ~/.codex/superpowers/skills ~/.agents/skills/superpowers
```

其含义不是“把文件复制过去”，而是：

- 真正的仓库存放在 `~/.codex/superpowers/skills`
- Codex 从 `~/.agents/skills` 扫描时，看到了一个名为 `superpowers` 的软链接
- 然后顺着软链接找到真实 skill 目录并加载

这种方案的优点是：

- 仓库本身可以继续用 `git pull` 更新
- `~/.agents/skills` 保持为统一的发现入口
- 不需要每次更新都重新复制 skill 文件

---

## 5. 一个 skill 的最小结构是什么

一个最小可用 skill 至少要满足：

```text
my-skill/
└── SKILL.md
```

`SKILL.md` 必须包含 YAML frontmatter，至少有两个字段：

- `name`
- `description`

最小示例：

```md
---
name: my-skill
description: Use when Codex needs to handle ...
---

Skill instructions for Codex to follow.
```

官方推荐的完整目录结构通常是：

```text
my-skill/
├── SKILL.md
├── scripts/
├── references/
├── assets/
└── agents/
    └── openai.yaml
```

其中：

- `SKILL.md`
  - 必需
  - 这是 skill 的核心文件
  - 主要用来告诉 Codex：这个 skill 是什么、什么时候该触发、触发后应该怎么做
- `scripts/`
  - 可选
  - 场景： 对于一些skill来说，可能会经常执行一些重复的命令，这是你将这些命令写到scripts中，ai可以直接执行这些重复命令
  - 用来放可执行脚本
  - 适合那些“每次都重复写一遍很麻烦”的动作
  - 例如：批量处理文件、格式转换、调用命令行工具
  - 可以理解为：让 skill 不只是“会说”，还能“按固定方式做”
- `references/`
  - 可选
  - 用来放参考资料
  - 适合放较长的说明文档、业务规则、接口文档、schema、操作规范
  - 这些内容通常不是一开始全加载，而是在需要时再读
  - 可以理解为：给 skill 提供背景知识和查阅材料
- `assets/`
  - 可选
  - 用于给skill用的素材
  - 适合放模板、图标、图片、静态资源、脚手架代码等
  - 这些通常不是给 Codex 重点阅读的，而是给它直接拿来使用的
  - 例如：生成新文件时直接套用模板
- `agents/openai.yaml`
  - 可选
  - 这是某个具体 skill 自己的补充配置文件
  - 常用于配置 UI 展示信息、依赖、默认提示、是否允许隐式触发等
  - 可以理解为：skill 的附加元信息配置

---

## 6. 如何自己写一个 skill

官方文档给了两种方式：

### 6.1 用内置工具创建

推荐先用：

```text
$skill-creator
```

它会帮助你明确：

- skill 是做什么的
- 什么时候触发
- 是纯说明型 skill，还是需要脚本/资源

这属于“官方推荐路径”。

### 6.2 手工创建

你也可以手工新建一个目录，再写一个 `SKILL.md`。

例如：

```text
.agents/skills/my-git-helper/
└── SKILL.md
```

`SKILL.md` 示例：

```md
---
name: my-git-helper
description: Help with safe, non-destructive git inspection and branch hygiene. Use when Codex needs to inspect git status, compare branches, summarize commits, or suggest safe cleanup steps without rewriting history.
---

# Git Helper

1. Inspect the current branch and worktree state first.
2. Prefer non-destructive git commands.
3. Do not run reset, checkout --, or force-push unless explicitly requested.
4. Summarize findings before proposing cleanup.
```

然后只要把这个 skill 放到 Codex 的扫描路径中，例如：

- 当前仓库的 `.agents/skills/`
- 用户目录的 `~/.agents/skills/`

Codex 就能发现它。

---

## 7. 写 skill 时最重要的字段：`description`

官方文档明确说明，隐式匹配依赖 `description`，因此这个字段必须写清楚：

- skill 做什么
- 什么时候应该触发
- 什么时候不该触发

一个好的 `description` 应该：

- 范围清楚
- 边界明确
- 包含触发场景
- 避免过于宽泛

不好的写法：

```yaml
description: Help with coding.
```

问题：

- 范围太大
- 几乎任何任务都可能匹配
- Codex 难以判断是否应该触发

更好的写法：

```yaml
description: Analyze and update protobuf service definitions. Use when Codex needs to add RPC methods, revise proto message fields, or keep API contracts aligned with generated backend interfaces.
```

这样 Codex 更容易在“proto 设计、RPC 变更、接口契约”相关任务里选中它。

---

## 8. 官方对 skill 内容的设计思想

官方文档背后的设计思想可以总结成三点：

### 8.1 Skill 要聚焦

一个 skill 只解决一类任务，不要试图包办所有事情。

### 8.2 说明优先，脚本其次

如果纯文字说明已经足够，就先不要上脚本。  
只有在这些情况下再考虑 `scripts/`：

- 需要稳定、确定的行为
- 某段逻辑经常重复写
- 调用外部工具更合适

### 8.3 把长内容拆出去

不要把所有细节都塞进 `SKILL.md`。  
推荐做法是：

- 把核心触发逻辑和主流程留在 `SKILL.md`
- 把大段文档放入 `references/`
- 把模板、静态资源放入 `assets/`

这样可以减少上下文占用，提高 Codex 选择和执行 skill 的效率。

---

## 9. Skill 和 Plugin 的区别

这是最容易混淆的一点。

### Skill

- 是工作流本身
- 是作者编写的能力单元
- 本质上是一个目录 + `SKILL.md`

### Plugin

- 是安装和分发单位
- 可以包含一个或多个 skill
- 还可以带上 app mappings、MCP server 配置、展示资源等

适合理解为：

- 你“设计工作流”时，写的是 skill
- 你“对外发布能力包”时，打的是 plugin

如果只是自己本地用，或者在仓库里团队内部使用，直接放 `.agents/skills` 往往就够了。  
如果你要更规范地给别人安装和复用，再考虑 plugin。

---

## 10. 推荐的实战做法

如果你只是想自己用一个 skill，最推荐的路径是：

1. 先想清楚 skill 的单一职责
2. 写一个清晰的 `name` 和 `description`
3. 在 `~/.agents/skills/` 或仓库的 `.agents/skills/` 下建目录
4. 写最小版本 `SKILL.md`
5. 重启 Codex 或等待自动发现
6. 用显式触发先测试，例如 `$my-skill`
7. 再逐步优化 `description`，让它能被隐式匹配

如果你已经有一个 Git 仓库来管理 skills，推荐：

1. 把仓库存放在一个你喜欢的位置
2. 用软链接把 skill 目录挂到 `~/.agents/skills`
3. 让 Codex 从统一入口发现它们

---

## 11. 一个最简的自定义 skill 示例

假设你想写一个专门处理 proto 变更的 skill，可以这样组织：

```text
~/.agents/skills/proto-editor/
└── SKILL.md
```

内容示例：

```md
---
name: proto-editor
description: Work with protobuf definitions and RPC contracts. Use when Codex needs to add or modify messages, services, enums, or field definitions in .proto files, and when API contract changes must stay consistent with generated backend code.
---

# Proto Editor

1. Read the existing `.proto` file before proposing changes.
2. Preserve package, version, and naming conventions already used by the repository.
3. Check whether service, request, response, and model fields stay aligned.
4. Prefer additive changes when backward compatibility matters.
5. Call out any breaking schema changes before editing.
```

这个 skill 的好处是：

- 触发范围明确
- 行为约束明确
- 非常适合通过 `description` 被隐式触发

---

## 12. 结合你的问题，最终可以这样理解

关于 “Codex 如何导入 skill”，最准确的说法是：

- Codex 不是通过“安装某个仓库名”来识别 skill
- Codex 是通过扫描约定目录，发现其中符合结构的 skill 目录
- 一个 skill 目录里必须有 `SKILL.md`
- 如果目录是软链接，Codex 也会跟随链接继续读取

关于 “如何自己写一个 skill”，最准确的说法是：

- 先写好一个聚焦的 skill 目录
- 至少包含一个带 `name` 和 `description` 的 `SKILL.md`
- 把它放进 `.agents/skills` 或 `~/.agents/skills`
- 必要时再补充 `scripts/`、`references/`、`assets/`、`agents/openai.yaml`

---

## 13. 参考链接

- Codex Skills 官方文档  
  `https://developers.openai.com/codex/skills`

- Codex Plugins 官方文档  
  `https://developers.openai.com/codex/plugins/build`
