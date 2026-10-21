---
title: 当 AI 接管我的 GitHub 账户——一次从零到部署的全流程心得
date: 2026-10-21 10:30:00
tags:
  - AI
  - GitHub
  - 自动化
  - 工作流
categories:
  - 技术随笔
---

## 缘起：一个大胆的念头

事情要从一只蝴蝶说起——不，要从一句简单的指令说起。

那天我对 AI 助手说："GitHub 新建 blog 仓库。"它没有反问我用什么名字，没有让我先打开浏览器登录，没有让我在网页上填写表单点击按钮。它只是安静地执行了一条 `gh repo create` 命令，几秒后回复我一个链接：`https://github.com/xiaokang1993/blog`。

仓库已经在那里了。空的，干净的，等待被填充。

这个瞬间让我意识到一件事：AI 控制我的 GitHub 账户，不再是科幻小说里的场景，而是此刻正在发生的事实。我所要做的，仅仅是用自然语言描述意图，剩下的——创建仓库、初始化项目、编写配置、提交代码、配置 CI/CD、部署上线——全部由 AI 代劳。

这篇文章，就是这次实践的完整记录与思考。它本身，也是 AI 写的，然后被 AI 部署到了 AI 搭建的网站上。

有点套娃的意思，但确实如此。

## 一、为什么让 AI 控制 GitHub 账户

### 1.1 传统方式的痛点

回想一下，过去搭建一个 GitHub Pages 博客需要多少步骤？

打开 GitHub 网页，点击 New repository，填写名称和描述，选择公开或私有，点击 Create。然后回到终端，`git init`、`git remote add`、创建 `.gitignore`、编写 `_config.yml`、安装依赖、`hexo generate`、`git add`、`git commit`、`git push`。再去仓库 Settings 里找到 Pages，配置 source 分支。如果用 Actions 部署，还要编写 YAML 工作流文件。中间任何一步出错，就得排查、回退、重来。

这些步骤没有一个是困难的，但每一步都需要人去执行。人的注意力是有限的，复制粘贴配置时漏一个缩进，写 YAML 时少一个冒号，push 时选错分支——这些都是真实发生过的错误。错误本身不可怕，可怕的是排查它所消耗的时间。一个缩进问题可能花你二十分钟，而 AI 可能只需要一次重新读取文件就能发现。

### 1.2 AI 介入的价值

让 AI 控制 GitHub 账户，核心价值不在于"AI 比人更聪明"，而在于"AI 不会忘记步骤、不会跳过环节、不会因为疲倦而犯错"。它像一个不知疲倦的执行者，你给它一个目标，它按部就班地完成所有中间环节。

具体来说，AI 介入后在以下环节带来了质变：

- **仓库创建**：一条指令完成，无需浏览器交互
- **项目初始化**：从模板到配置全自动生成
- **依赖管理**：自动选择包管理器，生成 lockfile，处理版本冲突
- **CI/CD 配置**：直接编写 Actions 工作流，推送后自动触发
- **部署验证**：监控 workflow 运行状态，失败时自动读取日志定位问题

这些环节串联起来，原本可能需要一个下午的工作，压缩到了几分钟。

### 1.3 信任的边界

当然，把 GitHub 账户的控制权交给 AI，不是没有顾虑的。

GitHub Personal Access Token 是一把钥匙，它能创建仓库、推送代码、配置 Pages，也能删除仓库、修改设置、泄露密钥。把这样一把钥匙交给 AI，需要建立信任。

这种信任的建立，靠的不是盲目放权，而是**可观察性**——AI 每一步都告诉我它要做什么、做了什么、结果如何。它执行 `gh repo create`，我能看到命令和输出；它修改 `_config.yml`，我能看到变更内容；它推送代码，我能看到 commit message 和 diff。如果哪一步我不认可，我可以随时叫停。

这是关键：AI 不是在"代替"我做决定，而是在"加速"我执行决定。每一个关键节点，决策权仍然在我手中。AI 做的是把"我想做"到"它完成"之间的距离缩到最短。

## 二、实践全景：从零搭建博客

### 2.1 创建仓库

整个流程的第一步是创建 GitHub 仓库。AI 使用的是 GitHub CLI（`gh`），这是 GitHub 官方的命令行工具，通过 Personal Access Token 认证后可以直接操作 GitHub API。

```bash
gh repo create xiaokang1993/blog --public --description "我的博客" --clone=false
```

一条命令，完成了网页上需要七八次点击才能完成的事。`--public` 指定公开仓库，`--description` 设置描述，`--clone=false` 表示不在本地克隆（因为本地已有目录）。

这里有一个细节值得注意：AI 先检查了仓库是否已存在（`gh repo view xiaokang1993/blog`），确认不存在后才创建。这种"先检查再操作"的模式贯穿了整个流程，避免了重复创建或覆盖已有资源的风险。

### 2.2 初始化本地项目

本地已经有一个 Hexo 博客的初始化目录，包含 `_config.yml`、`package.json`、`scaffolds/`、`source/` 等文件。但这些文件是 Hexo 默认生成的，充满了占位内容：`title: Hexo`、`author: John Doe`、`url: http://example.com`。

AI 的工作是把这些占位内容替换为真实信息：

```yaml
title: 小康的博客
subtitle: '记录技术与生活'
description: '小康的个人博客 - 记录技术学习与生活点滴'
author: 小康
language: zh-CN
timezone: Asia/Shanghai
url: https://xiaokang1993.github.io/blog
```

每一项都经过思考：语言设为 `zh-CN` 因为博客面向中文读者，时区设为 `Asia/Shanghai` 因为作者在中国，URL 设为 `https://xiaokang1993.github.io/blog` 因为这是 project 类型的 Pages 站点（而非 user 站点）。

### 2.3 主题管理

博客使用了 Cactus 主题——一个极简、暗色调的 Hexo 主题。理想情况下，主题应以 git submodule 的形式引入，这样既能追踪版本，又不会把主题代码混入博客仓库。

然而，实践中遇到了网络问题：本地无法访问 `github.com`，clone 主题仓库多次超时。这是一个真实的、非理想化的场景。AI 的处理方式体现了"适应性"——它没有卡在反复重试上，而是调整策略：

- 在 CI 工作流中直接 `git clone` 主题到 `themes/cactus`
- 本地提交时不包含主题文件（`themes/` 下仅保留 `.gitkeep`）
- 构建时 CI 环境网络正常，可以成功 clone

这个决策的意义在于：**不为完美方案卡住进度，选择当前条件下的最优解。** submodule 是更优雅的方案，但当前网络条件下无法实现；CI clone 是务实的替代，功能完全等价，只是优雅度稍逊。等网络恢复后随时可以改为 submodule，不影响任何现有逻辑。

### 2.4 依赖管理：选择 pnpm

用户明确要求"默认使用 pnpm"。AI 据此做了以下调整：

- 删除了项目原有的 `yarn.lock`（避免两套 lockfile 共存）
- 从 `package.json` 中移除了 `hexo-theme-landscape`（改用 cactus 主题，landscape 不再需要）
- 运行 `pnpm install` 生成 `pnpm-lock.yaml`
- 在 CI 工作流中使用 `pnpm/action-setup` 和 `--frozen-lockfile` 确保一致性

这里有一个教训值得记录。第一次提交时，lockfile 是在移除 landscape 之前生成的，仍然包含 `hexo-theme-landscape` 的依赖树。CI 运行 `pnpm install --frozen-lockfile` 时，pnpm 检测到 lockfile 与 `package.json` 不一致，直接报错：

```
ERR_PNPM_OUTDATED_LOCKFILE  Cannot install with "frozen-lockfile"
because pnpm-lock.yaml is not up to date with package.json
```

AI 读取了 CI 失败日志，精确定位问题，删除旧 lockfile 后重新 `pnpm install` 生成正确的版本，再次提交推送后 CI 通过。

这个过程完美诠释了 AI 介入的价值：**失败不可怕，快速定位和修复才是关键。** 传统方式下，看到 CI 报红，你需要打开 Actions 页面，找到失败的 step，展开日志，阅读报错信息，理解原因，回到本地修改，重新推送。AI 把这个循环压缩到了秒级——它读取日志、分析原因、执行修复、推送验证，一气呵成。

### 2.5 GitHub Pages 配置

Pages 的配置通过 GitHub API 完成，无需打开浏览器：

```bash
gh api repos/xiaokang1993/blog/pages -X POST -f build_type=workflow
```

`build_type=workflow` 告诉 GitHub：这个站点通过 GitHub Actions 构建和部署，而非传统的 `gh-pages` 分支或 `docs/` 目录。这是 GitHub 推荐的现代部署方式，更灵活，构建过程完全可控。

### 2.6 部署工作流

CI 工作流文件 `.github/workflows/deploy.yml` 是整个自动化链条的核心。它的结构清晰：

```yaml
on:
  push:
    branches: [main]
  workflow_dispatch:
```

触发条件：推送到 `main` 分支，或手动触发。

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Clone cactus theme
        run: git clone --depth 1 https://github.com/probberechts/hexo-theme-cactus.git themes/cactus
      - uses: pnpm/action-setup@v4
        with:
          version: 9
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
      - uses: actions/configure-pages@v5
      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./public
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
    steps:
      - uses: actions/deploy-pages@v4
```

两个 job：`build` 负责构建，`deploy` 负责发布。`deploy` 依赖 `build`（`needs: build`），构建失败则不会尝试部署。`concurrency` 配置确保同一时间只有一个部署在运行，避免重复构建和覆盖。

这个工作流的设计体现了几个原则：

- **幂等性**：每次构建从零开始，不依赖之前的状态
- **原子性**：构建和部署分离，构建产物作为 artifact 传递，部署步骤只负责发布
- **可观测性**：每一步都有明确的名称，失败时能快速定位

### 2.7 本地搜索

最后一步是添加站内搜索功能。安装 `hexo-generator-search` 插件后，构建时会生成 `search.xml` 索引文件，Cactus 主题内置的搜索组件读取这个文件实现即时搜索。

```bash
pnpm add hexo-generator-search
```

创建搜索页面：

```bash
pnpm hexo new page search
```

在页面 front-matter 中声明 `type: search`，主题会自动渲染搜索组件。导航菜单中添加 `search: /search/`，用户点击即可进入搜索页。

整个过程——安装插件、创建页面、修改配置、提交推送、CI 自动部署——从指令到上线，不超过两分钟。

## 三、技术细节的深度思考

### 3.1 Token 的安全模型

AI 控制 GitHub 账户，底层的认证机制是 Personal Access Token（PAT）。这个 Token 存储在本地 keyring 中（macOS 的安全密钥库），`gh` CLI 自动读取，无需每次输入密码。

当前的 Token scopes 是 `gist, read:org, repo, workflow`，这意味着 AI 能够：

- `repo`：读写所有公开和私有仓库的代码
- `workflow`：修改和触发 GitHub Actions 工作流
- `read:org`：读取组织信息
- `gist`：管理 Gist

注意，这个 Token **没有** `delete_repo` 权限。这是一种纵深防御——即使 AI 出了错，最坏情况下也不会删除仓库。Token 权限的最小化原则，在 AI 介入的场景下尤为重要。

### 3.2 冻结锁文件的意义

CI 中使用 `pnpm install --frozen-lockfile` 而非普通的 `pnpm install`，这不是随意的选择。`--frozen-lockfile` 要求 lockfile 与 `package.json` 完全一致，任何不一致都会导致安装失败而非"自动修复"。

这个机制的价值在于**确定性**。开发环境与 CI 环境安装完全相同版本的依赖，避免了"我本地能跑"的经典问题。前面提到的第一次 CI 失败，正是因为这个机制及时拦截了不一致的 lockfile，而不是在构建阶段产生难以调试的问题。

### 3.3 构建产物的路径处理

Hexo 默认生成的站点根路径是 `/`，但 project 类型的 Pages 站点部署在 `https://username.github.io/blog/` 下，根路径是 `/blog/`。如果不做处理，所有静态资源（CSS、JS、图片）的路径都会指向错误的地址，导致页面样式丢失。

幸运的是，Hexo 的 `url` 配置和 `configure-pages` Action 配合默契。`configure-pages` 会自动检测 Pages 站点的 base path 并注入到构建环境中，Hexo 根据 `url` 配置生成正确的相对路径。`url: https://xiaokang1993.github.io/blog` 让 Hexo 知道站点部署在 `/blog/` 子路径下，生成的链接和资源引用都会带上这个前缀。

### 3.4 并发控制

工作流中的 `concurrency` 配置看似简单，实则重要：

```yaml
concurrency:
  group: pages
  cancel-in-progress: true
```

`group: pages` 把同一组的运行串行化，`cancel-in-progress: true` 则取消正在运行的旧构建。想象一个场景：你连续推送了三次 commit，第一次触发的构建还没跑完。没有并发控制的话，三个构建会同时运行，争抢 Pages 的部署通道，最后部署的不一定是最新版本。有了并发控制，后两次推送会取消第一次的构建，只运行最新的，确保部署的始终是最新代码。

## 四、遇到的挑战与反思

### 4.1 网络不稳定的应对

实践过程中最大的挑战是网络。本地多次尝试 clone cactus 主题仓库，均以超时告终：

```
fatal: unable to access 'https://github.com/probberechts/hexo-theme-cactus.git/':
Failed to connect to github.com port 443 after 75003 ms: Couldn't connect to server
```

这个问题的处理过程值得复盘。AI 尝试了多种方案：

1. 直接 `git submodule add` → 超时
2. `git clone --depth 1`（浅克隆）→ 超时
3. `git://` 协议 → 超时
4. 镜像站（gitclone.com）→ 504 错误

四种方案全部失败后，AI 没有继续盲目重试，而是切换策略：放弃本地 clone，改为 CI 中 clone。这个决策的正确性在于，CI 运行在 GitHub 自己的数据中心，访问 github.com 不存在网络障碍。

**教训：当环境限制无法克服时，不要在受限环境中死磕，换一个不受限的环境解决问题。** CI 就是那个不受限的环境。

### 4.2 Lockfile 不一致

前文已述，这是一个典型的"操作顺序"错误。正确的顺序应该是：先修改 `package.json`（移除 landscape），再生成 lockfile。但实际操作中，lockfile 是在 landscape 还在依赖列表时生成的，之后移除 landscape 导致了不一致。

`--frozen-lockfile` 的严格性在这里发挥了价值——它在 CI 阶段拦截了问题，而不是让不一致的依赖树混入构建，产生更难调试的运行时错误。这个经历让我更加确信：**严格的校验机制看似"碍事"，实则是安全网。**

### 4.3 主题配置的覆盖机制

Cactus 主题有自己的 `_config.yml`，但因为主题文件不在本地（网络问题），无法直接编辑主题配置。解决方案是利用 Hexo 的 `theme_config` 覆盖机制：在站点 `_config.yml` 中通过 `theme_config` 字段覆盖主题的默认配置。

```yaml
theme_config:
  colorscheme: dark
  projects_url: https://github.com/xiaokang1993
  nav:
    home: /
    about: /about/
    articles: /archives/
    search: /search/
    projects: https://github.com/xiaokang1993
  social_links:
    - icon: github
      link: https://github.com/xiaokang1993
```

这种方式的好处是：配置与主题代码分离，升级主题时不会丢失自定义配置，也不需要修改主题文件（便于以 submodule 形式管理）。这是一个"关注点分离"的好实践。

## 五、安全与伦理的考量

### 5.1 最小权限原则

让 AI 控制 GitHub 账户，最核心的安全措施是 Token 的最小权限。当前的 Token 没有 `delete_repo`、没有 `admin:org`、没有 `user` scope。AI 能创建仓库、推送代码、配置 Pages，但不能删除仓库、不能管理组织成员、不能修改用户设置。

这种设计遵循了最小权限原则：**授予的权限刚好够完成任务，不多给一分。** 即使 AI 的行为偏离了预期，它能造成的破坏也被限制在权限范围内。

### 5.2 可审计性

AI 的每一步操作都有审计痕迹：

- Git commit message 记录了每次变更的意图和内容
- GitHub Actions 运行日志记录了 CI 的完整过程
- `gh api` 调用产生 API 请求记录
- `git log` 保留了完整的提交历史

这些痕迹意味着，任何操作都可以追溯。如果未来发现某个变更有问题，可以精确地找到它是在哪个 commit 中引入的、当时做了什么、为什么做。这种可审计性是 AI 介入生产环境的基础——**没有审计，就没有信任。**

### 5.3 人的监督角色

在整个过程中，人的角色从"执行者"转变为"监督者"和"决策者"。AI 负责执行，人负责：

- 确定目标（"新建 blog 仓库"）
- 审核关键决策（是否公开、用什么主题、选哪个包管理器）
- 叫停错误方向（"跳过 submodule"、"不用镜像"）
- 验证最终结果（站点是否可访问）

这个角色转变并不意味着人变得不重要了，恰恰相反，人的决策变得更加关键。当执行环节被自动化后，决策的质量决定了结果的上限。**AI 提升了执行的效率，但方向仍然由人掌舵。**

### 5.4 透明度与知情同意

AI 在操作 GitHub 账户时，保持了全程透明：每一步都说明意图、展示命令、报告结果。这种透明度让"知情同意"成为可能——我知道 AI 要做什么，我同意它做，我看到了做的结果。

如果 AI 在后台悄悄操作，不告知、不展示，即使结果完全正确，这种模式也是不可接受的。因为问题不在于"这次做对了没有"，而在于"如果下次做错了，我能否及时发现"。透明度是及时纠错的前提。

## 六、效率提升的量化思考

### 6.1 时间维度

让我们粗略量化一下这次实践节省的时间。

传统方式搭建一个完整的 GitHub Pages 博客（含 CI/CD 部署、搜索功能），一个有经验的开发者大概需要：

| 环节 | 传统耗时 | AI 介入后 |
|------|---------|----------|
| 创建仓库 | 2 分钟 | 5 秒 |
| 初始化项目 + 配置 | 15 分钟 | 2 分钟 |
| 编写 .gitignore + README | 10 分钟 | 1 分钟 |
| 编写 deploy.yml | 10 分钟 | 1 分钟 |
| 配置 GitHub Pages | 3 分钟 | 5 秒 |
| 调试 CI 失败（lockfile） | 15 分钟 | 2 分钟 |
| 添加搜索功能 | 8 分钟 | 1 分钟 |
| **合计** | **约 63 分钟** | **约 8 分钟** |

这只是一个粗略的估算，实际节省因人而异。但核心结论是清晰的：**AI 介入后，总耗时降低了一个数量级。**

更重要的是，节省的不仅仅是时间，还有注意力。传统方式下，你需要在整个过程中保持高度专注，任何一步走神都可能引入错误。AI 介入后，你只需要在关键节点做决策，中间环节可以完全交给 AI，你的注意力可以留给更有价值的事情。

### 6.2 错误维度

时间节省是显性的，错误减少是隐性的——但后者可能更有价值。

传统方式下，手动操作容易引入的错误包括：

- YAML 缩进错误（四空格 vs 两空格混用）
- 配置项拼写错误（`permalink` 写成 `permaLink`）
- 遗漏步骤（忘记配置 Pages、忘记 push）
- 版本不一致（lockfile 与 package.json 不匹配）

AI 在这些方面有明显优势：它不会"遗漏步骤"（因为它有完整的流程意识），不会"拼写错误"（除非理解有误），YAML 缩进通常正确（因为它理解结构而非凭感觉缩进）。当然，AI 也会犯错（前述的 lockfile 问题就是），但它犯错后的修复速度远快于人类——它能直接读取 CI 日志、定位问题、执行修复，而不需要人去打开浏览器、翻找日志、理解报错。

### 6.3 认知负载

还有一个容易被忽略的维度：认知负载。

搭建博客需要同时关注很多东西：Hexo 配置、主题配置、Git 操作、CI/CD 配置、GitHub Pages 设置、搜索插件配置。每一项都有自己的文档、自己的约定、自己的陷阱。把这些全部装进工作记忆里，对任何人都是负担。

AI 介入后，认知负载显著降低。你不需要记住 `configure-pages` Action 的参数名，不需要记住 `pnpm install --frozen-lockfile` 的确切写法，不需要记住 Pages API 的 endpoint。你只需要描述"我想做什么"，AI 负责把这些意图翻译成具体的工具调用。

这就像从汇编语言到高级语言的跨越——你不再需要关心寄存器分配和内存地址，只需要描述算法逻辑。AI 把"操作 GitHub"这件事的抽象层级提高了一层。

## 七、局限性与诚实的边界

在赞美了这么多之后，必须诚实地面对局限性。否则这篇文章就变成了自吹自擂的广告，而非真实的实践心得。

### 7.1 网络依赖

AI 的执行能力高度依赖网络。整个实践中最大的障碍就是网络不稳定——clone 主题失败、curl 验证超时、ls-remote 不通。AI 没有绕过网络限制的能力，它只能在网络可用时高效执行，网络不可用时同样束手无策。

这意味着 AI 介入的前提是基础设施可用。如果网络本身是瓶颈，AI 的效率提升会被网络延迟完全吞噬。

### 7.2 上下文窗口

AI 在执行多步骤任务时，需要维持对整个流程的上下文理解。随着步骤增多、文件增多、错误增多，上下文的复杂度也在增长。虽然现代 AI 的上下文窗口已经很大，但在极端复杂的任务中，仍然可能出现"忘记之前做过什么"或"忽略已有配置"的情况。

实践中，AI 在移除 landscape 依赖时，没有同步重新生成 lockfile，就是一个"忽略已有状态"的例子。虽然 `--frozen-lockfile` 在 CI 阶段拦截了这个问题，但这个失误本身说明了 AI 在多步骤任务中的一致性仍有提升空间。

### 7.3 主题理解的深度

AI 对 Cactus 主题的理解来自训练数据和实时检索。它能配置 `nav`、`social_links`、`colorscheme` 这些文档中明确记载的选项。但如果需要深度定制主题——修改 EJS 模板、自定义 CSS 变量、添加自定义页面布局——AI 的能力取决于它对主题源码的理解程度。

实践中，因为主题文件不在本地，AI 无法读取主题源码，深度定制无从谈起。即使主题在本地，深度定制也需要对 Hexo 的渲染机制、EJS 模板语法、Stylus CSS 预处理器有综合理解，这对 AI 来说是更高的挑战。

### 7.4 创造性工作的边界

AI 擅长执行明确的、结构化的任务——创建仓库、编写配置、修复错误。但在需要创造性判断的领域——选择什么主题、写什么文章、设计什么视觉风格——AI 的建议是通用的、安全的，但不一定是最好的。

博客的内容、风格、调性，最终应该由人决定。AI 可以帮你把想法变成现实，但"想法"本身，目前还是人的领域。

## 八、未来展望

### 8.1 从"指令驱动"到"意图驱动"

当前的交互模式是指令驱动的：用户说"新建仓库"，AI 执行。用户说"添加搜索"，AI 执行。每一步都需要用户明确发出指令。

未来的方向是意图驱动：用户说"我想搭一个技术博客，要有搜索、评论、标签云"，AI 自主规划完整方案，分解为多个步骤，依次执行，中间遇到问题自主解决，最终交付一个完整的站点。用户从"逐步指令"转变为"描述目标"和"审核方案"。

这次实践已经初现端倪——用户说"将本地目录 blog 初始化，修改相应文档和说明链接，添加 git 忽略，推送到远程 blog 仓库，并用 github pages 生成博客"，这是一个复合指令，包含了多个步骤。AI 自主分解为子任务、规划顺序、执行验证。虽然中间有纠偏（跳过 submodule、不用镜像），但整体流程是 AI 自主完成的。

### 8.2 Agent 生态的成熟

AI 控制 GitHub 账户，本质上是 AI Agent 的一种应用。Agent 不同于单轮对话——它有状态、有记忆、能自主决策、能调用外部工具。当前 Agent 生态正在快速发展，工具调用（Function Calling）、计算机使用（Computer Use）、代码执行（Code Interpreter）等能力日趋成熟。

未来，Agent 将能够处理更复杂的 DevOps 任务：从搭建博客到管理微服务、从配置 CI 到自动化事故响应。GitHub CLI 和 API 是 Agent 操控基础设施的"手"，而 AI 模型是"脑"。随着"手"越来越灵活、"脑"越来越聪明，Agent 能做的事情会指数级扩展。

### 8.3 人机协作的新范式

最终的目标不是 AI 替代人，而是人机协作达到新的效率高度。人负责战略决策和创造性工作，AI 负责战术执行和重复性工作。两者各自发挥优势，协作产出超越任何一方单独能达到的成果。

这次博客搭建就是一个微缩的人机协作样本：人决定了博客的名字、主题、功能需求；AI 负责把需求变成可运行的系统。人在关键节点纠偏（跳过 submodule、不用镜像），AI 在执行环节高效推进。最终产出一个完整、可用、持续部署的博客站点。

## 九、结语

这篇文章从一句"GitHub 新建 blog 仓库"开始，到一篇 5000 余字的心得文章结束。它的诞生本身就是 AI 控制 GitHub 账户的产物——仓库是 AI 创建的，配置是 AI 修改的，部署工作流是 AI 编写的，CI 是 AI 触发和监控的，甚至这篇文章也是 AI 撰写的，然后通过 AI 搭建的管道发布到 AI 部署的站点上。

这是不是有点魔幻？也许是。但这已经是 2026 年的日常。

我并不认为 AI 会取代开发者。开发者从来不仅仅是"写代码的人"——开发者是问题解决者、是系统设计者、是价值创造者。写代码只是手段，不是目的。当 AI 把"写代码"这件事的门槛降到接近零时，开发者的价值不降反升，因为"知道写什么代码"、"知道为什么写这些代码"、"知道这些代码服务什么目标"——这些判断力比执行力更稀缺、更珍贵。

让 AI 接管 GitHub 账户，不是放弃控制权，而是把控制权提升到更高的层级。你不再控制每一行命令，你控制的是方向和目标。AI 是你的手和眼，你仍然是大脑。

这次实践教会我的最重要的一课是：**工具的进步不是为了让人变得多余，而是为了让人有更多余裕去做那些真正需要人的事情。**

把重复的交给 AI，把创造性的留给自己。这，大概就是 2026 年开发者该有的姿态。

---

> 本文作者：小康  
> 本文链接：https://xiaokang1993.github.io/blog/2026/10/21/ai-control-github-account/  
> 版权声明：转载请注明出处
