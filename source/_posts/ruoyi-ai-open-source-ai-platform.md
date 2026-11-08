---
title: RuoYi AI 拆解：一个 Java 团队如何把 AI 助手平台做成开源产品
date: 2026-11-08 14:30:00
tags:
  - AI
  - RuoYi AI
  - RAG
  - 开源项目
  - Java
categories:
  - 技术随笔
---

## 一句话说明它是什么

RuoYi AI 是一个面向开发者的开源 AI 应用开发平台：Java 后端 + 管理端 + 用户端，三套代码分开维护，能力覆盖模型接入、知识库 RAG、智能体、工具与 MCP、可视化流程编排。

它不属于“又一个聊天套壳”。把文档从头翻一遍你会发现，作者写的不是营销页，而是排障手册——大量篇幅花在解释“为什么这个字段保存的是向量库类型而不是模型名”“为什么上传成功不等于解析成功”“为什么 Redis 密码留空字符串会让后端起不来”。这种文档风格，只有真正被自己坑过的人才写得出来。

- 文档：<https://doc.ruoyiai.chat/>
- 源码：<https://gitee.com/ageerle/ruoyi-ai>
- 在线体验：管理端 `129.226.199.247:25666`，用户端 `129.226.199.247:25137`，默认 `admin / admin123`
- 商业版演示：<https://web.ruoyiai.chat>

## 一、它由哪几块拼起来

很多人第一次看会在这里绕晕：后端仓库里有个目录叫 `ruoyi-admin`，另外还有一个独立的前端仓库也叫 `ruoyi-admin`。它们不是同一个东西。

| 仓库 | 职责 | 启动方式 |
| --- | --- | --- |
| `ruoyi-ai` | Java 后端，登录、数据、模型调用、解析、检索 | Maven 打包，`java -jar` |
| `ruoyi-admin` | 管理端前端，配模型、知识库、智能体 | `pnpm run dev:antd`，5666 端口 |
| `ruoyi-web` | 用户端前端，对话与应用市场 | `pnpm run dev`，5180 端口 |

后端本身是一套标准的 RuoYi 模块化结构：`ruoyi-common` 下面二十多个公共模块（安全、缓存、MyBatis、OSS、SSE、WebSocket、多租户、限流、脱敏、幂等……），`ruoyi-modules` 下是业务模块。真正跟 AI 相关的只有两块：

- `ruoyi-chat`——智能体、MCP、知识库、模型工厂、各厂商适配
- `ruoyi-aiflow` / `ruoyi-workflow`——流程编排与工作流

这个分层很务实。绝大多数“AI 平台”项目会把认证、租户、文件存储这些脏活写得很潦草，然后在 demo 里绕开。RuoYi AI 反过来：AI 只是长在成熟脚手架上的一个业务模块，权限、租户、字典、审计全都是现成的。

推荐环境写得很明确：JDK 21（源码目标 17）、MySQL 8.0、Redis 6.2+、Maven 3.9.x、Node 24 LTS、pnpm 10.14.0。pnpm 版本是锁死的，因为管理端仓库的 `packageManager` 字段会强制切版本——你在仓库外 `pnpm -v` 看到 11，进目录再敲一次就是 10.14.0，别慌。

## 二、模型接入：厂商编码才是真身

模型管理拆成两层：**厂商** 决定“用哪套接入代码 + 默认连哪个地址”，**模型** 决定“具体是哪个模型、什么用途、什么 Key”。

一个厂商可以有多个模型。`chat_provider` 存厂商，`chat_model` 存模型，但模型表里存的是 `providerCode` 字符串，不是厂商主键。原因在调用链上很直白：

```java
// 读取用户选择的模型
ChatModelVo chatModelVo =
    chatModelService.selectModelByName(chatRequest.getModel());

// 根据厂商编码选择对应的接入实现
AbstractChatService chatService =
    chatServiceFactory.getOriginalService(chatModelVo.getProviderCode());
```

所以厂商名称随便改，编码不能乱动。内置编码里能直接看到这个项目的野心和克制：`deepseek`、`ppio`、`openai`、`qianwen`、`zhipu`、`ollama`、`minimax`、`xiaomi`（小米 MiMo）、`atlas`、`custom_api`、`custom_anthropic`、`dify`、`coze`。

想加新厂商，实现 `AbstractChatService` 三个方法就够了：`getProviderName()` 返回编码，`buildStreamingChatModel()` 给流式回答，`buildChatModel()` 给 Agent 这类需要完整结果的场景。这是 LangChain4j 的模型抽象，不是自己造的轮子。

几个细节值得记住：

- 模型分类由 `chat_model_category` 字典维护。把对话模型塞进 `vector` 分类里，页面能选、调用会炸。
- 停用厂商会同时把它从模型表单的供应商选项里摘掉，用户端列表也不再返回。但停用不会中断已经在途的请求。
- 厂商地址只在创建模型时“复制”一次。之后改厂商地址，已有模型还在用老地址，得手动同步 `apiHost`。
- 选 `custom_api` / `custom_anthropic` 时表单会清空地址，要求每个模型自己填。

最后一条最容易踩：不是 bug，是设计——自定义协议本来就没有统一的默认地址。

## 三、RAG 链路：它诚实的地方在这

知识库文档是整个站里最长的一页，也是最值得读的一页。因为它把“哪些环节验证过、哪些没有”写在了明处，末尾还有一节 `验证说明`。

链路是标准的：上传 → 对象存储 → 解析 → 分段 → 向量化 → 写入向量库 + 片段表 → 检索 → 重排 → 拼进对话上下文。

**上传去重**用 SHA-256，按“知识库 ID + 文件摘要”判重。同一个文件改个名再传，仍然算重复。想重解析，用那一行的解析按钮，不要重复上传。这一条能省很多存储和向量库的脏数据。

**分段参数**在“知识库配置”页签，不在新增弹窗里。后端有硬校验：

```java
if (blockSize <= 0) {
    throw new ServiceException("文本块大小必须大于0");
}
if (overlap < 0 || overlap >= blockSize) {
    throw new ServiceException("重叠字符数必须大于等于0且小于文本块大小");
}
```

**一个容易搞反的字段**：`vectorModel` 存的是向量库类型（weaviate / milvus / qdrant），`embeddingModel` 才是向量模型名称。名字起得有歧义，但 API 里就是这样。

**向量模型不能混用**。同一知识库里的文档和查询必须是同一个向量模型。哪怕两个模型输出维度都是 384，混用之后的余弦相似度也是无意义的噪声。用 Milvus 时还要额外确认集合维度和模型实际输出一致——文档专门提了这一句，说明踩过。

**解析是异步的**：`upload()` 先落附件记录，再通过 Spring 代理调 `@Async("knowledgeParseExecutor")` 的 `parse()`。所以“上传成功”和“已解析”是两件事，中间有窗口期。解析完成的写入是有顺序的：先删旧向量，写新向量，成功了才替换片段记录，最后让检索缓存失效。这个顺序保证了失败时不会留下半截数据。

**混合检索 + 重排**是可选开关。文档建议先用纯向量检索跑通基础流程，再逐项打开。这个建议是对的——一开始就把三个变量同时打开，出问题你根本不知道是哪一层。

上传格式覆盖文本、Markdown、PDF、Word、Excel、CSV、JSON 和多种代码文件。文档里也明说了：**用户端普通聊天页目前没有直接选择知识库的控件**，只能通过智能体使用。这种“把边界写清楚”的做法，比含糊其辞强太多。

## 四、智能体、工具、MCP、Skills

**智能体**基于 LangChain4j Supervisor 编排。可以绑模型、绑知识库、绑工具，也能预检索知识库。管理端配好，用户端在应用市场里选。

**工具**分四类：Function Calling、内置工具（文件、命令、数据库）、本地 MCP、远程 MCP。MCP 那块文档写得像教程而不像手册——从协议概念讲到本地 / 远程 / 魔搭 ModelScope 三种实战，还结合后端源码讲怎么排障、怎么扩展。

**Skills** 是这套系统里最容易被误解的部分。它能管 docx、pdf、xlsx 之类的技能资源，但真正执行链路只接在 **Coding Harness** 上。普通智能体表单里能关联技能，关联完并不会进入执行流程——目前只是保存了关系。

这一点如果不说清楚，用户会以为是 bug。文档说了。这就是我前面讲的“诚实”。

**流程编排**是可视化画布 + SSE 执行：输入节点、模型节点、知识检索、条件分支、邮件、HTTP、网络搜索、图片生成。公开后的流程会出现在用户端应用市场。SSE 而不是 WebSocket 的选择也很合理——工作流是单向推送，没必要维持双向长连接。

## 五、开源版和商业版的分界线

这是全文最实在的一页。作者没有玩“开源版阉割得没法用”的套路，而是把差异做成了一张对照表：

| 能力 | 开源版 | 商业版 |
| --- | --- | --- |
| 文档解析 | 文本 / Word / Excel 基础解析 | 增加 PDF、Markdown、TXT、JSON、XML、PPTX、CSV |
| 文档图片 | 不支持 | 配视觉模型后可识别图片、图表、扫描页 |
| 原文定位 | 不支持 | PDF/DOCX 可定位页码并高亮区域 |
| 知识图谱 | 无 | 实体关系抽取，检索时关系扩展 |
| 分块与检索 | 基础分块 + 向量检索，可选重排 | 按格式分块、查询改写、混合检索、关系辅助、重排、检索过程可查 |
| 人机介入 | — | 对话中给选项，用户确认后继续 |
| 跨会话记忆 | 单会话 | 个人记忆可保存、编辑、删除 |
| 问数 | — | 自然语言查业务库，出图表、明细、统计口径、SQL |
| 文档编辑 | — | Word/Excel 创建与修订，PDF 批注 |

商业源码版 ¥15,800，含首年更新，第二年起 ¥3,980/年。基于 AgentScope Java，交付前后端源码。

我的看法：这条线划得合理。开源版是“能跑通全链路、能二次开发”的骨架，商业版卖的是解析器、图谱、文档编辑这些又脏又贵的工程件。对多数团队来说，开源版的 RAG 链路已经够搭内部知识库了；真要处理扫描件 PDF、要定位原文页码、要知识图谱，那部分工作量确实值这个价。

## 六、部署：真实踩坑记录

文档把部署分成两条路：源码启动（推荐首次）和 Docker 部署。源码路线以 Windows 11 + PowerShell 为例，Linux/macOS 只需要换路径。

后端编译就一条：

```bash
mvn -pl ruoyi-admin -am -Dmaven.test.skip=true package
java -Dfile.encoding=UTF-8 -jar ruoyi-admin/target/ruoyi-admin.jar
```

文档特意提醒：如果 `BUILD FAILURE`，先解决第一个 `[ERROR]`，别抱着没有产物的 `target/` 去 `java -jar`，那样只会得到“无法访问 JAR”。

基础服务用一份只含 MySQL / Redis / 可选 Weaviate / MinIO 的 Compose：

```bash
docker compose --env-file .dev/.env -f .dev/compose.yml config --quiet
docker compose --env-file .dev/.env -f .dev/compose.yml up -d mysql redis
docker compose --env-file .dev/.env -f .dev/compose.yml ps
docker exec ruoyi-local-redis redis-cli ping
```

然后是最有价值的一条坑：**Redis 没设密码时，配置里直接省略 `password` 字段，不要写 `password: ""`**。空字符串会触发一次空密码认证，后端直接起不来。这种错误自己排查，够你烧掉一个下午。

另外两条：

- 后端不会自动读取 `.dev/.env`。MySQL 密码要手动抄进 `application-local.yml`，两处得对得上。
- 改 `.env` 里的初始密码不会改已存在数据卷里的账号密码。数据卷还在，老密码就还在。

值得一提的是 2026-09 的更新日志：修了 Qdrant 在集合不存在时删除报错的问题（现在按已删除处理，但连接和权限错误仍然抛），修了高核数服务器上 `mainExecutor` 核心线程数超过最大线程数导致启动失败的问题，补了 Windows Docker 部署步骤。三条都是“只有在真实环境里跑过才会遇到”的 bug。

## 七、我的判断

**适合谁**：想搭内部知识库助手的企业 Java 团队，或者要给客户交付 AI 应用的软件服务商。后端是 Spring 生态，前端是两个独立 Vue 工程，任何熟悉 RuoYi 的人上手成本接近零。

**不适合谁**：想做极简个人 Chatbot 的人。这套东西的复杂度来自多租户、权限、模块化——你不需要这些，用 LangChain 几十行就够。

**它做对的三件事**：

1. **先有脚手架，再长 AI**。权限、租户、字典、审计这些地基没重造，省下的时间全投在 RAG 和编排上。
2. **文档敢写边界**。“这个功能要补厂商凭据才能真跑”“普通聊天页还不能直接选知识库”“Skills 在普通智能体里只保存关系不执行”——把未完成的部分写清楚，比假装完整更有说服力。
3. **开源和商业的界线是工程件，不是功能锁**。解析器、知识图谱、文档编辑，这三样都是硬投入。

**它需要你接受的三件事**：

1. 术语不统一。`vectorModel` 指向量库、`embeddingModel` 指模型，这种命名坑会持续存在。
2. 一次只验证一个变量。RAG 出问题时，分段、向量模型、阈值、重排、对话模型，五个嫌疑人，别同时改。
3. 它是平台，不是产品。默认界面、默认流程都是给二次开发准备的，不是开箱即用的成品。

## 上手路径

如果你想试，按这个顺序走，不要跳：

```bash
# 1. 拉三个仓库
git clone https://gitee.com/ageerle/ruoyi-ai.git
git clone https://gitee.com/ageerle/ruoyi-admin.git
git clone https://gitee.com/ageerle/ruoyi-web.git

# 2. 起 MySQL + Redis
cd ruoyi-ai && mkdir -p .dev
# 放入 compose.yml 和 .env（LOCAL_MYSQL_PASSWORD / LOCAL_MINIO_PASSWORD）
docker compose --env-file .dev/.env -f .dev/compose.yml up -d mysql redis

# 3. 编译并启动后端
mvn -pl ruoyi-admin -am -Dmaven.test.skip=true package
java -Dfile.encoding=UTF-8 -jar ruoyi-admin/target/ruoyi-admin.jar

# 4. 起管理端和用户端
cd ../ruoyi-admin && pnpm install && pnpm run dev:antd   # 5666
cd ../ruoyi-web   && pnpm install && pnpm run dev        # 5180
```

登录管理端，先配一个对话模型，确认能聊；再配向量模型，建知识库，用官方那份 `rag-verification-sample.md` 验证解析和检索；最后才把知识库挂到智能体上。

顺序反过来的人，通常会在“到底是检索没命中，还是模型没答对”这个问题上卡一整天。

---

这个项目最打动我的地方不是功能列表，而是文档里那种“我知道你会在这里出错”的语气。写文档的人把自己的失败经验一条条摊开来，这比任何 README 里的徽章都更能说明一个开源项目的成熟度。

仓库在 Gitee：<https://gitee.com/ageerle/ruoyi-ai>，文档在 <https://doc.ruoyiai.chat/>。想搭内部 AI 助手的，值得花一个下午读一遍。
