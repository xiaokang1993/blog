---
title: omp 实战教程：把终端变成你的编码 agent
date: 2026-10-30 10:00:00
tags:
  - omp
  - AI
  - coding agent
  - 工具
categories:
  - 技术教程
---

## 一、omp 是什么

[omp](https://github.com/can1357/oh-my-pi)（Oh My Pi）是一个**终端优先**的 coding agent：你在项目目录里运行它，用自然语言描述想要的结果，它会自己去读代码、改文件、跑命令、解释结果，你只需要审阅。

它和"把代码粘贴进聊天窗口"的区别在于：omp 直接使用你机器上现有的仓库和开发环境。搜索、编辑、执行、语言智能、调试、浏览器、GitHub 上下文都在同一个会话里完成，工作过程全程可见。

本文基于 [Oh My Pi 中文文档](https://aieguu.github.io/omp-docs-cn/)（对应上游 `18.1.10`）整理，按"装 → 用 → 管 → 扩"的顺序走一遍，重点放在**能立刻用上的操作和容易踩的坑**上。

<!-- more -->

## 二、安装与首次配置

### 2.1 安装

macOS / Linux：

```bash
curl -fsSL https://omp.sh/install | sh
omp --version
```

Windows 用 PowerShell：`irm https://omp.sh/install.ps1 | iex`。如果提示找不到 `omp`，按安装器打印的 PATH 提示操作，再开一个新终端。

顺手把补全装上，后面敲 `/` 命令和子命令会舒服很多：

```bash
# zsh: 写入 ~/.zshrc
eval "$(omp completions zsh)"
# bash: 写入 ~/.bashrc
eval "$(omp completions bash)"
```

### 2.2 启动目录就是项目根目录

```bash
cd ~/code/my-project
omp
```

第一次启动会进入设置向导：先登录 Provider（可以登录多个），再选默认模型，最后调终端外观。之后的启动直接进入提示符。

如果你想用 API key 而非 OAuth，在启动前导出即可：

```bash
export ANTHROPIC_API_KEY=sk-ant-...
omp
```

不想反复写 export 又不想污染 shell 启动文件，可以把密钥放进 `.env`。omp 的加载顺序是：`<当前目录>/.env` → `~/.omp/agent/.env` → `~/.omp/.env` → `~/.env`，已导出的进程变量优先。**项目里的 `.env` 别提交。**

### 2.3 选 Provider 的几条路线

| 路线 | 适用场景 | 第一步 |
| --- | --- | --- |
| OAuth / 账户登录 | ChatGPT Plus、Claude Pro/Max、Copilot 等订阅 | `/login openai-codex` |
| API key | 按量计费、CI、脚本 | 导出环境变量，或 `/login` 引导录入 |
| 本地引擎 | Ollama、llama.cpp、LM Studio | 先启动引擎，`omp models ollama` |
| 自定义网关 | 团队网关、自托管 | 写 `~/.omp/agent/models.yml` |

一个高频坑：**订阅和 API 是两套产品**。ChatGPT 凭据用 `openai-codex`，开发者 API 用 `openai`（读 `OPENAI_API_KEY`）；Google 的 `google` / `google-gemini-cli` / `google-vertex` 同样不可互换。拿不准就先跑 `omp models`，那份输出才是权威的。

自定义网关的最小配置：

```yaml
# ~/.omp/agent/models.yml
providers:
  my-gateway:
    baseUrl: https://gateway.example.com/v1
    api: openai-completions
    apiKey: MY_GATEWAY_API_KEY   # 写环境变量名，别写明文
    models:
      - id: fast-chat
        name: Fast Chat
        contextWindow: 128000
        maxTokens: 8192
```

### 2.4 第一个任务：小而可验证

第一轮不要上来就"重构整个项目"。给它一个边界明确、有验证手段的任务：

```
Inspect this project for one small bug or inconsistency that can be verified locally.
Explain what you found, make the smallest safe fix, and run the most relevant check.
```

按 Enter，omp 会以工具卡片的形式流式展示每一步：读了哪些文件、跑了什么命令、改了哪几行。按 `Ctrl+O` 展开完整输出和 diff。

只想要结论时，明确说"只调查，不要修改"——这句话能省掉大量返工。

## 三、日常交互：读懂屏幕、控制轮次

### 3.1 三个区域

- **Transcript（对话记录）**：消息、思考、工具卡片。卡片会标记等待/成功/失败/取消，编辑卡片带路径和 diff 摘要。
- **Composer（输入区）**：写提示词的地方。`@` 引用文件，`Ctrl+G` 用外部编辑器写长草稿。
- **Status line（状态栏）**：当前模型与思考级别、工作模式、路径、Git 状态、上下文用量、成本。**怀疑它跑错目录、用错模型或处于 Plan 模式时，先看状态栏。**

### 3.2 必须记住的几个键

| 按键 | 作用 |
| --- | --- |
| `Enter` | 发送；若 agent 正在执行，则作为**引导消息**立即影响当前轮次 |
| `Ctrl+Q` / `Ctrl+Enter` | 排队为**后续消息**，等当前任务做完再处理 |
| `Escape` | 中断当前轮次（不会退出 omp） |
| `Ctrl+O` | 展开/折叠工具输出 |
| `Alt+R` | 重试上一次失败的助手轮次 |
| `Ctrl+P` / `Ctrl+Shift+P` | 循环切换模型角色 |
| `Shift+Tab` | 循环思考力度 |
| `Ctrl+D` | 退出并保存草稿 |

引导和排队的区别值得单独说：发现它要改 schema 了，直接 `Enter` 发"不要改 schema"；只是想在完事后解释一下风险，用 `Ctrl+Q`。搞混了会导致提示词在错误的时间点插入。

如果某一轮被打断但方向是对的，发 `.` 或 `c` 就能不重复陈述地继续。

### 3.3 在输入区直接跑命令

不用让模型帮你跑 `git status`：

| 前缀 | 行为 |
| --- | --- |
| `! git status` | 在你的 shell 里执行，结果进上下文 |
| `!! git diff --stat` | 执行但**不进**上下文 |
| `$ print(2+2)` | 在会话共享 Python 内核里执行，结果进上下文 |
| `$$ ...` | 执行但不进上下文 |

注意：`!!` 和 `$$` 只是把输出排除在上下文之外，**不是沙箱**。

### 3.4 会话不会丢

omp 没有"保存"按钮，完成的消息会自动归档到与工作目录绑定的会话里：

```bash
omp -c   # 继续当前目录最近一次会话
omp -r   # 打开会话选择器
```

选择器里按 `Tab` 可在"当前文件夹"和"所有项目"之间切换，输入文字可以按标题、ID、路径甚至对话内容搜索。常用会话建议 `/rename` 加 `/pin`，列表顶部就有了固定入口。

走岔路时：

| 想做的事 | 命令 |
| --- | --- |
| 回到更早的消息，另起一条线 | `/tree` |
| 从某条旧消息开出替代分支 | `/branch` |
| 整段对话克隆成新会话 | `/fork` |
| 上下文快满了 | `/compact` |

## 四、让它少走弯路：规则与技能

### 4.1 `AGENTS.md`：一次性交代项目规则

在仓库根目录放一个 `AGENTS.md`，omp 会自动加载并在整个会话生效：

```markdown
# Project instructions

## Build and test
- Install with `pnpm install`.
- Run unit tests with `pnpm test`.
- Never edit generated files under `src/generated/`.

## Conventions
- Keep public API changes backward compatible.
- Before finishing, run the narrowest test that covers the change.
```

发现规则是**从启动目录向上遍历**，从通用到本地叠加加载：

```
~/.omp/agent/AGENTS.md       用户级
~/work/AGENTS.md             工作区级
~/work/repo/AGENTS.md        仓库级
~/work/repo/packages/api/AGENTS.md   包级
```

这对 monorepo 非常好用：共享命令放根目录，包专属命令放嵌套目录，不用复制粘贴。

一条容易踩的边界：同一作用范围内**优先级最高的那份会遮蔽其它**。`<repo>/.omp/AGENTS.md` 会遮蔽 `<repo>/AGENTS.md`，两者不会拼接。要确认到底加载了哪份，运行 `/extensions` 看 **Context Files** 面板，那才是权威视图。

还有 `RULES.md`（始终生效的粘性规则，用户级与最近项目级同时加载）和 `APPEND_SYSTEM.md`（在内置系统提示后追加文本）。想改人格才动 `SYSTEM.md`，否则用 `APPEND_SYSTEM.md` 更安全。

### 4.2 技能：教它"怎么做事"

技能是一个带 `SKILL.md` 的目录，不增加权限，只改变处理任务的方式。最小可用版本：

```markdown
---
description: Use when preparing a release; verify versioning, changelog, artifacts, publish status.
---

# Release check

1. Identify the version and release tag.
2. Verify the changelog describes the shipped behavior.
3. Check build artifacts before publishing.
4. Report blockers before changing a tag or publishing.
```

放到 `.omp/skills/release-check/SKILL.md`（项目级）或 `~/.omp/agent/skills/release-check/SKILL.md`（用户级）。目录名就是调用名，可以写 `/skill:release-check` 显式调用，或者干脆用自然语言描述任务——`description` 写得准，就会自动匹配。

写 `description` 的诀窍是把**动作 + 对象 + 边界**都写进去，例如"Use when adding or reviewing Vitest tests in src/importer; covers fixtures, snapshots, and integration setup."

注意发现深度只有一层：`skills/databases/postgres/SKILL.md` **找不到**。

## 五、大改动先规划，能并行就并行

### 5.1 `/plan`：只读地探索，批准后再动手

跨文件重构、陌生代码库、有顺序约束的迁移，先 `/plan`：

```
/plan Replace the in-memory job queue with Postgres. Preserve the public API,
include a rollback path, and call out every migration step.
```

规划阶段 omp 可以读代码、验证假设，但**不会修改工作树**。结束后会弹出计划审阅界面，四个批准选项分别对应不同的上下文处理策略：

| 选项 | 上下文怎么处理 | 何时用 |
| --- | --- | --- |
| 批准并执行 | 开新会话，规划讨论不带入 | 计划自包含，想给实现留最多空间 |
| 批准并压缩上下文 | 压缩成聚焦已批准计划的摘要 | 探索过程有价值但太占地方 |
| 批准并保留上下文 | 完整保留 | 决策依赖没写进计划的细节 |
| 保存并退出 | 落盘，先不执行 | 留给以后或交接给别人 |

审阅界面里 `a` 批注、`d` 删章节、`c` 复制、`Escape` 关闭但不回滚编辑。`/plan` 本身是个开关：再运行一次是"暂停"，暂停时 `/plan <补充>` 会带着补充意见重新进入。

### 5.2 子代理：拆分独立工作

当任务能按文件或目录切成互不重叠的几块时，让 omp 并行：

```
并行化这次迁移。给每个子代理分配一组文件互不重叠的包，
用隔离工作区进行编辑，由主会话集成并验证结果。
```

内置的专项子代理有 `scout`（只读探查）、`designer`、`reviewer`、`security-reviewer`、`librarian`（查外部库源码）、`task`（通用）、`sonic`（机械改动）。大多数情况下不用点名，描述清楚任务即可。

三件事必须清楚：

1. **子会话不继承主对话**，只拿到任务和共享上下文。关键约定要写在任务里，比如"`packages/api/schema.ts` 是唯一事实来源，生成文件不得编辑"。
2. **默认共享同一个检出**，并发写入可能互相覆盖。要么按文件划分所有权，要么把 `task.isolation.mode` 设为 `auto` 用隔离工作区。
3. **并行会成倍消耗 token 和费用**。从最小够用的扇出开始。

按 `Alt+A` 打开 Agent Hub 看每个子代理的状态、用量、当前工具；`x` 终止，`r` 复活已停放的子代理。

### 5.3 其他顺手的工作流

- `/review`：选基准分支、未提交改动、某个提交或 PR，分发给多个审查 agent，返回按优先级排序的发现。
- `omp commit`：分析改动，提出 conventional commit 分组和 changelog 建议，然后提交；先看一遍加 `--dry-run`。
- `/goal`：给一个跨轮次的长期目标并设 token 预算。

## 六、配置与权限

### 6.1 配置的层次

```text
内置默认值
  < 全局 / Profile 配置（~/.omp/agent/config.yml）
  < 项目配置（<cwd>/.omp/config.yml）
  < PI_CONFIG_FILES 覆盖层
  < --config 覆盖层
  < 运行时 flag 与环境变量
```

映射（mapping）深度合并，**标量和数组直接替换**。项目里的 `disabledProviders: [groq]` 会完全替换全局的 `[anthropic, openai]`，而不是追加——这是"项目配置不生效"最常见的误会。

常用命令：

```bash
omp config list                 # 全部生效值、类型、枚举
omp config get theme.dark
omp config set compaction.enabled false
omp config path                 # 当前 agent 目录
```

`omp config set` 只写全局；项目值必须直接编辑 `.omp/config.yml`。另外注意 omp **不会向上查找**父目录的 `.omp/config.yml`，必须在确切的工作目录里放文件。

### 6.2 审批模式：默认是 yolo

这一步要特别注意：内置默认审批模式是 **`yolo`**，读取、写入、执行全部自动放行。omp 能改文件、能跑命令，这既是它的能力，也是它的风险。

想让它在写入和执行前停下来确认：

```bash
omp --approval-mode always-ask
```

或在 `/settings` → **Interaction → Approvals** 永久修改。也可以按能力单独设置 `tools.approval.<name>` 为 `allow` / `prompt` / `deny`，或用一个受限的工具集启动：

```bash
omp --tools read,glob,grep      # 只读会话
omp --no-tools                  # 不带内置工具
```

提醒一句：审批提示**不是沙箱**。项目配置、插件、自定义工具、MCP 服务器本质上都是你信任的代码，启用前该看源码的还是得看。

## 七、文件、上下文与成本

**文件能力**比想象中多：PDF、Word、Excel、PPT、EPUB 都能读；ZIP、tar 系、7z、RAR 等归档不解包就能看成员；SQLite 能列表、看 schema、查行，也能按主键改一行（改之前先备份）；`ssh://` 路径可以只读远程 UTF-8 文件（直接读取上限 1 MiB）。网页给 URL 会提取正文，依赖 JS 或登录的页面让它用浏览器打开。

**上下文管理**是长会话能不能跑完的关键。`/context` 看当前轮次的 token 预算分布；接近上限时先把手头思路收尾，再 `/compact`。`/shake` 可以在不摘要的情况下剥掉大块工具结果、图像或 thinking 块，比压缩更便宜。

**成本控制**：状态栏的模型、思考级别和成本一直可见。日常任务用默认模型，难推理再 `Shift+Tab` 提思考力度；子代理扇出规模按需给；`omp usage` 能看 Provider 的限额和重置时间。

## 八、写在最后

omp 的价值不在于"AI 会写代码"，而在于它把**调查、修改、验证**这三件事收进了同一个可见的会话里。你依然是决策者，它负责执行和汇报。

上手路径建议按这个顺序：

1. 在一个熟悉的仓库里跑一个小而可验证的任务；
2. 写好 `AGENTS.md`，把团队约定一次性交代清楚；
3. 大改动先 `/plan`，可并行的再拆子代理；
4. 生产环境考虑 `always-ask` 或受限工具集。

完整文档见 [Oh My Pi 中文文档](https://aieguu.github.io/omp-docs-cn/)，上游项目在 [can1357/oh-my-pi](https://github.com/can1357/oh-my-pi)。
