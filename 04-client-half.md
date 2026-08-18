# 4. 浏览器半边

浏览器半边只有一个真正的扩展面:把**会话事件流**变成**对话里的节点**,再用 React 画出来。dsh 一方插件 `ui-workflow-run` 是模板;cookbook 的[添加一个 Conversation Node](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cookbook/adding-a-conversation-node.zh.md)是仓库内视角;样式规范见 [web-styling](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/web-styling.zh.md)。genui 的浏览器半边全部在 `dsh-valuz-genui:src/client/`。

## 4.1 两个注册

```ts
// dsh-valuz-genui:src/client/index.tsx
export const inject = ['conversationEvents', 'slots', 'sessions', 'theme']
export function apply(ctx: ClientContext): void {
  ctx.conversationEvents.register(genuiSurfaceDefinition)              // ① 事件 → 状态 → 节点 的状态机
  ctx.slots.inject('conversation.chat.node', () => ctx.slots.register({ // ② 按 key 渲染这种节点的组件
    name: 'conversation.chat.node',
    key: 'genui-surface',
    inject: (sessionId) => ({ sendAction: createActionSender(ctx, sessionId), colorScheme: ctx.theme.getTheme().active.colorScheme }),
  }, GenuiSurface))
}
```

- ① `ConversationNodeDefinition` 来自 `@deepseek-ai/dsh-client-runtime/client`,是公开契约(`contract/conversation.ts`)。
- ② `conversation.chat.node` 是 chat 区域的具名 slot;`key` 对应节点的 `kind`;`inject(sessionId)` 返回要塞给组件的额外 props(每会话计算)。组件 props 类型是 `PropsRuntime<'conversation.chat.node', 'genui-surface'> & GenuiSurfaceInjected`。
- 节点的 `data` 类型通过 declaration merging 注册进 `ChatNodeDataMap`(`node.ts:36`),这样 `node.data` 在组件里是强类型的。

## 4.2 状态机:`ConversationNodeDefinition`

```ts
export const genuiSurfaceDefinition: ConversationNodeDefinition<GenuiNodeState> = {
  kind: 'genui-surface',
  target: 'chat',
  match: (event) => ...,        // 无状态:只看这一个事件,返回 {id, role:'start'|'update'} 或 null
  start: () => initial(),       // 唯一的 start 匹配 → 初始 State
  update: (context, match) => foldMatch(context.state, match),   // 每个 update 匹配按日志顺序折叠
  publication: (match) => (match.event.type === 'assistant/chunk' ? 'animation-frame' : 'immediate'),
  buildViewNode: (context) => ...,   // State → 视图节点(或 null 表示暂无可画)
}
```

引擎保证:`match` 拿不到历史与上下文;同一 `id` 只能 start 一次,之后全是 update;update 按 seq 升序;`publication` 决定多久把 State 物化成节点(流式增量用 `animation-frame` 合帧,结果用 `immediate`)。

## 4.3 键:为什么按 `turn:step` 而不是 callId

直觉是按工具调用 id 键——但 `match` 是**无状态**的:后续的 `tool-call-delta` 事件不带工具名(只有第一片带 `name`),`match` 无法判断某个 delta 属不属于 `generate_ui`;而且引擎在"已有 update 匹配之后又来 start"时会抛错。所以:

- **键 = `${turn}:${step}`**(`node.ts:120`):`step/start` 是 start,同一步的所有 `assistant/chunk` 与 `tool/result` 都是 update。
- **在 State 里过滤**:第一片带 `name === 'generate_ui'` 的 delta 记下 `renderIndex`(块索引)与 `surfaceId`;之后只累积同索引的 delta。别的工具的 delta 进来是 no-op(测试 `ignores tool-call-delta for other tools in the same step`)。
- 一步里没有 `generate_ui` 调用时 `buildViewNode` 返回 null——引擎不画。

## 4.4 锚:节点出现在哪

chat 按 `anchorSeq` 排节点。如果用 `step/start` 的 seq 当锚,你的界面会跑到**用户消息上面**(那正是我们踩的 bug:"图在最上面")。所以 State 里记 `anchorSeq`/`location`:

- 流式:第一片 `generate_ui` delta 的 `event.seq` 与 `match.location`(`node.ts:75-78`);
- 重放(没有 chunk):`tool/result` 的 seq(`node.ts:91`)。

`buildViewNode` 用 `state.anchorSeq ?? context.start.event.seq`。`ui-workflow-run` 也是锚在它自己的工具事件上。

## 4.5 折叠流式增量

`assistant/chunk` 的 `data.chunk` 在工具调用阶段是 `{ type: 'tool-call-delta', index, id, name?, argumentsDelta }`(`name` 只在首片)。累积 `argsRaw += argumentsDelta`,得到一段**未闭合的 JSON**,例如 `{"messages":[{"a":1},{"b":2},{"c":`。

`partial-args.ts` 的 `extractCompleteArrayElements(argsRaw, 'messages')` 用一个小型括号/字符串状态机取出**已完整的数组元素**(丢掉正在写的尾巴),每个元素 `JSON.stringify` 一行,拼成 A2UI JSONL 交给渲染器。这就是"模型每写完一个组件,界面就多一块"的机制。要点:

- 只依赖参数是**原生 JSON 数组**(第 3 章的 `items: {type:'json'}`);若模型写的是转义字符串,这条路走不通。
- 渲染器要**容忍不完整**:valuz 的 `A2UIRenderer` 保留最后一版好的 surface,`status: 'running'` 时缺 root 也不报错。
- `publication: 'animation-frame'` 避免每个 token 触发一次 React 渲染。

## 4.6 settle 与重放

`tool/result` 到达时,如果 `meta` 通过 `isGenuiSurfaceMeta`(`kind === 'genui-surface'` 等),State 采纳其中的**权威文档**与事实(组件名、被丢弃组件数、标题),`status: 'success'`。刷新页面时没有 chunk 流,同一个 `tool/result` 直接建起节点——**`tool/result.meta` 既是 settle 源也是重放源**,这一条设计让"流式"和"重放"共用一套代码路径,只多了两行 `?? event.seq`。

## 4.7 渲染器组件

`GenuiSurface.tsx` 是普通 React 18 组件,几条实践:

- **ErrorBoundary 必须有**:第三方渲染器抛错不能把整个 chat 拖垮;fallback 一行提示 + `console.warn`。
- **主题跟随宿主**:`inject` 给的 `colorScheme` 来自 `ctx.theme.getTheme().active.colorScheme`,组件把它传给渲染器(`theme={theme}`)。渲染器内部只切 `data-theme`,颜色是自己的调色板——所以"跟随亮暗"有了,"用宿主的确切颜色"还没有(主题令牌桥,第 6 章 deferred)。
- **样式**:CSS Modules(`GenuiSurface.module.css` → `css.root`)+ 第三方整包样式(`import '@valuz/a2ui/styles.css'`),都由第 2 章的 tsdown CSS 插件内联进 `client.js` 并在 factory 执行时注入 `<style data-plugin>`。第三方样式要**自带作用域**(A2UI 的全部规则都在 `.valuz-a2ui` 下,不污染宿主)。
- **交互出口只有一个函数**:`onAction` → `sendAction({surfaceId, name, component, context?})`。

## 4.8 动作桥:回到模型的唯一正道

```ts
// dsh-valuz-genui:src/client/action-bridge.ts
const conversation = ctx.sessions.scope(sessionId)?.get('conversation') as { send(text: string): Promise<void> } | undefined
void conversation.send(formatUiAction(action)).catch(...)   // → 下一步的 user/message
```

- 没有第三方私有 RPC。浏览器要影响宿主,就把它变成**用户消息**(`conversation.send`)或 `session.prompt(content, 'queue'|'steer')`。天然满足"模型可见 ⟺ 已记录"。
- 信封是机器可解析、模型可读的标签:`<ui_action surface="call-1" component="btn" name="refresh">{"range":"7d"}</ui_action>`,`meta.ts` 里 `formatUiAction` / `parseUiAction` 成对,属性值过白名单或 `encodeURIComponent`。
- 失败只 `console.warn` 不 throw,**且不打印 payload**(可能含表单内容)。
- 系统提示词段告诉模型收到 `<ui_action>` 该怎么办(文本回答,或再调 `generate_ui` 给完整新文档)。

## 4.9 不能做与不必做

- **不做 DOM 观察/劫持**:先例插件三次静默故障都来自宿主 DOM 漂移。我们只用注册式扩展点。
- **不写模块级单例状态**:热替换会留下旧实例;状态放 State 或 React 内。
- **不在渲染函数里发副作用**:`sendAction` 只在事件处理器里调用。
- **不需要自定义会话事件**:流式来自 `assistant/chunk`,持久来自 `tool/result.meta`,回流是 `user/message`——第 5 章解释为什么这不仅够用,而且是唯一合规的路。

## 4.10 浏览器半边检查点

- `inject` 与 `dsh.client.inject` 都写全;`PLUGIN_ID === package.json.name`。
- `match` 无状态、键稳定;工具过滤放在 State。
- 锚在自己的工具事件上,刷新后位置不变。
- 增量用 `animation-frame`,结果 `immediate`;渲染器容忍不完整。
- ErrorBoundary、主题跟随、样式有作用域。
- 交互只经 `conversation.send`;失败不 throw、不泄 payload。
- 客户端测试:状态机纯函数测试 + jsdom 渲染测试(第 7 章)。
