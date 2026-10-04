# 5. 会话与事件模型

这一章讲插件与**会话日志**的关系:什么能写、什么只能读、为什么。它决定了第 4 章那套"不写自定义事件"的做法不是妥协,而是唯一合规的路。官方描述见[架构 › 会话日志](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.zh.md)与[持久化目录](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/persistence-catalog.zh.md)。

## 5.1 日志是唯一事实源

会话日志是一串 `SessionEvent { type, seq, time, data, ignorable? }`。模型历史由 `deriveMessages()` 从日志投影;浏览器 UI 从日志(实时推送 + 重放)构建;fork、恢复、transcript、遥测都派生自它。由此得到 dsh 最重要的一条不变量:

> **模型可见 ⟺ 已记录。** 凡是进入模型请求的东西都必须能从日志重建;新增一项模型可见输入就要新增一个会话事件。

对插件的含义:你能让模型看到的东西只有三条路——工具的 `render` 文本、系统提示词段、以及变成 `user/message` 的输入(`conversation.send` / `agent.inject`)。三条都是平台原生、都被记录。

## 5.2 会话格式变得很快:v0 → v4

dsh 的会话格式在几周内从 v0 升到了 v4,当前 `SESSION_FORMAT_VERSION = 4`(`deepseek-harness:packages/core/session/src/types.ts:89`)。每一步都有一个迁移包 `packages/session/session-format-vN-to-vN+1`,旧会话打开时自动迁移。和插件有关的变化:

- **v2:顶层 `assistant/chunk` 没了。** 模型的完整流在本次尝试结束时一次性嵌进 `assistant/message` 的 `stream` 字段;没有产生消息的失败、重试、取消尝试记成只进日志的 `assistant/attempt`(`types.ts:345-355`;迁移见 `packages/session/session-format-v1-to-v2/README.md`)。实时显示改由浏览器侧的瞬时事件 `assistant/live-chunk` 承担(第 4 章 4.4)。
- **v3:PTC 子调用事件改名。** `tool/code-dispatch(-start)` 变成 `tool/ptc-dispatch(-start)`,仍然**没有 `meta`**(`deepseek-harness:packages/core/tools/src/types.ts:10-25`,后果见第 3 章 3.7)。
- **v4:工具结果改成独立的 tool 消息。** `tool/result.data.message` 是 `role: 'tool'`、带 `toolCallId` 和 `source.callId` 的消息(`deepseek-harness:packages/llm/llm/src/message.ts:172-179`);`tool/result.meta` 仍在(`types.ts:375-385`)。

对插件作者的教训:**只依赖公开类型,不要手写日志行的形状**。genui 0.1 在测试里手造的 `assistant/chunk` 事件,到 0.2 全部要重写。

## 5.3 事件词汇表是封闭的:`KNOWN_SESSION_EVENT_TYPES` 与 `ignorable`

`@deepseek-ai/dsh-session` 生成一份 `KNOWN_SESSION_EVENT_TYPES`(`known-event-types.ts`),是这个构建认识的全部事件类型。持久化读取端遇到不在表里的类型时:

- 若事件带 `ignorable: true` → 安全跳过;
- 否则 → **拒绝重建整个会话**(宁可多拒绝,不可静默丢掉一个可能改变后续解释的必需事件)。

源码注释写明了第三方事件的处境(`deepseek-harness:packages/core/session/src/known-event-types.ts:15-20`):

> Downstream (out-of-repo) plugin events are outside this list by construction. The persisted `SessionEvent.ignorable` marker is the compatibility mechanism; event-name registration was rejected because it does not classify omission safety and would make reads composition-dependent.

也就是说:

- **"让插件注册自己的事件类型"这条路被明确否决了**(理由见 dsh 的 Agent Note `2026-08-30-retain-ignorable-external-session-events.md`)。
- `ignorable` 字段保留了下来,但 `Session.append(type, data, ...opts)` 仍然**没有参数能置 `ignorable`**:它构造信封时只写 `type/seq/time/data` 和 surface 元数据(`deepseek-harness:packages/core/session/src/index.ts:722-747`)。
- V3 → V4 迁移会把不认识的 ignorable 事件改名成 `plugin:<原类型>`(`deepseek-harness:packages/session/session-format-v3-to-v4/src/extension-identities.ts:75-79`)。就算你设法写进了 ignorable 事件,格式升级后它的类型名也会变。

推论:**树外插件不要写自定义会话事件。**你若通过 declaration merging 声明一个 `genui/delta` 并 append 出去,任何没装你插件的 dsh(或换个版本)都打不开这个会话——你"毒化"了用户的日志。genui v0.2 曾计划这么做,发现这条后全部改走原生通道:持久化走 `tool/result.meta`、`tool/call` 参数和 `user/message`。

## 5.4 插件可用的持久化/可见性通道

| 通道 | 谁写 | 持久 | 重放 | 模型可见 | 大小 | 用途 |
|---|---|---|---|---|---|---|
| `tool/result` 的 `render` 文本 | 你的工具 | ✅ | ✅ | ✅ | >~50 KB 会 spill | 给模型的结果摘要 |
| `tool/result.meta`(`presentationMeta`) | 你的工具 | ✅ | ✅ | ❌ | 无上限 | UI 权威数据、重放源 |
| `tool/call.arguments` | agent loop(模型写的参数原文) | ✅ | ✅ | ✅ | — | **你只读**:重放时重建流式阶段的内容 |
| `user/message`(`conversation.send` / `session.prompt` / `agent.inject`) | 浏览器或宿主 | ✅ | ✅ | ✅ | — | 交互回流、注入上下文 |
| 系统提示词段 | 你的宿主插件 | 每步组装 | — | ✅ | 稳定前缀 | 教模型 |
| `assistant/live-chunk`(含 `tool-call-delta`) | 宿主推给浏览器 | ❌ | ❌ | (它就是模型输出) | — | **你只读**:流式渲染的输入 |
| `tool/call` / `step/*` / `turn/*` | agent loop | ✅ | ✅ | — | — | 你只读:定键、定锚 |
| `tool/ptc-dispatch-start` / `tool/ptc-dispatch`(PTC 模式,程序内调用你的工具) | `run_code` 桥 | ✅ | ✅ | ❌ | — | 你只读;有 `arguments`/`content`/`isError` 但**没有 `meta`**(第 3 章 3.7) |

genui 的每一样需求都落进了这张表:文档 → `tool/result.meta`;交互 → `user/message`;流式 → 读 `assistant/live-chunk`;重放 → `tool/call` + `tool/result`;教学 → 提示词段。**没有一处需要自定义事件。**

## 5.5 流式渲染靠的是哪几层契约

| 层 | 依赖的事实 | 地位 |
|---|---|---|
| agent loop | 模型 chunk 以 `agent/assistant-stream` 帧实时发布,先于工具执行;整段流在尝试结束时落进 `assistant/message` / `assistant/attempt` | 文档化行为(`deepseek-harness:docs/architecture.md:99-108`) |
| 推送 | Web 的会话跟随适配器把实时帧转成浏览器侧的 `assistant/live-chunk` | 公开类型(`packages/api/session-controller/src/client/contract/events.ts:7-16`) |
| 持久化 | `tool/call.arguments` 与 `tool/result.meta` 原样落盘 | 平台原生格式 |
| 客户端扩展点 | `ConversationNodeDefinition` + `conversation.chat.node` / `conversation.chat.turnTail` slot | 公开契约,一方插件同用 |
| 提供者 | 增量流式输出 tool-call 参数(DeepSeek / OpenAI 兼容都是) | 行业事实;没有 delta 时优雅降级为 settle 后整块渲染 |

## 5.6 真正的脆弱点:宿主契约漂移

流式不脆弱,脆弱的是**加载**和**跨版本 API**。0.1 → 0.2 一次把三样都打穿了:浏览器服务换名(第 4 章)、平台模块清单变了(第 2 章 2.2)、会话事件改了形状(5.2)。现在的加固:

1. **peer 封顶**:genui 把 dsh peer 收成 `>=0.2.0-rc.2 <0.3.0`,dsh 安装和启动时会强制检查,不兼容在加载前就被拒(第 8 章 8.9)。代价是 dsh 每升一个 minor,插件都要发一版。
2. **平台模块清单**仍是手抄 `PLATFORM_MODULES`,在 `tsdown.config.ts` 注释里写明对应的 dsh 版本,改 peer 范围时一起复查(`dsh-valuz-genui:tsdown.config.ts:22-34`)。
3. 还没做:CI 里对最新 dsh 再跑一次 `check`;一条录制回放快照,把节点在真实会话上的输出钉住(genui README 的 Deferred work)。

## 5.7 什么时候你会真的需要插件事件

只有一种情形:**没有任何工具调用产生的 surface 更新**(实时数据推送、宿主主动更新的面板)——它没有 `tool/result` 可挂,又需要重放。到那时只能去 dsh 讨论一个能声明 `ignorable` 的写入面,带着真实用例。按 5.3,"注册事件名"这条路已经被否决,别再提。在那之前:

- 模型驱动的更新(包括 Phase 3 的 `update_ui`)仍是工具调用 → `tool/result.meta`;
- 临时 UI 状态(表单草稿、tab 选中)**不是模型可见的**,不受不变量约束;想跨刷新保留就放客户端存储(按 session + surface 键),别往日志里写。

## 5.8 检查点

- 你没有 append 任何自定义事件类型。
- 模型能看到的每样东西都来自 `render` / 提示词段 / `user/message`。
- 大而不需要模型看的数据在 `tool/result.meta`,且带 `kind` 判别与类型守卫。
- 流式渲染只读 `assistant/live-chunk`,并能在没有它时从 `tool/call` + `tool/result` 重建。
- 测试里的手造事件用当前格式的字段(`live-chunk` 带 `attemptId`、`tool/result.data.message.source.callId`)。
- 平台外部模块清单与 peer 范围有人盯着(第 8 章 8.9)。
