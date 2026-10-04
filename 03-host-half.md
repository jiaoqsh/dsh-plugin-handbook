# 3. 宿主半边

宿主半边就是 `apply(ctx, config)` 里做的那些注册:工具、配置、系统提示词段、(可选)skill,以及对 dsh 能力的消费。官方入门见[构建工具](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/tool.zh.md)与[插件配置](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/config.zh.md);cookbook 里的[添加一个工具](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cookbook/adding-a-tool.zh.md)是仓库内视角。这一章讲树外插件在真实项目里怎么把这几样写对。

## 3.1 工具:`defineTool` 的每一格该放什么

两个案例的工具注册结构一致(`dsh-document:src/tool.ts:171`、`dsh-valuz-genui:src/tool.ts:38`):

```ts
ctx.tools.register(defineTool({
  name, description,
  parameters: { ... },                 // 模型看到的参数 schema
  output: {
    schema: { ... },                   // execute 返回值的 schema(结构化,便于校验与展示)
    render: (args, value) => [...],    // ← 模型看到的文本
    presentationMeta: (args, value) => value.meta,   // ← 持久到 tool/result.meta,模型看不到
  },
  isConcurrencySafe: () => true,
  async execute(args, exec) { ... },   // 业务;throw 就是给模型的错误
  presentCall(args): GenericCallView { ... },       // UI 卡片:调用中
  presentResult(args, result) { ... },              // UI 卡片:结果(可选)
}))
```

### 描述与命名:写给模型的第一行代码

- **说清何时用、何时不用、和谁区分。** `read_document` 的描述末尾:"For plain text or source files use read; for images use read_image." 因为它和 dsh 自带的 `read` 家族在模型眼里是同类,不划清边界模型会乱选。
- **声明副作用边界。** 两个工具都写了 "Returns text only and writes no files"——权限系统和模型都会读这句。
- **名字跟着模型的意图走。** dsh-document 先叫 `convert_document`,真机测试发现模型把它当"读"用,改成 `read_document`(与 `read`/`read_image` 成一族)。genui 中途叫过 `render_ui`,最终回到 `generate_ui`——面向用户的动词是"生成一个交互界面"。命名改动便宜,趁没发布改。
- **参数用真实类型。** `messages: { type: 'array', items: { type: 'json' } }` 让模型写原生 JSON 对象数组而不是转义字符串——这直接决定流式解析可行、校验干净(第 6 章)。

### `render` 与 `presentationMeta`:两条出口,两个受众

- `render` 的返回进入模型上下文。**模型可见的文本超过约 50 KB 会被 spill(落盘、只给摘要)**,所以别把大文档塞这里。genui 只回一行收据:"Rendered an interactive UI (surface X) with N component type(s)... do not repeat it as text."
- `presentationMeta` 的返回写进 `tool/result.meta`:**持久、随会话重放、无大小上限、不进模型上下文**。genui 把整份 A2UI 文档、组件名、校验警告全放这里,浏览器节点靠它做权威渲染与刷新后重放(第 4 章)。给它一个 `kind` 判别字段(`GENUI_META_KIND = 'genui-surface'`)和类型守卫(`isGenuiSurfaceMeta`),两半共用(`src/meta.ts`)。

`presentationMeta` 的返回类型是 `JsonValue`。dsh 0.2 起它从 `@deepseek-ai/dsh-util-values` 导入(`deepseek-harness:packages/util/values/src/index.ts:4`,用法见 `dsh-valuz-genui:src/tool.ts:13,76`),不再从 `dsh-session` 拿。这个包要进 `devDependencies`。

### `presentCall` / `presentResult`:UI 意图在设计时定

dsh 的规范是"工具的 UI 渲染意图是设计的一部分,事先决定,展示方法是 `args` 的纯函数"。可选卡片是封闭 union(`generic` / `terminal` / `diff` / `search` / `read` / `web`),配 `kind` 与 `locations`:

```ts
// dsh-document:src/tool.ts:277 — 读类工具:generic 卡 + kind:'read' + 可跟随的文件位置
presentCall(args) {
  return { card: 'generic', title: `Read ${args.file_path} as Markdown${window}`, kind: 'read', locations: [{ path: args.file_path }] }
}
```

封闭 union 表达不了生成式 UI——所以 genui 用**自定义 Conversation Node** 做真正的界面,`presentResult` 只回一张 `generic` 卡作为**没有浏览器半边的宿主上的退路**(比如 headless、或旧版 shell)。

### `execute(args, exec)`:能用的东西

- `exec.signal`:一切 I/O 都传下去;dsh-document 的 worker 在它触发时 `terminate()`。
- `exec.callId`:稳定的调用标识,类型是带品牌的 `ToolCallId`(dsh 0.2 起由 `CallId` 改名,`deepseek-harness:packages/llm/llm/src/brand.ts:31`;测试里用 `ToolCallId('call-1')` 构造,见 `dsh-valuz-genui:tests/host/generate-ui.spec.ts:37`)。genui 拿它当 `surfaceId`——同一份 UI 在流式阶段(来自 `tool-call-delta.id`)和 settle 阶段(`callId`)有同一个 id。
- `exec.agent?.session.header.cwd`:会话工作区。dsh-document 用它解析相对路径,并给非 agent 调用者留 provider 缺省的退路。
- **抛错就是给模型的反馈**:`throw new Error('generate_ui: messages must not be empty')`——消息要可操作(说清哪里错、该怎么改),别泄内部栈。dsh-document 把引擎错误翻译成 `describeFailure(displayPath, error)` 再抛。
- `isConcurrencySafe: () => true`:只读工具标 true,允许与同一步的其他工具并行。

## 3.2 Config:schema 填缺省,`apply` 断言约束

```ts
export interface Config { maxDocumentBytes?: number; alwaysOnFullGuide?: boolean }
export const Config: Schema<Config> = Schema.object({
  maxDocumentBytes: Schema.number().default(DEFAULT_MAX_DOCUMENT_BYTES),
  alwaysOnFullGuide: Schema.boolean().default(false),
})
export function apply(ctx: Context, rawConfig: Config): void {
  const config = rawConfig as Required<Config>
  if (!Number.isInteger(config.maxDocumentBytes) || config.maxDocumentBytes < 1) throw new Error('genui: maxDocumentBytes must be a positive integer')
```

三条 dsh 规矩,树外插件也照做:

- **接口字段可选、`apply` 收到的已是完整记录**(schema 填了缺省)。导出 `DEFAULT_*` 常量只是为了文档与测试引用,不是"可配置"的替代品。
- **无硬编码可调值**:凡是部署会不同的选择(上限、超时、profile 名、模式开关)都是 Config 字段,能从 `cordis.patch.yml` 改。协议常量、外部规范、安全不变量才写死。
- **错配置在加载时炸**:schema 校验失败 loader 会拒绝该行;schema 表达不了的(正整数、互斥)在 `apply` 开头 throw。dsh-document 六个上限逐个 `assertPositiveInteger`。真机验证过:给 `cordis.patch.yml` 一个坏值,`dsh` 启动直接报错并指出行 id——这是想要的行为。

枚举用 `Schema.union(['fidelity', 'compact']).default('fidelity')`。

**`@deepseek-ai/schemastery` 的版本要和 dsh 对齐。** 它是你的 `dependency`,dsh 自己也带一份(0.2.0-rc.2 里是 `3.18.4`,`deepseek-harness:vendor/schemastery/package.json:4`)。两份版本不同时 pnpm 会装两份,`Schema<Config>` 的类型就对不上(症状见第 9 章)。genui 写成 `~3.18.4`(`dsh-valuz-genui:package.json`)。

## 3.3 系统提示词段

```ts
ctx.systemPrompt.section({
  name: 'tool:read_document',   // 具名,便于排查与覆盖
  order: 100,                   // 升序拼接;外部插件可用任意有限值,见下
  text: `Use the read_document tool — not read or shell commands — to inspect PDF, ...`,
})
```

- **`order` 决定位置。** 段按 `order` 升序拼接,相同 order 按名字排(`deepseek-harness:packages/core/system-prompt/README.md:56`)。0.1 时"工具指导用 100–199"的说法在 0.2 已经不成立:一方段位集中在 `SECTION_ORDERS`(`deepseek-harness:packages/core/system-prompt/src/index.ts:125-160`),工具指导在 1000–3100(`TOOL_BASH` 1000 … `MCP_SERVERS` 3100),`PTC_ONLY` 800、`TOOLS_SDK` 5000。两个插件仍写 100 / 110,所以它们排在全部一方工具说明**之前**。目前能用,但如果想让自己的工具说明挨着一方工具说明,就选 3100 和 5000 之间的值。
- `text` 可以是**函数**,每次组装提示词都调用。genui 用它按"当前有没有 skill 能力"动态选紧凑/完整指南——不必重启、不必监听事件。
- 内容是**稳定前缀**(KV-cache 友好):不要在里面放时间戳、随机内容、会话相关值。
- 写什么:何时用这个工具、如何续读/续写、格式约定。genui 的段还包含 A2UI 消息契约与"收到 `<ui_action>` 时怎么办"。
- **量化 token 成本**。genui 的完整指南 31,642 字符 ≈ 9k tokens,常驻太贵,于是有了 3.4。粗算 tokens ≈ 字符数 / 3.5(英文技术文本)。

## 3.4 按需 skill:把大块知识挪出常驻提示词

dsh 的 skill 能力(`ctx.skills`)是可选服务:web profile 装了 `tool-skill` + `skill-filesystem`,headless 可能没有。genui 的做法是**混合**:

```ts
// dsh-valuz-genui:src/index.ts
const guideAvailableAsSkill = (): boolean => !config.alwaysOnFullGuide && ctx.get('skills') !== undefined
ctx.systemPrompt.section({ name, order, text: () => (guideAvailableAsSkill() ? buildCompactGuide() : buildAuthoringGuide()) })
if (!config.alwaysOnFullGuide) {
  ctx.inject(['skills'], (skillCtx) => skillCtx.skills.register({
    name: 'genui', description: GENUI_SKILL_DESCRIPTION, content: buildAuthoringGuide(), source: 'runtime',
  }))
}
```

- 常驻:紧凑指南(组件名 + 一句话用途 + 消息契约 + "写之前先加载 genui skill"),10,851 字符 ≈ 3.1k tokens。
- 按需:完整字段签名目录,≈ 9k tokens,模型需要时加载。
- 每次请求省 ≈ 5.9k tokens;没有 skill 能力时自动退回完整指南;`alwaysOnFullGuide: true` 是逃生口。
- 运行时 skill 只校验 `name`(kebab-case)、`description`、invocation;`source: 'runtime'` 是类型要求。
- **行为风险要写进 README**:模型得先加载 skill 再写,否则靠猜字段会被 sanitizer 丢组件。真机测下来 DeepSeek 会照做,但这是提示词工程,不是保证。

## 3.5 消费能力:走 seam,不走 `node:*`

dsh-document 读文件全程 `ctx.fs`(`dsh-document:src/tool.ts:234-243`):

```ts
const target = await ctx.fs.resolve(input.filePath, { cwd, signal: exec.signal })   // 相对路径按会话工作区
const info = await ctx.fs.stat(target, exec.signal)                                  // 存在性、类型
const bytes = await ctx.fs.readBytes(target, exec.signal, caps.maxInputBytes)        // provider 执行上限,超大文件不进内存
```

好处是免费的:文件策略(允许/拒绝路径)、沙箱替换(E2B 等 provider)、`displayPath` 统一展示,全都不用你写。代价是测试要装一个 Provider(`@deepseek-ai/dsh-fs-local`,devDependency)。

**重活放 worker。** PDF 引擎是 wasm、CPU 密集、还可能卡死;dsh-document 每次调用 `new Worker(WORKER_URL)`(`src/pdf-inspector.ts:65`),`signal` 触发就 `terminate()`。三个工程细节:

- worker 文件是**独立产物**(`tsdown entry: ['src/index.ts', 'src/pdf-worker.ts']`),按路径 spawn、不 import。
- 源码模式下 worker 以 `.ts` 跑,靠 `--experimental-strip-types`,所以 worker 里**不能有 TS 专属运行时语法**(enum、参数属性等)。
- 每次调用一个 worker,简单、可取消、无共享状态;吞吐不是这个工具的瓶颈。

## 3.6 如果你真的要在工具里调模型

能用的是 `ctx.llm.stream(GenerateOptions)`——**只有流式,没有非流式,没有 JSON mode**;provider/model 从 `exec.agent.session.requestHeader()?.config` 取(跟随会话当前模型),退路 `exec.agent.options` → `ctx.agentDefaultModel.currentSelection()`;`dsh-llm-retry` **只覆盖 agent loop**,工具里的调用要自理重试。genui v0.2 这样做过,v0.3 **删掉了**——第 6 章说明为什么"让主模型直接产出"几乎总是更好的设计。仓库里也没有任何一方工具在内部调模型,这是个信号。

## 3.7 PTC 模式下你的工具长什么样

dsh 的 agent preset 里有一个 **PTC 模式**(Programmatic Tool Calling,0.1 时内部叫 Code Mode)。它与 Standard 的差别在 `deepseek-harness:packages/bundle/web-app/presets/ptc.patch.yml:144-147` 里只有一行:

```yaml
- id: tool-presentation
  name: '@deepseek-ai/dsh-agent-tool-presentation'
  config:
    mode: ptc         # native | ptc | both(0.1 时叫 code)
```

这一行改变的是**工具的呈现方式**,注册表本身不变。呈现方式按 preset 生效,所以同一进程里 PTC 会话与 Standard 会话可以并存(`deepseek-harness:packages/core/agent-tool-presentation/README.md:12`)。在 `ptc` 下模型看到的世界:

| | Standard(`native`) | PTC(`ptc`) |
|---|---|---|
| wire 上的工具目录 | 每个工具一个 JSON-schema 函数 | **只有 `run_code({code, description})`** |
| 其他工具 | 就是目录本身 | 系统提示词里一段**自动生成的 TypeScript SDK 声明**(`tools:sdk` 段,order 5000):`tools["read_document"](…)` 带参数类型、JSDoc、**返回值类型** |
| 规则 | — | `tools:ptc-only` 段(`deepseek-harness:packages/core/tools/src/index.ts:878`):只能直接调 `run_code`;执行端用**同一个谓词**拒绝直接调其他工具(`UNKNOWN_TOOL`) |
| 多步操作 | N 次往返,每个中间结果回灌上下文 | 一个程序跑完;只有 `console.log` 与 `return` 的内容回到模型 |

运行机制:`run_code` 走完整工具管线 → 每个可见工具变成一个 async binding,打包成程序里的全局 `tools` → `ctx.ptcRuntime` 执行程序。dsh 0.2 的 Node 实现每次调用**起一个全新的 Node 进程**,和 Bash 走同一套平台沙箱策略,有截止时间、输出上限和 V8 堆上限(`deepseek-harness:packages/ptc-runtime/ptc-runtime-node/README.md:12`)→ 程序内每次 `tools.x()` **重新进入完整工具管线**(审批、guard、`isConcurrencySafe` 分类,并发安全的可并行到 `maxParallelSubCalls`,默认 10,`deepseek-harness:packages/core/tools/src/index.ts:796`)→ 每个子调用记 `tool/ptc-dispatch-start` / `tool/ptc-dispatch` 事件(日志与 UI 可见,不进模型历史)→ `logs + return` 作为 `run_code` 的结果回模型。

**对你的工具意味着什么**(这些都是 3.1 里那几格的直接后果):

- **`output.schema` 就是 SDK 的返回类型**。binding 返回的是工具的**规范 JSON 值**(`execute` 的返回),不是 `render` 出来的文本;失败抛真实的 `ToolCallError`。所以:`output.schema` 要精确;规范值要对**程序**有用(id、结构化字段),而不只是对人可读——程序会 `const r = await tools["read_document"]({file_path}); r.lines.map(...)`。
- **`description` 变成 JSDoc**,`parameters` 变成参数类型;工具名作为带引号的对象键,任意名字都行。
- **审批、guard、并发分类照常生效**;`isConcurrencySafe` 为 true 的工具在程序里的连续调用会并行。
- **任何依赖"模型直接调用你的工具"的设计在 `ptc` 下失效**。两个典型:依赖模型流式输出你的工具参数(`tool-call-delta`)做实时渲染;依赖你的 `tool/result.meta` 做 UI。程序内调用产生的是 `tool/ptc-dispatch`,payload 为 `{rootCallId, parentCallId, subCallId, name, arguments, isError, content, error?}`——**仍然没有 `meta`**(`deepseek-harness:packages/core/tools/src/types.ts:20-25`,写入点 `packages/core/tools/src/ptc.ts:586`);唯一的 `tool/result` 属于 `run_code`。genui 就是这样:在 PTC 下模型流式输出的是 TypeScript 而不是 `messages[]`,settle 后 meta 也不在任何浏览器节点能读的事件里,**界面完全不出现**,只剩 run_code 卡片里一行文本收据。`read_document` 这类纯文本/纯数据工具则完全不受影响,反而更好用。

**该做的**:在 Standard 和 PTC 两个 preset 下各真机跑一遍;若你的工具必须被直接调用,在 README 里写明"用 Standard 或 `mode: both`";若要在 `ptc` 下也能画 UI,需要浏览器节点额外匹配 `tool/ptc-dispatch` **并且**上游把 `presentationMeta` 带进该事件——这是一个有真实用例、可以向 dsh 提的小改动。

## 3.8 宿主半边检查点

- 工具描述回答了"何时用 / 何时不用 / 与谁区分 / 有无副作用"。
- `output.schema` 精确,规范返回值对程序有用(PTC 下它就是 SDK 返回类型);在 Standard 与 PTC 两个 preset 下都跑过。
- 大载荷走 `presentationMeta`,`render` 只回摘要。
- UI 意图定了(`card`/`kind`/`locations`),`presentCall` 是 `args` 的纯函数。
- 每个可调值都是 Config 字段;`apply` 开头断言;坏配置启动即炸(试一次)。
- 提示词段具名、有 order、稳定前缀、算过 tokens;大块知识考虑 skill。
- I/O 走 `ctx.fs` 等 seam;重活进 worker,`signal` 能取消。
- `pnpm run check` + `--dump-config` 看到自己的行。
