# 5. 会话与事件模型

这一章讲插件与**会话日志**的关系:什么能写、什么只能读、为什么。它决定了第 4 章那套"不写自定义事件"的做法不是妥协,而是唯一合规的路。官方描述见[架构 › 会话日志](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.zh.md)与[持久化目录](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/persistence-catalog.zh.md)。

## 5.1 日志是唯一事实源

会话日志是一串 `SessionEvent { type, seq, time, data, ignorable? }`。模型历史由 `deriveMessages()` 从日志投影;浏览器 UI 从日志(实时推送 + 重放)构建;fork、恢复、transcript、遥测都派生自它。由此得到 dsh 最重要的一条不变量:

> **模型可见 ⟺ 已记录。** 凡是进入模型请求的东西都必须能从日志重建;新增一项模型可见输入就要新增一个会话事件。

对插件的含义:你能让模型看到的东西只有三条路——工具的 `render` 文本、系统提示词段、以及变成 `user/message` 的输入(`conversation.send` / `agent.inject`)。三条都是平台原生、都被记录。

## 5.2 事件词汇表是封闭的:`KNOWN_SESSION_EVENT_TYPES` 与 `ignorable`

`@deepseek-ai/dsh-session` 生成一份 `KNOWN_SESSION_EVENT_TYPES`(`known-event-types.ts`),是这个构建认识的全部事件类型(`user/message`、`step/start`、`assistant/chunk`、`assistant/message`、`tool/call`、`tool/result`、`step/end`、`turn/end`、`compaction/*`、`approval/*`、`request/header`……)。持久化读取端遇到不在表里的类型时:

- 若事件带 `ignorable: true` → 安全跳过;
- 否则 → **拒绝重建整个会话**(宁可多拒绝,不可静默丢掉一个可能改变后续解释的必需事件)。

而 `Session.append(type, data, ...opts)` 构造信封时只写 `type/seq/time/data`(+ surface 元数据),**没有参数能置 `ignorable`**(`session/src/index.ts:604-628`)。源码注释直说:

> Downstream (out-of-repo) plugin events are outside this list by construction; a registration surface for them is deferred until such a consumer exists.

推论:**树外插件今天不能写自定义会话事件。**你若通过 declaration merging 声明一个 `genui/delta` 并 append 出去,任何没装你插件的 dsh(或换个版本)都打不开这个会话——你"毒化"了用户的日志。genui v0.2 曾计划这么做,发现这条后全部改走原生通道。

## 5.3 插件可用的持久化/可见性通道

| 通道 | 谁写 | 持久 | 重放 | 模型可见 | 大小 | 用途 |
|---|---|---|---|---|---|---|
| `tool/result` 的 `render` 文本 | 你的工具 | ✅ | ✅ | ✅ | >~50 KB 会 spill | 给模型的结果摘要 |
| `tool/result.meta`(`presentationMeta`) | 你的工具 | ✅ | ✅ | ❌ | 无上限 | UI 权威数据、重放源 |
| `user/message`(`conversation.send` / `session.prompt` / `agent.inject`) | 浏览器或宿主 | ✅ | ✅ | ✅ | — | 交互回流、注入上下文 |
| 系统提示词段 | 你的宿主插件 | 每步组装 | — | ✅ | 稳定前缀 | 教模型 |
| `assistant/chunk`(含 `tool-call-delta`) | agent loop | ✅ | ✅ | (它就是模型输出) | — | **你只读**:流式渲染的输入 |
| `tool/call` / `step/*` / `turn/*` | agent loop | ✅ | ✅ | — | — | 你只读:定键、定锚 |
| `tool/code-dispatch-start` / `tool/code-dispatch`(PTC 模式,程序内调用你的工具) | `run_code` 桥 | ✅ | ✅ | ❌ | — | 你只读;payload 有 `arguments`/`content`/`isError` 但**没有 `meta`**(第 3 章 3.7) |

genui 的每一样需求都落进了这张表:文档 → `tool/result.meta`;交互 → `user/message`;流式 → 读 `assistant/chunk`;教学 → 提示词段。**没有一处需要自定义事件。**

## 5.4 `assistant/chunk` 的持久化细节(与流式渲染的关系)

- `assistant/chunk` 的文档定义是 "Raw stream chunk — token-level replay fidelity"(`session/src/types.ts:266`):逐 token 重放是它存在的目的,不是副产物。
- agent loop 在收到每个 chunk 时**先 append**,再继续;工具调用的参数增量(`tool-call-delta`)因此在**工具执行之前**就进了日志并推到浏览器。
- 落盘时 `chunk-rows.ts` 把同一 `turn/step/index` 的连续 delta 压成一行 `tool-call-chunks`(`packChunkRuns`),读回时 `decodeStorageRecord` 展开成一模一样的事件——**刷新后重放看到的流与实时一致**。压缩只接受白名单形状的 chunk,其它保持一事件一行。
- 宿主向浏览器转发会话事件**没有按类型的白名单**;`assistant/chunk` 是核心事件,不会被过滤。

所以第 4 章的流式渲染依赖的是五层**一等契约**,不是任何 hack:

| 层 | 依赖的事实 | 地位 |
|---|---|---|
| agent loop | delta 到达即 append 为 `assistant/chunk`,先于工具执行 | 文档化事件 |
| 持久化 | pack/decode 保持重放一致;`tool/result.meta` 权威 | 平台原生格式 |
| 推送 | 事件无类型白名单地到浏览器 | 核心事件 |
| 客户端扩展点 | `ConversationNodeDefinition` + `conversation.chat.node` slot | 公开契约,一方插件同用 |
| 提供者 | 增量流式输出 tool-call 参数(DeepSeek / OpenAI 兼容都是) | 行业事实;没有 delta 时优雅降级为 settle 后整块渲染 |

## 5.5 真正的脆弱点:宿主契约漂移

流式不脆弱,脆弱的是**加载**:

- 浏览器 bundle 的平台外部模块清单是**手抄** dsh 的 `tsdown.client.ts`(那份预设不导出);
- dsh 的 peer 范围是开放的 `>=0.1.0-rc.5`。

一个未来 dsh 若增删平台模块(升 React 19、加一个共享 UI 包),你的 `client.js` 需要重构建才能加载,而 peer 范围不会在安装期挡下这次升级——用户看到的是白屏或控制台里 `require` 找不到模块。三项加固(已写进 genui README 的 Deferred work):

1. CI 里对**最新** dsh 版本再跑一次 `check`(现在只锁 `rc.6`),提前看到漂移;
2. dsh 出稳定版后把 peer 收成 `^0.1.0` 之类,让不兼容在安装期报错;
3. 一条**录制回放快照**:拿一段真实会话日志(含 `tool-call-chunks` 行)喂给节点做端到端快照,把前四层一次钉住——这也是 dsh 自己的测试哲学。

## 5.6 什么时候你会真的需要插件事件

只有一种情形:**没有任何工具调用产生的 surface 更新**(实时数据推送、宿主主动更新的面板)——它没有 `tool/result` 可挂,又需要重放。到那时再去 dsh 提"给 `append` 加 `ignorable` 写入面"的 PR,带着真实用例(社区正等第一个 consumer)。在那之前:

- 模型驱动的更新(包括 Phase 3 的 `update_ui`)仍是工具调用 → `tool/result.meta`;
- 临时 UI 状态(表单草稿、tab 选中)**不是模型可见的**,不受不变量约束;想跨刷新保留就放客户端存储(按 session + surface 键),别往日志里写。

## 5.7 检查点

- 你没有 append 任何自定义事件类型。
- 模型能看到的每样东西都来自 `render` / 提示词段 / `user/message`。
- 大而不需要模型看的数据在 `tool/result.meta`,且带 `kind` 判别与类型守卫。
- 流式渲染只读 `assistant/chunk`,并能在没有 chunk 时从 `tool/result` 重建。
- 平台外部模块清单与 peer 范围有人盯着(CI 矩阵 / 上限计划)。
