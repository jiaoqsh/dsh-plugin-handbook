# 4. 浏览器半边

浏览器半边只有一个真正的扩展面:把**会话事件流**变成**对话里的节点**,再用 React 画出来。dsh 一方插件 `ui-workflow-run` 和 `ui-plan` 是模板;机制的官方描述见 [Conversation 组装](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/conversation.zh.md)与 [ui-conversation README › Conversation assembly](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-conversation/README.zh.md#conversation-assembly);样式规范见 [web-styling](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/web-styling.zh.md)。genui 的浏览器半边全部在 `dsh-valuz-genui:src/client/`。

> **dsh 0.2.0-rc.2 起这一章整体换了 API。** 0.1.x 的浏览器服务 `conversationEvents` 和包 `@deepseek-ai/dsh-client-runtime` 已经删除(dsh 提交 `c7d8e32aec` "separate Conversation, Chat, and Trajectory owners"、`be531688f3` "migrate consumers and remove Runtime",2026-08-22)。0.1.x 的客户端插件在 0.2 上加载不了,症状见第 9 章。

## 4.1 注册:`uiConversation`、slot 与文案

```ts
// dsh-valuz-genui:src/client/index.tsx:33-69(节选)
export const inject = ['uiConversation', 'slots', 'sessions', 'theme', 'locale']
export function apply(ctx: Context): void {
  ctx.uiConversation.events.register(genuiSurfaceDefinition)                    // ① 事件 → 状态 → 节点
  ctx.effect(() => ctx.locale.register(NS, { zh, en }), 'valuz-genui: dictionaries')   // ② 文案词典
  ctx.slots.inject('conversation.chat.node', () => ctx.slots.register({         // ③ 按 key 渲染这种节点
    name: 'conversation.chat.node',
    key: 'genui-surface',
    locale: NS,
    inject: (sessionId) => ({ sendAction: createActionSender(ctx, sessionId), colorScheme: ctx.theme.getTheme().active.colorScheme }),
  }, GenuiSurface))
  ctx.slots.inject('conversation.chat.turnTail', () => ctx.slots.register({ ... }, GenuiTurnSurfaces))   // ④ 轮次结尾,见 4.6
}
```

- ① `ConversationNodeDefinition` 注册进 `ctx.uiConversation.events`。一方插件同样这么写(`deepseek-harness:packages/client/ui-workflow-run/src/client/index.ts:27`、`packages/client/ui-chat/src/client/conversation-nodes/tool.ts:305`)。
- ② 文案走 locale。先用 declaration merging 声明命名空间,再在效果里注册词典;slot 注册时写 `locale: NS`,组件就会拿到类型安全的 `t`(`dsh-valuz-genui:src/client/index.tsx:25-30`、`src/client/locales.ts`)。
- ③ `conversation.chat.node` 是 chat 区域的 keyed slot;`key` 对应节点的 `kind`;`inject(sessionId)` 返回要塞给组件的额外 props(每会话计算)。组件 props 类型是 `PropsRuntime<'conversation.chat.node', 'genui-surface'> & PropsLocale<NS> & 你的注入`。
- 节点的 `data` 类型通过 declaration merging 注册进 `ChatNodeDataMap`(`dsh-valuz-genui:src/client/node.ts:37-42`),这样 `node.data` 在组件里是强类型的。

### `Context` 上的服务从哪来

浏览器入口的 `Context` 就是 `import type { Context } from '@deepseek-ai/cordis'`,0.1 的 `ClientContext` 已经没有了。`ctx.uiConversation`、`ctx.slots` 这些字段要靠 **type-only import** 把各包的增量声明拉进编译(原理见第 1 章 1.2)。漏了哪个,tsc 就报 "Property 'xxx' does not exist on type 'Context'"。

| 你用到的 | type-only import | 增量声明位置 |
|---|---|---|
| `ctx.uiConversation` | `@deepseek-ai/dsh-client-ui-conversation/client` | `packages/client/ui-conversation/src/client/index.ts:86` |
| `ctx.sessions` | `@deepseek-ai/dsh-api-session-controller/client` | `packages/api/session-controller/src/client/index.ts:93` |
| `ctx.slots` | `@deepseek-ai/dsh-client-ui-renderer/client` | `packages/client/ui-renderer/src/client/index.ts:45` |
| `ctx.theme` | `@deepseek-ai/dsh-client-ui-theme/client` | `packages/client/ui-theme/src/client/index.ts:113` |
| `ctx.locale` | `@deepseek-ai/dsh-client-locale/client` | `packages/client/locale/src/client/index.ts:85` |
| 会话级 slot 的 `inject(sessionId)` 拿到带品牌的 `SessionId` | `@deepseek-ai/dsh-client-ui-session/client` | `SessionStandardProps.sessionId`,`packages/client/ui-session/src/client/index.ts:163-166` |

(以上路径都在 `deepseek-harness` 仓库。)`SessionId` 类型本身从 `@deepseek-ai/dsh-session/types` 导入。定义相关的类型 `ConversationNodeDefinition / ConversationMatch / ConversationNodeContext / ConversationLocation` 来自 `@deepseek-ai/dsh-client-ui-conversation/client`;`ChatConversationViewNode` 与 `ChatNodeDataMap` 来自 `@deepseek-ai/dsh-client-ui-chat/client`,用 `declare module '@deepseek-ai/dsh-client-ui-chat/client'` 扩展。genui 的全部 import 见 `dsh-valuz-genui:src/client/index.tsx:9-17`。

这些包只在类型层出现,所以放进 `devDependencies`(精确版本)即可;运行时需要的那几个写进 `dsh.client.inject` 和 `peerDependencies`(第 2 章 2.2)。

## 4.2 状态机:`ConversationNodeDefinition`

```ts
// dsh-valuz-genui:src/client/node.ts:107-145(节选)
export const genuiSurfaceDefinition: ConversationNodeDefinition<GenuiNodeState> = {
  kind: 'genui-surface',
  target: 'chat',
  match: (event) => ...,        // 无状态:只看这一个事件,返回 {id, role:'start'|'update'} 或 null
  start: (_context, match) => startState(match),
  update: (context, match) => foldMatch(context.state, match),
  publication: (match) => (match.event.type === 'assistant/live-chunk' ? 'animation-frame' : 'immediate'),
  buildViewNode: (context) => ...,   // State → 视图节点(或 null 表示暂无可画)
}
```

`match` 拿不到历史与上下文。组装器的规则(`deepseek-harness:packages/client/ui-conversation/README.md:34`):

- 同一 `id` 下,**最早加载到的 start** 初始化 State;之后的每个匹配(包括同 id 的又一个 start)都走 `update`。0.1 时"update 之后又来 start"会抛错,0.2 不会了。
- 只有 update、没有 start 的 id 会得到一个**没有 State** 的 Context。`buildViewNode` 要能处理 `context.state === undefined`(genui 返回 null,`node.ts:132`)。
- 不消费模型增量的定义,对 `assistant/live-chunk` 一律返回 null。
- 一次模型尝试结束(settle)时,组装器删掉这次尝试的全部瞬时匹配,再从剩下的最早 start 重新计算受影响的 Context。
- `publication` 决定多久把 State 物化成节点:流式增量用 `animation-frame` 合帧,结果用 `immediate`。

## 4.3 键与锚:按调用 id

0.1 时我们按 `${turn}:${step}` 定键,再在 State 里过滤工具名,并单独记一个锚点。原因是当时 `match` 判断不了某个无名 delta 属不属于 `generate_ui`,而且 start 晚到会抛错。0.2 的组装规则(4.2)让**按调用 id 定键**变得可行,写法也简单得多:

| 事件 | 条件 | 角色 |
|---|---|---|
| `assistant/live-chunk`,`chunk.type === 'tool-call-delta'` | 带 `name` 且等于 `generate_ui` | start |
| 同上 | 不带 `name` | update(别的调用的 delta 找不到 start,自然被忽略) |
| `tool/call` | `name === 'generate_ui'` | start |
| `tool/result` | `event.surfaceOp === 'append'` 且 `meta` 通过 `isGenuiSurfaceMeta` | update,id 取 `event.data.message.source.callId` |

代码在 `dsh-valuz-genui:src/client/node.ts:110-126`。几个要点:

- **只有一个调用的第一片 `tool-call-delta` 带 `name`**,`id` 每片都有(类型见 `deepseek-harness:packages/llm/llm/src/types.ts:456`,`name` 是可选字段)。所以"带名字的 delta"恰好是这个调用的起点。
- `tool/result` 只认 `surfaceOp === 'append'`。`replace` 是压缩(compaction)等替换已有节点用的(`deepseek-harness:packages/core/session/src/types.ts:449-464`)。
- **锚就是 start 事件**:`anchorSeq` 取 `context.start.event.seq`,`location` 取 `context.start.location`(`node.ts:102-104,139-140`)。start 是模型输出的第一片或 `tool/call`,一定在用户消息之后。0.1 里"界面跑到用户消息上面"的 bug 随之消失,不需要再单独记锚。
- 尝试结束后瞬时的 live-chunk 被删掉,Context 从 `tool/call` 这个 start 重新算;`foldMatch` 遇到 `tool/call` 直接用完整的 `arguments` 替换累积的 `argsRaw`(`node.ts:88`)。

## 4.4 折叠流式增量

`assistant/live-chunk` 的形状是 `{ data: { attemptId, turn, step, chunk } }`(`deepseek-harness:packages/api/session-controller/src/client/contract/events.ts:7-16`)。它只存在于浏览器,不落盘,`seq` 只用来把它排在两个持久事件之间。工具调用阶段的 `chunk` 是 `{ type: 'tool-call-delta', index, id, name?, argumentsDelta }`。累积 `argsRaw += argumentsDelta`,得到一段**未闭合的 JSON**,例如 `{"messages":[{"a":1},{"b":2},{"c":`。

`partial-args.ts` 的 `extractCompleteArrayElements(argsRaw, 'messages')` 用一个小型括号/字符串状态机取出**已完整的数组元素**(丢掉正在写的尾巴),每个元素 `JSON.stringify` 一行,拼成 A2UI JSONL 交给渲染器。这就是"模型每写完一个组件,界面就多一块"的机制。要点:

- 只依赖参数是**原生 JSON 数组**(第 3 章的 `items: {type:'json'}`);若模型写的是转义字符串,这条路走不通。
- 渲染器要**容忍不完整**:valuz 的 `A2UIRenderer` 保留最后一版好的 surface,`status: 'running'` 时缺 root 也不报错。
- `publication: 'animation-frame'` 避免每个 token 触发一次 React 渲染。

## 4.5 settle 与重放

`tool/result` 到达时,如果 `meta` 通过 `isGenuiSurfaceMeta`(`kind === 'genui-surface'` 等),State 采纳其中的**权威文档**与事实(组件名、被丢弃组件数、标题),`status: 'success'`。V4 会话格式里 `tool/result` 仍然带 `meta`(`deepseek-harness:packages/core/session/src/types.ts:375-385`)。

刷新或重新打开会话时没有 live-chunk,节点从 `tool/call`(start,带完整参数)和 `tool/result`(update,带 meta)重建。**流式、settle、重放共用一套 start/update 代码**,区别只是 start 来自哪个事件。

## 4.6 轮次结尾:turnTail

**现象。** 一轮对话结束后,ui-chat 会把"过程"(从过程开始到最终回答之间的节点)折叠成一行 "Completed in Ns"(`deepseek-harness:packages/client/ui-chat/src/client/locale.ts:258`)。你的节点如果落在过程里,用户看到的就是一行折叠标题,结果被藏起来了。

**规则。**

- 哪些节点不进折叠是写死的:`deepseek-harness:packages/client/ui-chat/src/client/conversation-nodes/process-groups.ts:13` 的 `INDEPENDENT` 与 `packages/client/ui-chat/src/client/contract/turn-process.ts:20-29` 的 `TURN_PROCESS_INDEPENDENT_KIND_LIST`。插件的节点 kind 不在里面,也没有注册入口。
- 还在进行、被中止或出错的轮次保持展开(`turnProcessAlwaysOpen`,`turn-process.ts:69-74`)。
- 只有"工作细节 = Verbose"会关掉折叠(`foldCompletedTurns: false`,`deepseek-harness:packages/client/ui-chat/src/client/presentation-policy.ts:46-48`)。

**正路是公开的 list slot `conversation.chat.turnTail`**:"Ordered feature contributions before a completed Turn's action row"(`deepseek-harness:packages/client/ui-chat/src/client/contract/slots.ts:310-315`)。每个结束的轮次都有一个 turn-tail 节点,它不进折叠;owner props 是 `{ turn, seq, openFile }`(`slots.ts:49-53`)。在里面按轮次读出你自己的节点数据:

```ts
// dsh-valuz-genui:src/client/index.tsx:49-68(节选);模板是 deepseek-harness:packages/client/ui-plan/src/client/index.ts:79-95
ctx.slots.inject('conversation.chat.turnTail', () => ctx.slots.register({
  name: 'conversation.chat.turnTail',
  id: 'valuz-genui-surfaces',
  locale: NS,
  inject: (sessionId) => {
    const binding = ctx.sessions.binding(sessionId)
    if (binding === undefined) throw new Error(`valuz-genui: unknown session "${sessionId}"`)
    const chat = ctx.uiConversation.binding(binding).target('chat')
    return {
      ...surfaceFace(sessionId),
      keyedHooks: {
        surfaces: (turn) => chat.getSnapshot()!.nodes.turnDataSource(Number(turn), 'genui-surface'),
      },
    }
  },
}, GenuiTurnSurfaces))
```

- `keyedHooks.surfaces` 在组件里变成 `useSurfaces(key)`(`dsh-valuz-genui:src/client/GenuiSurface.tsx:130-131`)。
- `turnDataSource(turn, kind)` 按轮次和 kind 索引,**包括 hidden 节点**(`deepseek-harness:packages/client/ui-chat/src/client/contract/snapshot.ts:38-45`)。ui-plan 就用一个 `visibility: 'hidden'` 的纯数据节点 `submitted-plan` 喂它的 turnTail(`packages/client/ui-plan/src/client/plan-definition.ts:28-32`)。
- genui 的做法:轮次结束后,turnTail 画出本轮已 settle 的界面(同标题只留最后一版);过程里的节点在 `location.turn.status === 'closed'` 时缩成一行引用 "已生成界面:X(见本轮结尾)"(`GenuiSurface.tsx:100-115`)。轮次进行中,过程节点照常流式渲染。

**设计规则**:过程数据(计划评审、workflow 成员)折起来没问题;**结果和可交互的输出**要放进 turnTail。

## 4.7 渲染器组件

`GenuiSurface.tsx` 是普通 React 18 组件,几条实践:

- **ErrorBoundary 必须有**:第三方渲染器抛错不能把整个 chat 拖垮;fallback 一行提示 + `console.warn`(`GenuiSurface.tsx:48-66`)。
- **主题跟随宿主**:`inject` 给的 `colorScheme` 来自 `ctx.theme.getTheme().active.colorScheme`,组件用 `useEffect` 跟随并传给渲染器(`theme={theme}`,`GenuiSurface.tsx:74-76`)。渲染器内部只切 `data-theme`,颜色是自己的调色板——所以"跟随亮暗"有了,"用宿主的确切颜色"还没有(主题令牌桥,第 6 章 deferred)。
- **样式**:CSS Modules(`GenuiSurface.module.css` → `css.root`)+ 第三方整包样式(`import '@valuz/a2ui/styles.css'`),都由第 2 章的 tsdown CSS 插件内联进 `client.js` 并在 factory 执行时注入 `<style data-plugin>`。第三方样式要**自带作用域**(A2UI 的全部规则都在 `.valuz-a2ui` 下,不污染宿主)。
- **文案不硬编码**:用 slot 给的 `t`(4.1 ②)。
- **交互出口只有一个函数**:`onAction` → `sendAction({surfaceId, name, component, context?})`。

## 4.8 动作桥:回到模型的唯一正道

```ts
// dsh-valuz-genui:src/client/action-bridge.ts:20-33(节选)
const conversation = ctx.sessions.scope(sessionId)?.get('conversation')
void conversation.send(formatUiAction(action)).catch(...)   // → 下一步的 user/message
```

- 0.2 里 `scope(...).get('conversation')` 已经有类型(来自 `@deepseek-ai/dsh-client-ui-conversation/client` 的增量声明),不用再 `as` 断言。
- 没有第三方私有 RPC。浏览器要影响宿主,就把它变成**用户消息**(`conversation.send`)或 `session.prompt(...)`。天然满足"模型可见 ⟺ 已记录"。
- 信封是机器可解析、模型可读的标签:`<ui_action surface="call-1" component="btn" name="refresh">{"range":"7d"}</ui_action>`,`meta.ts` 里 `formatUiAction` / `parseUiAction` 成对,属性值过白名单或 `encodeURIComponent`。
- 失败只 `console.warn` 不 throw,**且不打印 payload**(可能含表单内容)。
- 系统提示词段告诉模型收到 `<ui_action>` 该怎么办(文本回答,或再调 `generate_ui` 给完整新文档)。

## 4.9 不能做与不必做

- **不做 DOM 观察/劫持**:先例插件三次静默故障都来自宿主 DOM 漂移。我们只用注册式扩展点。想让结果不被折叠,用 turnTail,别去改折叠逻辑。
- **不写模块级单例状态**:热替换会留下旧实例;状态放 State 或 React 内。
- **不在渲染函数里发副作用**:`sendAction` 只在事件处理器里调用。
- **不需要自定义会话事件**:流式来自 `assistant/live-chunk`,重放来自 `tool/call` 参数与 `tool/result.meta`,回流是 `user/message`——第 5 章解释为什么这不仅够用,而且是唯一合规的路。

## 4.10 浏览器半边检查点

- 服务级 `inject`、`dsh.client.inject`、type-only import 三处对齐;`PLUGIN_ID === package.json.name`。
- `match` 无状态,按调用 id 定键;`buildViewNode` 能处理没有 State 的 Context。
- 锚在 start 事件上,刷新后位置不变。
- 增量用 `animation-frame`,结果 `immediate`;渲染器容忍不完整。
- 结果和交互输出在 turnTail 里也能看到;在 Standard 工作细节下验证过折叠后的样子。
- ErrorBoundary、主题跟随、样式有作用域、文案走 locale。
- 交互只经 `conversation.send`;失败不 throw、不泄 payload。
- 客户端测试:状态机纯函数测试 + jsdom 渲染测试(第 7 章)。
