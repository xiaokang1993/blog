---
title: 华通JSON：一个纯离线 JSON 工具箱的五端架构与工程取舍
date: 2026-12-24 10:00:00
tags:
  - 华通JSON
  - wt-json
  - Vue 3
  - Tauri
  - 前端工程
categories:
  - 技术随笔
---

## 缘起：一个为「少点一次」而生的工具

华通JSON（wt-json）是一个纯离线的开发者 JSON 工具箱：格式化、校验、100+ 种文本格式的智能提取、语义级对比、四视图浏览与格式转换。它的官网写得很直白——「数据不出本机」。

工具本身是开发者「小富」写的。他在仓库的笔记里留下了这段自述，我认为它比任何功能列表都更能说明这个产品的气质：

> 我做这个工具的核心理念就是，能少打开一个就少打开一个，能用点击完成的绝不用复制，能用复制完成的绝不用手打，就是能偷懒就偷懒，能不点就不点，鼠标尽量少动，键盘尽量少敲。

这段话解释了三个设计决策。第一，**功能密度**是核心指标：一个工具能干的事，绝不该拆成三个。第二，**入口数量**是隐性成本：每多一个菜单、多一个页面，都是一次打断。第三，**键盘与鼠标的行程**是真实的体感成本，虽然它很难被量化。

也正因为是「给自己用」做的，这个项目的演进路径很有代表性：最初只是给媳妇用着方便，结果发布之后用户不断提 bug、不断提新需求，一点点打磨，才逐渐成型。这个工具的核心理念就是能少打开一个就少打开一个，在这个 AI 时代写一个工具很容易，不过想写一个好工具，还是一件很难的事情，需要考虑很多细节，比如性能、用户体验、安全性等等。

这篇文章会拆开它的源码看几个关键工程点：提取链为什么要按优先级收敛、大数精度是怎么保住 2^53 之外的位数的、四视图各自的渲染策略、语义对比的「语义」边界到底在哪、数据脱敏为什么要按值而不是按 key、i18n 为什么必须在构建期预编译，以及一份 `src/` 如何产出五个平台的产物。

- 官网：<https://xkloveme.jixiaokang.com/>
- 源码：<https://github.com/xkloveme/wt-json>
- 版本：v1.2.1（`package.json` 为准）

## 一、技术栈：刻意保持克制

先看依赖清单。`package.json` 的 `dependencies` 只有 20 个条目：

```json
{
  "@codemirror/commands": "^6.11.0",
  "@codemirror/lang-json": "^6.0.2",
  "@codemirror/language": "^6.12.4",
  "@codemirror/merge": "^6.12.2",
  "@codemirror/state": "^6.7.4",
  "@codemirror/theme-one-dark": "^6.1.3",
  "@codemirror/view": "^6.43.11",
  "@replit/codemirror-indentation-markers": "^6.5.3",
  "codemirror": "^6.0.2",
  "diff": "^9.0.0",
  "lucide-vue-next": "^1.0.0",
  "vue": "^3.5.34",
  "vue-i18n": "^11.4.23"
}
```

值得注意的两点。其一，**CodeMirror 6 被完整引入**——7 个 `@codemirror/*` 包加一个 `codemirror` 元包。这不是编辑器组件的选择，而是编辑体验的承诺：语法高亮、括号匹配、缩进标记、增量重解析，全部由它承担。对比功能也直接用了 `@codemirror/merge`。

其二，**没有状态管理库、没有路由库、没有 HTTP 客户端、没有构建时类型检查**。`devDependencies` 里只有 Vite、Tauri CLI、x18n 三样东西。

没有路由库这一点在 `App.vue` 里体现得很清楚。视图状态 `currentView` 是 `'editor' | 'test' | 'changelog'` 三个值，路由判断直接读 `window.location.pathname`，写入用 `window.history.pushState`：

```js
const currentView = ref('editor')
// popstate + pathname 判断：/test、/changelog
```

30 行代码解决了问题。引入 vue-router 反而要处理 history 模式、基路径、守卫，得不偿失。

没有 HTTP 客户端是**刻意的安全声明**。整个应用不发起任何数据请求——官网把「0 网络请求」作为四个数字指标之一列出来。这意味着没有 axios、没有 fetch 封装，也意味着审计面极小：你不需要审查它把数据发去了哪里，因为它没有出口。

## 二、智能提取：一条按优先级收敛的链

这是整个项目最有价值的部分，也是代码量最集中的部分。`src/utils/jsonExtractor.js` 有 2179 行，`src/utils/testFixtures.js` 有 809 行。提取器回归用例 111 例，其中 106 例在 Node 下通过——剩下 5 个是 XML 用例，依赖浏览器 `DOMParser`，Node 环境必然失败，不算回归。

### 2.1 主链结构

入口是 `extractJsonFromText(text)`，返回 `{ json, format }` 或抛异常。它的组织方式分五个阶段，严格串行，命中即返回：

```
阶段一：Pre-process（命令 / 包裹层）
  1. tryCurlCommandToJson    cURL 参数
  2. tryJwtToJson           JWT
  3. 剥离 Markdown 代码围栏
  4. tryMongoShellToJson    MongoDB Shell
  5. tryUnescapeJsonString  转义 JSON
  6. tryGoMapToJson         Go map
  7. JSONP 包裹剥离
  8. JSONC / JSON5 注释剥离

阶段二：整体解析
  9. tryParseCandidate      内部有 7 层降级链

阶段三：独立 Case
 10. tryChineseQuotesJson   中文引号 JSON
 11. tryProtobufTextToJson  Protobuf text 格式

阶段四：结构化文本格式
 12. tryXmlToJson
 13. tryYamlToJson
 14. tryPropertiesToJson
 15. tryTomlToJson
 16. tryQueryStringToJson
 17. tryCsvToJson
 18. tryMarkdownTableToJson
 19. tryPhpPrintRToJson
 20. tryRepairJson          JSON 自动修复

阶段五：括号兜底
 21. looksLikeJson 成立但全失败 → 直接抛错
 22. scanBraces 扫描最长可解析片段 → 格式「混合文本」
```

官网把这条链概括成一句话：**每一层只做一件事，命中即停。越靠前越「有结构」，越靠后越「靠猜」。**

这个优先级排序是有信息量的。MongoDB Shell 输出、JWT、cURL 命令，这些是**有明确语法的机器输出**，命中概率高、误判风险低，所以排在前面。XML、YAML、TOML、CSV 是**有正式规范的结构化格式**，需要真正的解析器。Query String、Markdown 表格、Properties、PHP `print_r` 是**弱结构文本**，容易产生歧义，所以排在后面。最后才轮到「括号扫描」这个纯猜的兜底。

### 2.2 命中即停的实现

主链本身是纯顺序 if-return：

```js
const curlResult = tryCurlCommandToJson(trimmed)
if (curlResult) return { json: curlResult, format: 'cURL 参数' }
```

关键在于**每个 `try*` 函数的契约**：成功返回 JSON 字符串，失败返回 `null`。降级不靠主链判断，靠函数内部静默吞掉异常：

```js
export const tryMongoShellToJson = (text) => {
  if (!mongoTypes.test(text)) return null   // 零耗时前置守卫
  try {
    // ...三级 try-catch 逐级降级
  } catch (e) {
    return null
  }
}
```

**前置守卫是这个设计的性能基础。** `mongoTypes` 是一组特定类型名的正则：

```js
const mongoTypes = /\b(ObjectId|ISODate|NumberLong|NumberInt|NumberDecimal|Decimal128|Int32|Long|BinData|Timestamp|DBRef|UUID|MinKey|MaxKey)\(/g
```

不匹配就立即 `return null`，连 try 都不进。`tryGoMapToJson` 先测 `prefixRe = /^map\[[^\]]+\][\w.{}[\]]*\s*\{/`；`tryCurlCommandToJson` 先测 `/^\s*curl(\.exe)?\s+/i`；`tryProtobufTextToJson` 先测消息块结构。这些守卫加起来不到一微秒，却决定了 111 个提取器不会互相拖慢。

**错误信息被完全吞掉。** 所有 `catch` 块一律 `catch (_) {}` 或 `catch (e) { return null }`，异常对象从不记录、不上抛。只有 `extractJsonFromText` 末尾的两处 `throw new Error(...)` 会把结论交给调用方——「JSON 解析失败。」和「未能在文本中定位到…」。

这是一个明确的取舍：**牺牲调试能力，换取容错能力。** 对于工具型应用这是对的。用户粘贴进来一段脏文本，得到「无法识别」就够了；内部哪一层失手对用户毫无意义。代价是开发者调试新格式支持时没有线索，只能靠 `test-fixtures.mjs` 的断言反推。

### 2.3 兜底括号扫描

`scanBraces(text, openChar, closeChar)` 是最后的通用手段。它不假设任何格式，只做一件事：找到所有配对的括号区间，逐个尝试解析，取能解析的。

```js
export const scanBraces = (text, openChar, closeChar) => {
  const matches = []
  let index = 0
  while (true) {
    const start = text.indexOf(openChar, index)
    if (start === -1) break
    let depth = 0, inString = false, inSingle = false, escape = false
    let end = -1
    for (let i = start; i < text.length; i++) {
      const ch = text[i]
      if (escape) { escape = false; continue }
      if (ch === '\\') { escape = true; continue }
      if (ch === '"' && !inSingle) { inString = !inString; continue }
      if (ch === "'" && !inString) { inSingle = !inSingle; continue }
      if (!inString && !inSingle) {
        if (ch === openChar) depth++
        else if (ch === closeChar) { depth--; if (depth === 0) { end = i; break } }
      }
    }
    if (end !== -1) {
      const candidate = text.substring(start, end + 1)
      let result = tryParseCandidate(candidate)
      if (!result) result = tryChineseQuotesJson(candidate)
      if (!result) result = tryProtobufTextToJson(candidate)
      if (result) matches.push({ text: result, start })
    }
    index = start + 1
  }
  return matches
}
```

四个状态位（`depth` / `inString` / `inSingle` / `escape`）解决了三个经典陷阱：**转义字符不被当作字符串边界**、**单引号内不算双引号字符串**、**字符串内的括号不参与深度计数**。

值得注意的细节是 `index = start + 1`。扫描完一个区间后，指针只前进一格而不是跳到 `end + 1`。这意味着它会**重新扫描已经匹配过的内部括号**——`{"a":{"b":1}}` 里，外层匹配成功后，`{"b":1}` 还会再被匹配一次。配合「取最长可解析片段」的策略，这是有意的：宁可多算，不可漏掉嵌套对象。代价是嵌套很深的大文本会有 O(n²) 风险。

还有一个刻意的保守设计：如果 `looksLikeJson` 判定输入「看起来是完整的 JSON」（以 `{`/`[` 开头、以 `}`/`]` 结尾）但所有层都失败了，它会**直接抛错，不做任何子对象提取**——`extractJsonFromText` 在这一分支不调用 `scanBraces`。

这个选择防的是篡改。用户贴了一段明确是 JSON 的文本却解析失败，说明内容本身有问题（截断、编码错误、非法字符）。此时去里面「捞」出一个子对象来展示，会掩盖原始错误，让用户以为数据是这个子对象。宁可不给，也不给错误的数据。

### 2.4 一个真实的提取案例

MongoDB Shell 这一层有代表性。日志里常见这样的输出：

```javascript
{ "_id" : ObjectId("5f9d4e3a9b1c8e2a1c4d5e6f"),
  "createdAt" : ISODate("2020-10-30T12:00:00Z"),
  "balance" : NumberLong("9007199254740993") }
```

`mongoTypes` 正则命中 `ObjectId(`、`ISODate(`、`NumberLong(` 三个特征子串，进入处理。各类型用独立替换规则：

```js
text.replace(/ObjectId\(['"]([^'"]*)['"]\)/g, '"$1"')
text.replace(/ISODate\(['"]([^'"]*)['"]\)/g, '"$1"')
text.replace(/NumberLong\(['"]?([^'")]+)['"]?\)/g, '$1')
```

结果：

```json
{ "_id": "5f9d4e3a9b1c8e2a1c4d5e6f",
  "createdAt": "2020-10-30T12:00:00Z",
  "balance": 9007199254740993 }
```

注意 `NumberLong` 那一条：**它的替换结果是裸数字，没有引号**。这是精度传递的关键——如果加了引号，`safeParse` 会把它当字符串，而下游的 `9007199254740993` 超过 `Number.MAX_SAFE_INTEGER`（`2^53 - 1` = 9007199254740991），原生 `JSON.parse` 会静默丢掉精度。这也是官网代码示例里 `// 未丢精度` 那行注释的由来。

### 2.5 `tryParseCandidate` 的七层降级

标准 JSON 这一层内部还有一条降级链，处理各种「看起来像 JSON 但不是」的变体：

1. **标准 JSON** — `safeParse`
2. **TypeScript 去类型** — `convertTypescriptToJs`，剥离 `export const x: Type =`、`as const`、`satisfies`、类型断言
3. **JS 无引号 key** — `convertJsUnquotedKeys`
4. **Java flat KV** — `tryJavaFlatKeyValueToJson`
5. **Java toString** — `convertJavaToJson`
6. **Python dict** — `convertPythonToJson`
7. **宽松 JS/TS** — `safeParseJsLike`

其中第 3 层的正则很直接：

```js
/([{,]\s*)([a-zA-Z_$][\w.$]*)\s*:/g  →  $1"$2":
```

而第 5 和第 6 层的消歧是这段代码最见功力的地方。`ClassName{...}` 既可能是 Java 的 `HashMap.toString()`，也可能是 Python 的 `SomeClass(...)`。作者的解法是给每个候选加**三重守卫**：

```js
// Java toString 处理 ClassName(...) 时的守卫
// 1. 有 '=' 且值像 Python 字面量 → 让位给 Python
// 2. 出现 '[..]=>' 结构 → 让位给 PHP
// 3. 值用单引号包裹 → 让位给 Python
```

HashMap 的判定用计数消歧：

```js
if (eqCount >= colonCount) { /* 视为 Java HashMap 的 toString */ }
```

Java 用 `key=value` 分隔，Python dict 用 `key: value` 分隔。`{a=1, b=2}` 里 `=` 多于 `:`，判为 Java；`{'a': 1}` 里 `:` 多于 `=`，判为 Python。这类规则无法靠单一正则解决，只能靠统计特征加守卫。

Python 转换的两个关键处理：

```js
// tuple → list（前瞻字母避免误伤函数调用）
text.replace(/(\w)\(([^()]*)\)/g, '$1[$2]')
// 字面量
text.replace(/\bTrue\b/g, 'true')
    .replace(/\bFalse\b/g, 'false')
    .replace(/\bNone\b/g, 'null')
```

`hasPythonTraits` 要求输入里有单引号或 Python 关键字，否则 `return null` 交给 JS 处理——因为 `{a: 1}` 在 JS 里合法，抢过来反而错了。

## 三、大数精度：绕过 2^53 的三种手段

`src/utils/jsonBigInt.js` 有 214 行，是整个项目里我最喜欢的文件。原生 `JSON.parse` 有个不可回避的问题：JSON 数字用 IEEE 754 双精度表示，`Number.MAX_SAFE_INTEGER` 只有 9007199254740991。超过这个值，`9007199254740993` 会被读成 `9007199254740992`。**静默出错，不抛异常**，这是它危险的地方。

官网把这一点列为四个数字指标之外的技术特性：**绕开原生 `JSON.parse` 的 2^53 限制，超大整数与高精度小数原样展示、原样回写。**

实现分三步。

**第一步：`protectBigInts` 预处理。** 在 `JSON.parse` 之前，用占位标记包裹所有大整数：

```js
// 8. 纯大整数（>= 16 位）：转为 BigInt 原生类型
// 10. 高精度小数：转为 LosslessNumber 占位
```

阈值定在 16 位，是因为 15 位数最大是 999999999999999，仍在安全范围内；16 位起才可能越界。这样只有真正需要处理的数字才付出代价。

**第二步：JSON.parse + reviver 精确还原。**

```js
export const safeParse = (jsonStr) => {
  const protectedStr = protectBigInts(jsonStr)
  const obj = JSON.parse(protectedStr, (key, value) => {
    if (typeof value === 'string' && value.startsWith(BIGINT_TAG)) {
      return BigInt(value.slice(BIGINT_TAG.length))
    }
    if (typeof value === 'string' && value.startsWith(LOSSLESS_TAG)) {
      return restoreLosslessNumber(value.slice(LOSSLESS_TAG.length))
    }
    return value
  })
  return obj
}
```

**第三步：`safeStringify` 反向还原。** 序列化时把 BigInt 与 LosslessNumber 还原为**无引号的裸数字**，保持原生数字格式：

```js
export const safeStringify = (obj, replacer, space) => {
  // BigInt / LosslessNumber → 裸数字（不加引号）
}
```

第三步是关键设计。很多人实现大数支持时会把大数存成字符串，这样精度保住了，但**类型变了**——`{"id": 9007199254740993}` 变成 `{"id": "9007199254740993"}`。下游做数值比较、做算术、做 `typeof` 判断全部失效。这里选择还原成裸数字，让序列化产物和输入在语义上等价。

这个取舍有代价：`JSON.stringify(BigInt)` 原生会抛 `TypeError: Do not know how to serialize a BigInt`。所以必须在序列化前替换掉所有 BigInt 实例。项目用 `replacer` 参数在遍历时完成替换。

`jsonExtractor.js` 的 MongoDB 处理之所以要保留裸数字，正是为了让 `safeParse` 能把 `NumberLong("9007199254740993")` 的字符串形式转成数值类型而非字符串。

## 四、四视图：三种渲染策略应对同一份数据

四种视图共享同一个已解析对象，但渲染策略完全不同。

| 视图 | 组件 | 行数 | 策略 |
| --- | --- | --- | --- |
| 代码 | `CodeMirrorEditor.vue` | 1950 | 增量重解析 + 视口渲染 |
| 树形 | `JsonVirtualTreeView.vue` | 2169 | 扁平化 + 虚拟滚动 |
| 表格 | `JsonTableView.vue` | 3464 | 虚拟滚动 + 内层独立滚动 |
| 拓扑 | `JsonGraphView.vue` | 3857 | 全量渲染 + 布局计算 |

### 4.1 树视图：扁平化 + 二分查找定位

树视图不能直接用递归渲染，否则几万个节点会生成同样多的 DOM。它的做法是先把树**压平成行数组** `flatRows`，再对 `flatRows` 做虚拟滚动。

核心是两个计算属性 `startIndex` 和 `endIndex`，配合二分查找：

```js
const startIndex = computed(() => {
  const target = scrollTop.value
  // 二分查找：找到第一个「累积高度 >= target」的行
  let low = 0, high = flatRows.value.length - 1
  while (low < high) {
    const mid = (low + high) >> 1
    if (cumulativeHeight(mid) < target) low = mid + 1
    else high = mid
  }
  return low
})
```

`endIndex` 同样以 `scrollTop.value + viewportHeight.value` 为目标。`visibleRows` 只 slice 出 `[startIndex, endIndex)` 这段：

```js
const visibleRows = computed(() => {
  const slice = flatRows.value.slice(startIndex.value, endIndex.value)
  return slice.map((row, idx) => {
    const actualIdx = startIndex.value + idx
    // ...
  })
})
```

树的高度不是固定的，所以有个 `rowHeightMap` 记录每行的实际高度，配合 `rowHeightVersion` 做响应式失效。展开/折叠节点时更新对应行的测量高度，版本递增触发重算。

### 4.2 表格视图：双层虚拟滚动

表格视图复杂在它有**嵌套结构**。外层是数组元素，内层可能是对象。所以有两套独立滚动：外层 `VIRTUAL_THRESHOLD = 60`（超过 60 行开启虚拟滚动），内层 `INNER_VIRTUAL_THRESHOLD = 50`、`INNER_BUFFER = 12`：

```js
const VIRTUAL_THRESHOLD = 60   // 超过 60 行自动开启虚拟滚动
const VIRTUAL_BUFFER = 15      // 上下各缓冲 15 行，避免高速滚动出现白屏
const INNER_VIRTUAL_THRESHOLD = 50
const INNER_BUFFER = 12
```

内层用 `innerScrollMap` 按路径键记录独立的滚动位置和视口高度：

```js
innerScrollMap[pathKey] = { scrollTop: top, viewportHeight: e.target.clientHeight || 500 }
const startIdx = Math.max(0, Math.floor(top / rowH) - INNER_BUFFER)
const rows = arr.slice(startIdx, endIdx)
```

这个设计解决了实际问题：表格里嵌套数组元素时，内层内容可能几千行，如果跟着外层一起滚，用户会迷失位置。内层独立滚动让每一层的浏览范围可控。

缩放有明确的范围约束：

```js
const DEFAULT_TABLE_SCALE = 0.9   // 默认比例 90%
const MIN_TABLE_SCALE = 0.6
const MAX_TABLE_SCALE = 1.5
```

### 4.3 拓扑视图：为什么不做虚拟滚动

`JsonGraphView.vue` 是全项目最大的组件（3857 行），也是唯一**不做虚拟滚动**的视图。原因是图布局需要全量坐标计算：节点位置由力导向或分层布局算法决定，只渲染可视区内的节点会让布局错乱（边连不上、层次塌陷）。

它的策略是：**全量渲染 + 布局计算 + 节点数上限保护**。节点过多时降级显示提示，而不是硬撑。这个取舍是对的——拓扑视图本来就是给中小规模结构看关系的，硬要支持 80 万行没有意义，那是树视图和表格视图的职责。

### 4.4 代码视图：CodeMirror 6 的增量能力

代码视图用 CodeMirror 6，优势是**增量重解析**：改一行只重新解析受影响的文档区间，不是全文重解析。`@codemirror/language` 的 `Language` 实例配合 Lezer 解析器做语法树缓存，配合 `@codemirror/view` 的视口渲染（只渲染 DOM 可见区的行）实现大文件流畅。

`@codemirror/indentation-markers` 提供缩进标记，`@codemirror/merge` 支撑对比视图的字符级差异高亮。

### 4.5 字段类型识别与预览

`src/utils/advancedDetectors.js` 提供 `detectValueType(val)`，被 `JsonTreeNode.vue` 调用决定节点图标：

```js
export function detectValueType(val) {
  const isNumber = typeof val === 'number' && Number.isFinite(val)
  if (!isNumber && typeof val !== 'string') return null
  const text = isNumber ? String(val) : val.trim()
  if (!text || text.length > 200) return null
  if (EMAIL_RE.test(text)) return { type: 'email', label: '邮箱' }
  const compactPhone = text.replace(/[\s()-]/g, '')
  if (MOBILE_RE.test(compactPhone)) return { type: 'phone', label: '手机号' }
  if (LANDLINE_RE.test(compactPhone)) return { type: 'phone', label: '电话' }
  if (ADDRESS_EN_RE.test(text)) return { type: 'address', label: '地址' }
  for (const re of ADDRESS_PATTERNS) { if (re.test(text)) return { type: 'address', label: '地址' } }
  return null
}
```

三个设计细节值得注意。

**长度门槛 200**：`text.length > 200` 直接返回 null。一段 300 字符的文本可能是合法的 JSON 字符串，但它不是「邮箱」或「手机号」。这个上限避免了把长文本误判成短类型。

**手机号先做紧凑化**：`text.replace(/[\s()-]/g, '')` 去掉空格、括号、横杠，让 `+86 138-0000-8000` 这类格式也能命中。紧凑化必须在测试前进行，否则正则要覆盖所有分隔符组合。

**地址判定分中英两套**：英文用 `ADDRESS_EN_RE` 匹配 `123 Main Street` 这类模式（含 20+ 种英美街道后缀），中文用 `ADDRESS_PATTERNS` 数组里的 12 条正则覆盖省市区街道门牌。中文地址形态变化太大，单条正则无法覆盖，只能用多条模式叠加。

预览模式（`SmartDataPopover.vue`、`ImagePreviewPopover.vue`）覆盖了官网列出的所有场景：时间戳多格式展示、unicode、图片、链接跳转、音视频、嵌套 JSON、base64、HTML 代码块、Cron 表达式自然语言翻译、URL 编码字符串。

其中 Cron 翻译和 JWT 解码有个巧妙的处理：

```js
// cron/JWT 动态文案在渲染时从结构化字段（parts / payload.exp）现算，
// 切换语言后立即刷新
const cronText = computed(() => { /* 从 props.data.parts 计算 */ })
```

翻译结果不是存好的字符串，而是**渲染时从结构化字段现算**。这样切换语言时文案立即刷新，不需要重新解析。JWT 的 `payload.exp` 也是运行时算出的动态文案。

HTML 预览走 `sanitizeHtml(data.raw)`，再 `v-html` 渲染——先清洗后插入，这是基本的安全实践。嵌套 JSON 的判定是检测字符串是否可被 `safeParse` 解析，能解析就提供「展开为对象」的入口。

## 五、语义对比：它的「语义」边界在哪

`JsonComparer.vue` 有 3993 行，是第二大组件。官网宣称「语义级对比：先做结构正规化再 Diff，字符级差异高亮，忽略空格与 Key 顺序」。这个描述**大体准确但不完整**——它的「语义」范围是明确限定的，超出范围的部分仍然报差异。

### 5.1 两条独立的正规化路径

关键发现：**「结构正规化」不是统一管线，而是分两条独立路径，一条改文本，一条只改比较逻辑。**

**路径 A：写回文本的格式化/排序（真正改变编辑器内容）**

`sortJSONKeys(obj, desc)` 递归排序对象的 key：

```js
export const sortJSONKeys = (obj, desc) => {
  if (Array.isArray(obj)) return obj.map(item => sortJSONKeys(item, desc))
  if (obj === null || typeof obj !== 'object') return obj
  const sorted = {}
  const keys = Object.keys(obj).sort()
  if (desc) keys.reverse()
  for (const key of keys) sorted[key] = sortJSONKeys(obj[key], desc)
  return sorted
}
```

注意 `Array.isArray(obj)` 分支排在对象判断之前，且数组只是 `map` 递归元素——**数组顺序永远不重排**。这是正确的语义选择：数组顺序通常有业务含义（时间序、优先级序），重排会改变数据含义。

排序后 `safeStringify(parsed, null, 2)` 写回 `tab.leftText/rightText`。diff 看到的已经是同序文本，行级 diff 自然不因键序产生差异。

关闭排序时用快照还原原文：

```js
if (sortKeys.value && !tab._unsortedLeftText) {
  tab._unsortedLeftText = tab.leftText   // 存快照
}
```

`tab._unsortedLeftText` / `tab._unsortedRightText` 是私有快照字段，排序开关切换时用来还原。

**路径 B：diff 期比较选项（不改变文本，只影响「算不算差异」）**

```js
const options = { ignoreCase: caseInsensitive.value, ignoreWhitespace: ignoreWhitespace.value }
diff.diffLines(leftText, rightText, options)
```

`ignoreWhitespace` **完全由 jsdiff 库选项实现**，项目没有自研空白归一化。默认 `true`，持久化键 `ej_global_ignore_whitespace`。`caseInsensitive` 是组件内部 `ref(false)`，不持久化、不在 `App.vue` 提供。

### 5.2 明确不做的事

这一点必须说清楚，否则使用者会误判：

| 能力 | 是否支持 | 说明 |
| --- | --- | --- |
| 忽略空白 | ✅ | jsdiff `ignoreWhitespace` 选项 |
| 忽略大小写 | ✅ | 可选，默认关 |
| 忽略 Key 顺序 | ✅ | 路径 A 的排序，改写文本实现 |
| **类型归一**（`1` vs `"1"`） | ❌ | 仍报差异 |
| **数组顺序归一**（`[1,2]` vs `[2,1]`） | ❌ | 仍报差异 |
| 数字精度归一 | ❌ | 依赖 `safeParse` 先保住精度 |

所以「语义对比」的准确范围是：**空白 + 可选大小写 + 可选键排序**。数组顺序和类型这两个最容易被误会的维度，项目明确不支持——这是合理的选择，因为强行支持会引入「什么才算相等」的无穷争论。

### 5.3 字符级 diff 与行配对

diff 库是 `diff@^9.0.0`（npm 上的 jsdiff）。行级用 `diffLines`，字符级用 `diffChars`。

难点在**行配对**：`diffLines` 返回的是 removed/added 交错的 chunk 流，需要把块内的行两两配对成 `modified`。`calculateLineSimilarity(a, b)` 做这件事：

```js
const calculateLineSimilarity = (a, b) => {
  const parts = diff.diffChars(a, b)
  let score = 0, total = 0
  for (const part of parts) {
    total += part.value.length
    if (!part.added && !part.removed) {
      score += part.value.length * (STRUCTURAL_RE.test(part.value) ? 0.5 : 1.5)
    }
  }
  return total ? score / total : 0
}
```

**结构字符权重 0.5、有效字符权重 1.5**。这个加权很关键：`{`、`}`、`,`、`:` 这类结构字符即使不同，语义差异也小；而字母数字不同才是实质性改动。不加权的话，两个长度相同的行相似度恒为 1.0（因为字符数相同），配对就失效了。

阈值 `sim >= 0.4` 才判为 `modified`，否则视为独立的删除/新增。不等时还做前瞻搜索（`for d = 1..4` 的三角搜索）找最近的相似配对。

字符级着色走三层正则分派：

```js
// 1. 键值行：拆分 key 与 value，各自 diffChars
const m = lLine.match(/^(\s*)"((?:[^"\\]|\\.)*)"(\s*:\s*)(.*)$/)
// 2. 字符串字面量：只 diff 引号内内容
const m2 = lLine.match(/^"((?:[^"\\]|\\.)*)"(,?)$/)
// 3. 兜底：整行 diffChars
```

第 2 层的设计有讲究：**只 diff 引号内内容，保留引号与尾逗号不被拆分**。这样 `{"name":"张三"}` 和 `{"name":"李四"}` 只会高亮 `张三`→`李四`，不会把 `{`、`"`、`:` 也卷进差异高亮，保证 JSON 语法着色不被破坏。

渲染用 CSS 类区分：`added` / `removed` / `modified` / `empty`，字符 span 用 `.word-added` / `.word-removed`。`stats` computed 统计 additions / deletions / modifications。

## 六、数据脱敏：按值形态匹配，而非按 key 名

`src/utils/dataMasker.js` 有 383 行。它的设计思路和多数脱敏工具不同——**内置规则全部基于值的形态匹配，与字段名无关**。

```js
const PHONE_RE = /(?<!\d)(1[3-9]\d)(\d{4})(\d{4})(?!\d)/g
const ID_CARD_RE = /(?<!\d)([1-9]\d{5})(?:\d{4}|\d{8})(\d{3}[\dXx])(?!\w)/g
const EMAIL_RE = /(?<![a-zA-Z0-9_.+:/])([a-zA-Z0-9_.+-]+)(@...)/g
const BANK_CARD_RE = /(?<!\d)(\d{15,19})(?!\d)/g
const LICENSE_PLATE_RE = /([京津沪渝冀豫云辽黑湘皖鲁新苏浙赣鄂桂甘晋蒙陕吉闽贵粤青藏川宁琼][A-HJ-NP-Z]...)/g
const USCI_RE = /(?<![A-Z0-9])([159Y][1239]\d{6}...)/g
const PASSPORT_RE = /(?<![A-Z0-9])([GEPDSC]\d{8}|EA\d{7})(?!\w)/g
```

十种内置规则，每种一个开关，默认全 `true`：

| 规则 | 常量 | 开关 |
| --- | --- | --- |
| 手机号 11 位 | `PHONE_RE` | `maskPhone` |
| 身份证 15/18 位 | `ID_CARD_RE` | `maskIdCard` |
| 邮箱（需合法 TLD） | `EMAIL_RE` | `maskEmail` |
| IPv4 / IPv6 | `IPV4_RE` / `IPV6_RE` | `maskIp` |
| 银行卡 15–19 位 | `BANK_CARD_RE` | `maskBankCard` |
| 车牌（含新能源） | `LICENSE_PLATE_RE` | `maskLicensePlate` |
| 统一社会信用代码 18 位 | `USCI_RE` | `maskUsci` |
| 中国护照 | `PASSPORT_RE` | `maskPassport` |
| 数据库连接串 | `URI_DSN_RE` 等 4 条 | `maskDsn` |
| 自定义字段 | — | `customKeys` |

**为什么按值不按 key？** 因为字段命名太不可靠了。同一份数据，手机号可能在 `phone`、`mobile`、`tel`、`contactNumber`、`user_contact` 里。按 key 匹配必然漏。按值形态匹配则不管字段叫什么，只要长得像手机号就脱敏。

代价是**误脱敏**：一个合法的 18 位订单号会被当身份证脱敏。项目用前后断言（`(?<!\d)` / `(?!\d)`、`(?!\w)`）收紧匹配，降低误报，但无法根除。这是这个设计的固有 tradeoff。

**唯一按 key 命中的是用户的 `customKeys`**：大小写不敏感，`Set` 化后 `toLowerCase()` 比较，默认空数组。

**数值走区间快筛**，避免全量字符串化：

```js
// maskPhone 候选：13000000000..19999999999
// maskBankCard 候选：1e14..~1e19
// 命中后才 String() + 正则 + Luhn
```

银行卡还有 `isValidBankCard()` 做 Luhn 校验，不只是长度匹配。这个细节很关键——15–19 位的纯长度匹配会把很多非卡号数字误命中，Luhn 校验能筛掉大部分。

**DSN 脱敏是亮点。** 四种连接串格式分别有正则：

```js
const URI_DSN_RE = /((?:jdbc:)?[a-zA-Z0-9_+.-]+:\/\/)([^:\s\/]*):([^@\s\/]+)@/g
const GO_DSN_RE   = /(?<![a-zA-Z0-9_.+])([a-zA-Z0-9_.-]+):([^@\s\/]+)(@(tcp|unix)?...)/g
const KV_DSN_RE   = /(?<=\b(?:password|pwd|secret)\s*=\s*)([^;,\s"']+)/gi
const ORACLE_DSN_RE = /((?:jdbc:oracle:...)?...)([A-Za-z0-9_.-])\/([^@\s\/]+)@.../g
```

覆盖 JDBC URI、Go 的 `user:pass@tcp(...)`、KV 形式的 `password=xxx`、Oracle 的 `user/pass@host`。这是从真实排障经验里长出来的规则——开发者经常把连接串打进日志或 JSON 里。

脱敏策略是**严格 1:1 等长**：`maskMiddle(str)` 保持总位数不变，只隐藏中间字符。URL 还有专门的 `maskUrl` 保留协议与域名，对路径等长脱敏。等长设计让脱敏后的文本在等宽字体下对齐不变，树视图的行高不会跳动。

## 七、持久化：向后兼容是硬约束

项目从旧产品名 `easy_json` 迁移而来，所有持久化标识符都故意保留旧名，改了就丢用户配置和历史。

### 7.1 两套存储

**localStorage**：26 个 `ej_*` 键。`ej_theme_mode`、`ej_theme_variant`、`ej_tab`、`ej_fmt_tabs`、`ej_fmt_active`、`ej_global_sort_keys`、`ej_global_ignore_whitespace`、`ej_mask_options`、`ej_lang` 等。前缀统一 `ej_`。

**IndexedDB**：

```js
const DB_NAME = 'easy_json_history_db'     // 旧名，不改
const DB_VERSION = 1
const STORE_NAME = 'history_records'
db.createObjectStore(STORE_NAME, { keyPath: 'id' })
```

`easy_json_history_db` / `history_records` 这两个名字是从旧产品继承的，README 明确写了：**改名会丢历史**。

### 7.2 历史记录结构

每条记录带 `versionNumber`，从已有记录的最大值 +1：

```js
let maxVersion = 0
for (const item of items) {
  if (typeof item.versionNumber === 'number' && item.versionNumber > maxVersion) {
    maxVersion = item.versionNumber
  }
}
const versionNumber = maxVersion + 1
```

标题生成有 `DEFAULT_RECORD_TITLE_KEYS` 一组优先 key（如 `name`、`title`、`id`），用来给历史记录自动生成可读标题。

### 7.3 多 Tab 状态持久化

编辑器用 `ej_fmt_tabs` / `ej_fmt_active` 做 Tab 持久化。刷新页面后 Tab 列表和激活项都恢复。这与「多 Tab 并行工作」功能配套——官网把它列为独立功能项。

## 八、i18n：为什么必须构建期预编译

项目支持中英双语，默认中文（`localStorage.ej_lang`）。语言包有 15 个命名空间，两套 locale 各 15 个文件：

```
zh-CN/  en-US/
  app.json      app.json
  changelog.json changelog.json
  common.json   common.json
  compare.json  compare.json
  editor.json   editor.json
  extract.json  extract.json
  formatter.json formatter.json
  graph.json    graph.json
  history.json  history.json
  import.json   import.json
  mask.json     mask.json
  popover.json  popover.json
  table.json    table.json
  test.json     test.json
  tree.json     tree.json
```

共 78 KB（zh-CN 38.5 KB + en-US 39.1 KB）。**目录名 = locale，文件名 = key 第一段**，由 `src/i18n/index.js` 的 `import.meta.glob` 归并。

### 8.1 核心约束：必须构建期预编译

这是整个 i18n 设计里最重要的一条，来自仓库的明确警告：

> 必须构建期预编译：`@intlify/unplugin-vue-i18n` + `__INTLIFY_DROP_MESSAGE_COMPILER__: true`。Chrome MV3 默认 CSP 禁 `eval`/`new Function`，走运行时编译会让插件端文案退化成 key（web/dev 端测不出来）。

技术原因在这里。`vue-i18n` 的消息默认是**编译成 JavaScript 函数**的——把 `{ "greeting": "你好 {name}" }` 编译成 `function (ctx) { return '你好 ' + ctx.name }`。这个编译过程用了 `new Function`。

Web 端没有 CSP 限制，运行时编译没问题。**但 Chrome MV3 扩展的 CSP 默认禁止 `eval` 和 `new Function`**。如果走运行时编译，函数无法创建，vue-i18n 会退化成直接返回 key 本身——界面上出现 `popover.cronTranslation` 而不是「定时任务翻译」。

web 端测不出来，因为 web 端能正常编译。这个 bug 只在插件端暴露，是典型的「多端差异导致的隐蔽缺陷」。解决方式是把消息编译提前到构建期：

```js
// vite.config.js
__INTLIFY_DROP_MESSAGE_COMPILER__: true
```

构建时把消息编译成纯数据结构（AST 或预渲染函数体），运行时不再需要 `new Function`。

### 8.2 key 的管理规则

- key 一律**静态字面量**，禁止拼接。动态场景用「存 key + params，渲染时 `t()`」
- `src/utils/*` 的 `format` / `label` / `type` 是**数据标识符**（`test-fixtures.mjs` 用它断言、UI 用它做分支），一律不翻译；展示层用 `src/i18n/labels.js` 查表
- 不翻译：更新日志正文、代码注释、`testFixtures.js`、死代码 `HelloWorld.vue` / `test.vue`、`manifest.json` / 后台右键菜单 / uTools `cmds` / VS Code `displayName`、CodeMirror 内置 UI
- 用户数据与编辑器内容容器加 `data-i18n-skip`，避免英文模式下的中文扫描误判

`data-i18n-skip` 这个细节很实用。扫描式 i18n 工具（扫描源码里的中文字符串自动提取）会误把用户数据里的中文当成待翻译文案。加跳过属性是最直接的隔离手段。

`common.json` 只放真实在用的高频词，新词先写自己命名空间——避免所有词条堆在一个文件里造成合并冲突。

### 8.3 校验脚本

```json
"i18n:check":  "x18n check",
"i18n:verify": "node scripts/check-i18n-keys.mjs"
```

`i18n:check` 走 x18n 查缺译/未用/占位符不一致；`i18n:verify` 反向校验 `t()` 调用与描述符 key 是否都能在 zh-CN / en-US 两套语言包解析。x18n 需要 node >= 22。

两个方向都校验，是因为这两类错误都会发生：key 有但代码没调用（冗余），代码调用了但 key 缺失（运行时显示 key）。占位符不一致（代码里 `{name}` 但语言包里 `{username}`）是第三类，同样会让渲染出 bug。

## 九、主题：8 个色块必须同步

这是 AGENTS.md 里明确警告的坑，原文：

> 所有界面颜色必须走 CSS 变量，禁止在组件里硬编码十六进制。变量集中在 `src/style.css`：`:root`（默认浅色）、`:root.dark-mode`（默认深色）、6 个 `:root[data-theme-variant="…"]`（墨观配色，自带深浅）。**新增或修改界面颜色变量时，必须同时补默认浅色、默认深色和全部 6 个变体块**，否则切到非默认配色时该处不跟随（表头/选择格曾因此不生效）。

所以是 **2 + 6 = 8 个色块**。默认浅色、默认深色各一个，外加 6 套墨观变体，每套变体自带深浅。

配套的两个约束：

- `src/theme/inkstone.js` 只是主题元数据（id/深浅/配对/光标色），**CSS 变量才是真源**
- 配色主题不管语法高亮色，`--json-*` 由 `one-dark-syntax` class 与 Premium 风格独立控制

`src/components/` 下能看到 `ElectricBorder.vue`（electric border 效果）、`SplashCursor.vue`、`ClickSpark.vue` 这类纯视觉组件，说明主题系统不只是换色，还有光标轨迹和点击反馈的视觉层。

深色覆盖的写法也有限定：

```css
/* 组件 <style scoped> 里 */
:global(.dark-mode) .selector { ... }
```

变体选择器在 `style.css` 中必须**位于 `.dark-mode` 之后**，靠顺序覆盖，不能移到前面。这是 CSS 层叠的规则——同特异性下后者胜出。

这个坑的本质是**维护性问题，不是正确性问题**。漏掉一个色块不会报错，只会在切换到某套配色时某处颜色不跟随。用户遇到会以为是 bug，实际是维护遗漏。所以 AGENTS.md 把它写成了硬约束。

## 十、五端一份源码

这是官网的核心卖点：**一份 `src/`，五端产物，行为一致。**

| 端 | 构建命令 | 产物 |
| --- | --- | --- |
| Web | `npm run build` | `dist/` |
| Chrome / Edge 插件 | `npm run build:plugin` | `wt-json-plugin/` |
| Tauri 桌面端 | `npm run tauri:build` / `build:win` / `build:mac:universal` | 安装包 |
| uTools | `npm run build:utools` | `utools-dist/` |
| VS Code | `npm run build:vscode` | `vscode-dist/` |
| 全部 | `npm run build:all` | — |

### 10.1 桌面端：Tauri 2 与 5173 的耦合

Tauri 配置里有个明确约束：

```json
{
  "version": "../package.json",
  "frontendDist": "../dist",
  "devUrl": "http://localhost:5173"
}
```

`tauri.conf.json` 的 `devUrl` **写死** `http://localhost:5173`。AGENTS.md 解释了后果：

> `tauri:dev` 依赖 Rust，`tauri.conf.json` 的 `devUrl` 写死 `http://localhost:5173`：5173 被占用时 Vite 换端口会直接导致 Tauri 起不来。

这是一个真实的耦合风险。Vite 的 dev server 默认 5173，被占用时会顺延到 5174、5175……。但 Tauri 的 `devUrl` 写死了 5173，Vite 换端口后 Tauri 找不到前端，直接起不来。本地开发时必须保证 5173 空闲，或者手动同步两边。

版本号是唯一来源 `package.json`，`scripts/sync-version.mjs` 负责同步 `src-tauri/Cargo.toml`，`tauri.conf.json` 的 `version` 保持 `"../package.json"` 引用。

### 10.2 插件端与统计开关

AGENTS.md 第一条就点明了这个差异：

> 纯前端 Vue 3 + Vite 的离线 JSON 工具箱……Web 产物注入统计脚本，其余端构建时 `WT_ANALYTICS=off`。

macOS 构建明确带了 `WT_ANALYTICS=off npm run build`，Windows 与 uTools / VS Code 同样走这个开关。只有 Web 端保留统计。这与「无后端、无网络请求、无埋点」的产品定位一致——官网的「0 网络请求」指标主要针对数据处理链路，Web 端的访问统计是另一回事，且只对 Web 生效。

### 10.3 插件通信链路

Chrome 插件通过 `chrome.storage.local` 传 `ej_extract_text`、`ej_compare_text`、`ej_open_file_*`（分片）。改动这条链路要**同时动三个文件**：

- `public/background.js` — 后台脚本，接收消息
- `public/file-handler.js` — 文件处理
- `App.vue` / `JsonFormatter.vue` — 两端消费

这是 MV3 扩展的典型架构：content script、background service worker、页面之间的通信必须显式定义消息类型，没有共享内存。

**接管格式**（README 列出）：`.json` `.jsonc` `.json5` `.geojson` `.har` `.ndjson` `.jsonl` `.xml` `.yaml` `.yml` `.toml` `.csv` `.tsv` `.properties` `.ini` `.txt` `.log` `.md`。

有两个限制值得记录。其一，`.yaml`、`.toml`、`.csv` 这类是否被接管**取决于服务器的 `Content-Type`**——浏览器直接下载的文件不会自动接管，需要在插件内手动导入。其二，Chrome/Edge 扩展**无法注册为系统默认的 `.json` 打开程序**，双击文件交给插件打开需要桌面端或 PWA 文件关联。

### 10.4 发布流水线

`.github/workflows/release.yml` 有 4 个 job：

**Job 1：扩展三件套（ubuntu-latest）**

```
Checkout → Setup Node → Install deps
→ Build Chrome plugin → Build uTools plugin → Build VSCode extension
→ Upload extensions artifact
```

**Job 2：Tauri Windows（windows-latest）**

```
Setup Rust toolchain → Install deps → npm run build（sync version）
→ Build Tauri Windows App → Prepare Windows setup file
→ Upload Windows artifact
```

**Job 3：Tauri macOS（macos-latest）**

```
Setup Rust toolchain → WT_ANALYTICS=off npm run build
→ Build Tauri macOS Universal → Prepare macOS DMG file
→ Upload macOS artifact
```

macOS 用 Universal（Intel + ARM 双架构），Windows 用 MSVC target（`tauri:build:win` 指定 `x86_64-pc-windows-msvc`）。

**Job 4：发布（ubuntu-latest）**

```
Download all artifacts → List collected release files
→ Get Version from package.json → Create or Update GitHub Release
→ Setup Aliyun ossutil（阿里云 OSS 同步）
```

触发条件是 push `v*` tag（`release:tag` → `scripts/release-tag.mjs`）。产物除了 GitHub Release，还同步到阿里云 OSS。

## 十一、测试：111 例提取器回归

这个项目没有 eslint、没有 prettier、没有 tsconfig、没有单测框架。唯一的静态验证是 `npm run build`，唯一的回归测试是 `node test-fixtures.mjs`。

```bash
npm run dev           # vite devserver，默认 5173
npm run build         # 唯一的静态验证
node test-fixtures.mjs   # 提取器回归：111 例，当前 106 通过
```

5 个失败用例是 XML 用例，`tryXmlToJson` 依赖浏览器 `DOMParser`，Node 环境下必然失败。AGENTS.md 明确说**这不是回归**，完整用例要用 `npm run dev` 后打开 `/test`。

这个测试策略的选择有内在逻辑：**提取器是这个项目的核心竞争力**，也是最容易回归的部分。111 个用例覆盖 20+ 种格式的最小输入，每次改动提取链都必须跑。而 UI 组件（树视图、表格视图、对比视图）用 `npm run build` 验证能编译，运行时行为靠实际开发验证。

`TestView.vue`（553 行）是提取用例的自测页面，`/test` 路由。它让开发者在浏览器里跑完整用例（包括 XML），而不只是 Node 下的 106 例。

## 十二、边界与限制

把限制说清楚比罗列功能更有价值。以下是源码与文档确认的事实：

| 限制 | 说明 |
| --- | --- |
| 普通聊天页不能直接选知识库 | 用户端当前只通过智能体使用知识库 |
| Skills 在普通智能体里只保存关系 | 执行链路只接在 Coding Harness |
| 数组顺序不做语义归一 | `[1,2]` vs `[2,1]` 仍报差异 |
| 类型不做归一 | `1` vs `"1"` 仍报差异 |
| 按值脱敏会误伤 | 18 位订单号可能被当身份证 |
| 括号扫描有 O(n²) 风险 | 匹配后只前进 1 格，会重扫嵌套括号 |
| Chrome 扩展不能设默认程序 | 双击 .json 需桌面端或 PWA |
| Tauri devUrl 写死 5173 | 端口被占 Tauri 起不来 |
| 本地 `pnpm install --frozen-lockfile` | lockfile 必须同步（CI 严格校验） |
| 提取器 Node 下 5 例失败 | XML 依赖 DOMParser，非回归 |

这些限制没有一条是缺陷，它们都是**明确的取舍**。知道边界在哪，才不会在错误的地方期待它做不擅长的事。

## 十三、我的判断

**它做对的三件事：**

第一，**核心链路的工程质量高于 UI 工程**。`jsonExtractor.js` 2179 行、`jsonBigInt.js` 214 行、`advancedDetectors.js` 369 行——这些是真正难的部分，每一个都有明确的算法设计（优先级收敛、状态机扫描、占位标记还原、加权相似度）。而 UI 层大量用虚拟滚动和现成组件解决。

第二，**把「数据不出本机」做成产品承诺**。不依赖 HTTP 库、无埋点、无追踪。对于处理可能含敏感信息的 JSON（订单、配置、日志、连接串），这个承诺是可验证的，不是宣传语。数据脱敏里甚至专门写了四种数据库连接串的脱敏规则——这是从真实排障经验里长出来的。

第三，**多端共享源码但各自独立发布**。一份 `src/` 产出五个平台，行为一致，但每个平台的构建产物、发布渠道、统计策略都可以独立控制。Web 有统计、其余端关闭，桌面端走 GitHub Release + 阿里云 OSS，扩展走各自商店。这个架构让维护成本远低于五个独立项目。

**它接受的两个代价：**

其一，**大文件组件的复杂度被推迟而非消除**。树视图、表格视图都要自己实现虚拟滚动和扁平化，这是重复劳动。CodeMirror 承担了代码视图，但如果要支持更大的文件，树和表格视图需要更激进的分片。

其二，**i18n 的构建期预编译是硬约束**。任何新增的翻译逻辑都必须遵守这个约束，否则插件端会退化。这个约束对不熟悉 MV3 CSP 的开发者是个陷阱。

**适合谁：**

- 需要频繁处理「大、脏、怪」JSON 的开发者（日志里的结构化片段、各种语言的 `toString()` 输出）
- 对隐私有要求的团队（数据不出本机是可验证的）
- 需要多端一致体验的个人开发者（Web + 插件 + 桌面 + 两个编辑器插件）

**不适合谁：**

- 只需要格式化一个规整 JSON 的人——在线工具足够
- 需要严格类型检查的项目——这个项目没有 tsconfig
- 需要处理超大文件（千万行）的分析场景——树和表格视图会吃力

## 十四、上手路径

```bash
# 1. 克隆（注意只用 npm，不要混用 yarn/pnpm）
git clone https://github.com/xkloveme/wt-json.git
cd wt-json
npm install

# 2. 启动开发（默认 5173，Tauri 依赖这个端口）
npm run dev

# 3. 唯一静态验证
npm run build

# 4. 提取器回归（改了 jsonExtractor.js / testFixtures.js 才需要跑）
node test-fixtures.mjs

# 5. 构建多端
npm run build:plugin     # Chrome / Edge
npm run build:utools     # uTools
npm run build:vscode     # VS Code
npm run tauri:build      # 桌面端
npm run build:all        # 全部
```

第一次用建议直接粘贴官网代码示例那段日志：

```
INFO order={"id":9007199254740993,
  "user":{"name":"阿吉","phone":"138****8000"},
  "items":[{"sku":"A-1","qty":2}]} done
```

一次走完提取 → 校验 → 视图 → 对比，同时验证大数精度和脱敏是否都生效。然后打开设置看看 8 个色块的主题切换，试试切换到某套墨观配色看所有界面元素是否都跟随——这是检验主题系统维护质量的最快方式。

---

这个工具最打动我的地方，是它在「给人用」和「给开发者用」之间找到了清晰的位置：UI 层克制（20 个依赖、无状态管理、无路由库），核心层厚道（2179 行的提取链、111 个回归用例）。它解决的是一个很小但很真实的问题——**把一段乱七八糟的文本变成可读、可比、可用的 JSON**，并且把这件事做到了本地完成、五端一致的工程完成度。

仓库：<https://github.com/xkloveme/wt-json>，官网：<https://xkloveme.jixiaokang.com/>。如果你也经常被「日志里粘出来的那段 JSON」折磨过，值得一试。
