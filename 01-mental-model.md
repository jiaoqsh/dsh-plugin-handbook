# 1. 心智模型:everything is a plugin

这一章不复述 Cordis 教程,只回答"写插件时脑子里应该装着哪几张图"。概念定义请看官方 [Cordis Primer](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-primer.zh.md)、[插件与生命周期](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/framework/index.zh.md)、[服务与依赖](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/framework/service.zh.md)、[事件系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/framework/events.zh.md)、[架构](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.zh.md)。

## 1.1 一个插件在 dsh 里到底是什么

一个 ES 模块,导出四样东西(以 `dsh-valuz-genui:src/index.ts` 为例):

```ts
export const name = 'genui'                       // Cordis 里的插件名(日志、fiber 标识)
export const inject = ['tools', 'systemPrompt']   // 我需要的服务;齐了才加载
export const Config: Schema<Config> = Schema.object({ ... })   // schemastery 校验的配置
export function apply(ctx: Context, config: Config): void { ... }  // 拿到 ctx 后做注册
```

dsh 的 loader 从 `cordis.yml` 的行里读到 `name: '@valuz/dsh-valuz-genui'`,import 这个模块,用行上的 `config` 过一遍 `Config` schema(缺省值在这里填),然后在一个 Cordis fiber 里调用 `apply(ctx, config)`。`inject` 里列出的服务缺任何一个,fiber 就等着不加载;服务后来消失了,fiber 会被卸载,你在 `apply` 里做的所有注册被自动撤销。

**实战含义**:

- `apply` 里只做**注册**,不做长期运行的工作;需要后台任务时通过 `ctx.effect()` 挂上并返回清理函数。
- 配置校验分两层:schema 负责类型与缺省,`apply` 开头断言 schema 表达不了的约束(正整数等),**违反就 throw**——"misconfiguration fails loud",dsh 不接受静默兜底。两个案例都是这个模式:

```ts
// dsh-document:src/index.ts
export function apply(ctx: Context, rawConfig: Config): void {
  const config = rawConfig as ResolvedConfig
  assertPositiveInteger('maxInputBytes', config.maxInputBytes)
  ...
```

- 可选依赖不进 `inject`(会改变启动顺序、把可选变必需),而是探测或用作用域注入。genui 的 skill 是可选的:

```ts
// dsh-valuz-genui:src/index.ts
const guideAvailableAsSkill = (): boolean =>
  !config.alwaysOnFullGuide && ctx.get('skills') !== undefined   // 探测:有就用

ctx.inject(['skills'], (skillCtx) => skillCtx.skills.register({ ... }))  // 作用域注入:出现即挂、消失即撤
```

`ctx.inject(deps, cb)` 建立一个子作用域,`deps` 齐了跑 `cb`,任一服务卸载时子作用域连同其中的注册一起撤销。这是"平台有 skill 能力时多注册一个 skill,没有时也能加载"的正确写法。

## 1.2 Context 是服务仓库,`inject` 是依赖声明

`ctx.tools`、`ctx.systemPrompt`、`ctx.fs`、`ctx.llm`、`ctx.skills`、`ctx.sessions`……都是**服务**:某个插件把自己挂到 `ctx.<key>` 上,别的插件通过 key 找它,不 import 具体实现。你的插件靠 `inject` 声明要哪些 key,靠 `peerDependencies` 拿到它们的**类型**(第 2 章讲为什么必须是 peer)。

我们两个插件消费的服务:

| 平面 | 服务 | 用来 | 案例 |
|---|---|---|---|
| 宿主 | `ctx.tools` | 注册工具 | 两个都用 |
| 宿主 | `ctx.systemPrompt` | 注册提示词段(`section({name, order, text})`) | 两个都用 |
| 宿主 | `ctx.fs` | 读文件(`resolve / stat / readBytes`),受 dsh 文件策略约束 | dsh-document |
| 宿主 | `ctx.skills`(可选) | 注册运行时 skill | dsh-valuz-genui |
| 宿主 | `ctx.llm` | 调模型(`stream`) | genui v0.2 用过,v0.3 **刻意不用**(第 6 章) |
| 浏览器 | `ctx.conversationEvents` | 注册 Conversation Node 定义 | dsh-valuz-genui |
| 浏览器 | `ctx.slots` | 往具名 slot 注入渲染组件 | dsh-valuz-genui |
| 浏览器 | `ctx.sessions` | 拿到某会话的作用域服务(如 `conversation.send`) | dsh-valuz-genui |
| 浏览器 | `ctx.theme` | 读当前 colorScheme | dsh-valuz-genui |

服务的类型通过 **declaration merging** 进 `Context`:每个 dsh 包在自己的 `.d.ts` 里 `declare module '@deepseek-ai/cordis' { interface Context { tools: ToolRegistry } }`。所以你要 `import type {} from '@deepseek-ai/dsh-fs'` 这种"空 import"把增量声明拉进编译——`dsh-document:src/tool.ts` 顶部两行就是干这个的。宿主与浏览器对 `Context.sessions` 的增量声明**互相冲突**(一个是宿主会话,一个是浏览器会话代理),这就是第 2 章双 tsconfig 的由来。

## 1.3 注册都是效果

Cordis 的核心纪律:**每一项贡献都通过 `ctx.effect()` / `ctx.on()` 挂上,注册函数返回 disposer**。工具、提示词段、事件监听、slot 组件、Conversation Node 定义,全都遵守。你不需要写卸载代码——fiber 卸载或 HMR 时,所有效果按逆序撤销。

对插件作者的约束是**别绕开它**:不要在模块顶层做全局副作用(修改 `globalThis`、启动定时器、往 DOM 里塞东西),那些不会随 fiber 撤销。genui 浏览器半边注入 CSS 用的是"factory 执行时若无同 id 的 `<style>` 则插入"这种幂等方式,并把 `data-plugin` 打上标记,正是为了在热替换下可控。

## 1.4 事件:四种派发模式与两类事件

官方[事件系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/framework/events.zh.md)讲了 `emit`(广播)/ `bail`(短路)/ `serial`(顺序)/ `waterfall`(管道)。写插件时要区分两类事件:

- **Cordis 事件**:进程内实时扩展点(`agent/pre-step`、`agent/request`、`llm/stream`、`tools/pre-execute` 等)。**waterfall 监听器必须调用 `next()`**,否则链被短路——这是最容易犯的错。
- **会话事件**(`SessionEvent`):持久化到会话日志的记录(`user/message`、`step/start`、`assistant/chunk`、`assistant/message`、`tool/call`、`tool/result`、`step/end`……)。它们是**模型所见上下文的唯一来源**,也是浏览器 UI 的输入。第 5 章专讲。

我们两个插件**一个 Cordis 事件监听都没写**——工具、提示词、Conversation Node 这些注册式扩展点已经够用。这本身是个信号:能用注册式扩展点就别碰事件钩子,更别改 agent loop。

## 1.5 一次轮次的流程(你的代码在哪一格)

摘自官方架构文档:

```text
turn/start
  step/start
  entered messages → user/message
  agent/request → llm/stream → assistant/chunk* → assistant/message
  tool/call* → tools/pre-execute → tools/execute → tools/post-execute → tool/result*
  step/end
  (工具还欠一次请求,或有新输入 → 下一步)
turn/end
```

把两个插件放进去:

- 系统提示词段在每步组装时被读取(`ctx.systemPrompt.section` 的 `text` 可以是函数,**每次组装都调用**——genui 用它按 skill 能力动态选紧凑/完整指南)。
- 工具 schema 同样每步组装;`tools/execute` 阶段调用你的 `execute(args, exec)`;返回值经 `render` 变成模型可见文本,经 `presentationMeta` 变成 `tool/result.meta`(持久、不进模型上下文)。
- 模型流里的每个 chunk 先落成 `assistant/chunk`(**在工具执行之前**),同时推到浏览器——genui 的流式渲染完全建立在这个事实上。
- 浏览器 UI 上的交互经 `conversation.send(text)` 变成下一步的 `user/message`——所以模型看得见、日志里有。

## 1.6 两个平面:宿主与浏览器

一个 bundle 可以带两半:

| | 宿主半边 | 浏览器半边 |
|---|---|---|
| 运行在 | Node(dsh 进程) | `dsh web` 的浏览器 shell |
| 入口 | `main`(`lib/index.js`,ESM) | `exports["./client"]`(`lib/client.js`,CJS 包在 `window.__ModuleLoader__.load` 里) |
| 声明 | `dsh.bundle.patch` → `cordis.patch.yml` | `dsh.client { inject, platform: 'web' }` |
| Context 类型 | `@deepseek-ai/cordis` 的 `Context` | `@deepseek-ai/dsh-client-runtime/client` 的 `ClientContext` |
| 能拿到什么 | 会话、工具、模型、文件系统… | 会话事件流(只读)、slot、主题、`conversation.send` |
| 打包 | 普通 ESM,dsh 包 external | **纯度门**:只有平台模块能 `require`,其余全部 inline |

两半之间**没有私有 RPC**(typert `@Remote` 需要改仓库内包)。它们通过**会话事件**单向沟通(宿主写、浏览器读),浏览器要影响宿主只能走 `conversation.send` / `session.prompt` 这类"变成用户消息"的路。这既是限制也是保证:凡是浏览器让模型看到的东西,都被记录了。

## 1.7 能力 seam:读 dsh 代码的钥匙

dsh 把每种能力拆成三个角色:**Service Definition**(接口 + 类型)、**Service Provider**(实现)、**Consumer**(用它的人)。文件系统是 `dsh-fs`(定义)+ `dsh-fs-local`(本地实现)+ 各种工具(消费);模型是 `dsh-llm` + `dsh-llm-deepseek` + agent loop。

作为插件作者你几乎总是 **Consumer**:`inject: ['fs']` 然后用 `ctx.fs.readBytes`,不关心底下是本地盘还是 E2B 沙箱。dsh-document 就是这样拿到"自动遵守文件策略、可被沙箱替换"的免费收益。你的 `devDependencies` 里要装一个 Provider(如 `dsh-fs-local`)才能在测试里跑真实组合——第 7 章。

要写新能力(不只是消费)时再读官方[三角色能力设计](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/practice/index.zh.md)。

## 1.8 组合:profile → bundles → 行

用户运行的是一个 **profile**(`$DSH_HOME/profiles/<name>/`,如 `~/.dsh/profiles/web`),它的 `package.json` 里 `dsh.profile.bundles` 按顺序列出若干 **bundle**(npm 包)。每个 bundle 的 `cordis.patch.yml` 往组合里 `insert` 若干**行**(`{id, name, config}`),profile 自己的 `cordis.patch.yml` 最后按 `id` 覆盖。最终得到一份 `cordis.yml`,每行是一个插件实例。

```
~/.dsh/profiles/web/
├── package.json         dsh.profile.bundles: [dsh-base, dsh-web-app, @jiaoqsh/dsh-document, @valuz/dsh-valuz-genui]
├── cordis.patch.yml     用户覆盖(按 id)
├── cordis.yml           合成结果
├── pnpm-workspace.yaml  allowBuilds / minimumReleaseAgeExclude 等 pnpm 策略
└── node_modules/        dsh 的包 + 你的 bundle,只此一份
```

`dsh plugin --profile web add <pkg>` 做的就是:往 `dependencies` 加包、往 `bundles` 加名、跑 pnpm 安装。你的 `cordis.patch.yml` 里的 `id` 就是用户覆盖你配置时用的名字——起个有辨识度的(`document-tools`、`valuz-genui`),别叫 `main`。

`--dump-config` 能打印合成后的 `cordis.yml`,是验证"我的行进去了没、配置对不对"的第一手段。
