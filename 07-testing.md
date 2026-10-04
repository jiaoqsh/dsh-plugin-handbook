# 7. 测试

dsh 的测试哲学(官方 [testing.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/testing.zh.md))两句话:**测试描述行为,不描述"正确性"**;**真实组合优先于 mock**——一方包的模型/用户可见行为要有无 key 的快照。树外插件没有仓库的快照基建,但同样的精神可以落地为四层:真实运行时的宿主测试、纯函数 + jsdom 的客户端测试、随包文件的解析测试、加载器冒烟。

## 7.1 宿主测试:真实 Cordis 组合

不 mock `ctx.tools`,而是把真的注册表插件装进一个真的 `Context`,只缺模型:

```ts
// dsh-document:tests/read-document.spec.ts
async function mount(cwd: string, config?: DocumentTools.Config) {
  const ctx = new Context()
  contexts.push(ctx)
  await ctx.plugin(SystemPrompt)            // @deepseek-ai/dsh-system-prompt
  await ctx.plugin(ToolRuntime)             // @deepseek-ai/dsh-tools
  await ctx.plugin(LocalFileSystem, { cwd }) // @deepseek-ai/dsh-fs-local —— 真实 Provider
  const fiber = await ctx.plugin(DocumentTools, config)
  return { ctx, fiber }
}
afterEach(async () => { await Promise.all(contexts.splice(0).map(ctx => ctx.fiber.dispose())) })

function call(ctx: Context, args: unknown, agentCwd?: string) {
  return ctx.tools.execute({                // ← agent loop 用的同一个入口
    signal: new AbortController().signal,
    callId: CallId(`call-${++calls}`),
    name: 'read_document',
    arguments: args,
    ...agentCwd === undefined ? {} : { agent: { session: { header: { cwd: agentCwd } } } as never },
  })
}
```

dsh-document 的测试仍编译在 dsh 0.1.0-rc.6 上,所以还写 `CallId`;对 0.2 编译时要改成 `ToolCallId`(第 3 章 3.1)。

要点:

- **通过 `ctx.tools.execute` 调用**,而不是直接调你的函数——参数解析、schema、错误包装、`render` 都在这条路上。
- **每个测试后 `fiber.dispose()`**:验证你的注册都是效果、能干净卸载(泄漏会在这里露馅)。
- **fixtures 进临时工作区**(`mkdtemp` + `cpSync`),`ctx.fs.resolve` 的相对路径行为可测。
- **可选能力两个方向都测**:genui 的 `mount(config, withSkills)` 一版装 `SkillRuntime`、一版不装,分别断言提示词是紧凑版还是完整版、skill 有没有注册。
- **断言模型可见文本**:`result.content` 里 `type:'text'` 拼起来,检查收据/错误信息是给模型看的样子。
- **断言持久化面**:`isGenuiSurfaceMeta(result.meta)`(或工具输出里的 `meta`)—— `tool/result.meta` 是浏览器半边的契约,宿主测试要钉住它。
- **配置失败要炸**:`ctx.plugin(Genui, { maxDocumentBytes: 0 })` 应 reject。

Skill 注册在测试里走的是真实 `SkillRuntime` 的 `validateRuntimeSkill`——它检查 `name` kebab-case、`description`、invocation,`source` 是类型要求。

## 7.2 客户端测试:状态机纯函数 + jsdom 渲染

`ConversationNodeDefinition` 是纯数据流,不需要引擎就能测(`dsh-valuz-genui:tests/client/node.spec.ts`):

```ts
// dsh-valuz-genui:tests/client/node.spec.ts:5-30(节选)
const liveChunk = (seq: number, chunk: unknown) =>
  ({ type: 'assistant/live-chunk', seq, time: 0, data: { attemptId: 'a1', turn: 2, step: 1, chunk } }) as never
const toolCall = (seq: number, args: string, name = 'generate_ui', callId = 'call-9') =>
  ({ type: 'tool/call', seq, time: 0, data: { turn: 2, step: 1, callId, name, arguments: args } }) as never
const toolResult = (seq: number, resultMeta: unknown = meta, surfaceOp = 'append') => ({
  type: 'tool/result', seq, time: 0, surfaceOp,
  data: { turn: 2, step: 1, message: { role: 'tool', source: { callId: 'call-9' } }, meta: resultMeta },
}) as never

function build(events: unknown[]) {
  const matches = events.map((event) => ({ event, location: loc }))
  let state = def.start({ key: 'k', id: 'call-9' } as never, matches[0] as never, {} as never)
  for (const match of matches.slice(1)) state = def.update({ key: 'k', id: 'call-9', state } as never, match as never)
  return def.buildViewNode?.({ key: 'k', id: 'call-9', state, start: matches[0], matches } as never) ?? null
}
```

用手工事件驱动 `match / start / update / buildViewNode`,覆盖:带名 delta 才 start、无名 delta 是 update;别的工具的 delta 与非 delta chunk 被忽略;只有 `append` 的、带 genui meta 的 `tool/result` 才是 update;流式增量拼出部分文档;settle 采纳权威文档;**live-chunk 消失后从 `tool/call` 重建**(settle 与重新打开);锚在 start 事件;没有 start 的 Context 不画。`as never` 是这里唯一妥协——事件类型很宽,测试只关心几个字段。事件字段要跟着会话格式走(第 5 章 5.2),0.1 的 `assistant/chunk` 用例到 0.2 全部重写过。

渲染器用 jsdom + Testing Library(`tests/client/surface.spec.tsx`,文件头 `// @vitest-environment jsdom`):画一份含 `Button` 的 A2UI 文档,`fireEvent.click` 后断言 `sendAction` 收到 `{surfaceId, name, component, context}`;再断言警告计数的文案、轮次关闭后过程节点缩成一行引用、turnTail 只画已 settle 的最后一版。这一条同时验证了 `@valuz/a2ui` 的 `onAction` 接线和 React 18 兼容。几个要点:

- vitest 不开 `globals` 时,Testing Library 不会自动清理 DOM。要显式写 `afterEach(cleanup)`(`surface.spec.tsx:25`),否则上一个用例的按钮还在,`getByRole` 会报找到多个。
- `A2UIRenderer` 在 `status: 'running'` 时会把还在写的最后一条消息先压着不画。断言交互要用 `status: 'success'` 的数据。
- `t` 用英文词典加一个 `{param}` 替换函数就够(`surface.spec.tsx:18-19`),不必挂真实 locale 服务。

`vitest.config.ts` 要点:`@vitejs/plugin-react`、`resolve.dedupe: ['react','react-dom']`(避免双 React)、`test.server.deps.inline: [/@a2ui\//, /@valuz\//]`(第三方包的 `styles.css` 导出要过 vite 管线,纯 Node 导不进 CSS)、`pool: 'forks'`。

## 7.3 随包发布、由宿主解析的文件:必须有解析测试

genui 0.1.0 的事故(`cordis.patch.yml` 里未加引号的 `@valuz/...`)暴露了一个盲区:仓库里没有任何东西 parse 过那个文件——它不是代码、不进 tsc、不被 import。补上的测试:

```ts
// dsh-valuz-genui:tests/host/patch.spec.ts
const patch = parse(readFileSync(new URL('../../cordis.patch.yml', import.meta.url), 'utf8'))
expect(patch).toEqual([{ insert: [{ id: 'valuz-genui', name: pkg.name }] }])
```

它同时抓两类回归:YAML 语法(引号)与**包名漂移**(`name` 必须等于 `package.json.name`,改包名忘了改 patch 就红)。同理值得钉住的:`exports` 里 `./client` 指向的文件在 `pnpm pack` 后存在;`PLUGIN_ID === package.json.name`。

## 7.4 加载器冒烟

- `dsh plugin --profile scratch add /abs/path && dsh --profile scratch --dump-config`:看到自己的行、config 缺省已填。
- 给 patch 一个坏值再 `--dump-config` / 启动:应报错并指出行 id(fail loud 的实证)。
- 用一个假的 API key 启动 web:插件应正常挂载(它不依赖模型可用),浏览器控制台无 `require` 报错。

这几步没有自动化,但每次改骨架(package.json / patch / tsdown)后手动过一遍,10 秒。

## 7.5 真机验证脚本

发布前在一个带 web app 的 profile 上按这个顺序走一遍(genui):

1. link 安装插件(见 7.6),启动,硬刷新。
2. 让模型"画一个近 7 天收入的柱状图 + 三个 KPI 卡":看**流式**(组件逐个出现)、**位置**(在用户消息之后)、**主题**(切亮暗跟随)、**tooltip 颜色**(令牌注入正确)。
3. 等这一轮结束:过程折叠成 "Completed in Ns",界面完整出现在 turnTail 里(第 4 章 4.6)。
4. 点一个按钮:模型收到 `<ui_action ...>` 并回应。
5. 刷新页面:界面从 `tool/call` + `tool/result.meta` 重放,位置不变。
6. 换 headless profile 跑一句:工具仍可用(退回 generic 卡、完整指南)。

dsh-document 的对应版本:让模型读一个 PDF 的第 1–3 页、一个 docx、一个不存在的文件、一个超大文件——分别看到内容、页标记、可操作的错误、上限拒绝。

## 7.6 开发 profile + 无头浏览器:我们实际用的验证方式

把插件装进日常用的 `web` profile 会污染它,而且每次都要手点。0.2 移植时我们留了一个专用的开发 profile,用无头 Playwright 跑 7.5:

```sh
# 在 dsh 仓库根目录。从 web 模板建一个新 profile,只打印合成配置不启动
pnpm dsh genui-check --from-default-profile web --dump-config
# link 安装本地插件(改代码后只需 build + 重启,不用重装)
pnpm dsh plugin --profile genui-check add /abs/path/to/dsh-valuz-genui
# 启动,不自动开浏览器,固定端口
pnpm -s dsh genui-check --no-open --port 4877
```

- `dsh <name>` 等价于 `dsh --profile <name>`;`--from-default-profile` 只能用在还不存在的、非内置的名字上(`deepseek-harness:apps/cli/README.md:9-20`)。这个 profile 留在 `~/.dsh/profiles/genui-check`,下次直接启动。
- Playwright 直接用 dsh 仓库里那份,按**绝对路径** import:`await import('/…/deepseek-harness/node_modules/.pnpm/playwright@1.61.1/node_modules/playwright/index.mjs')`。ESM 不看 `NODE_PATH`,裸说明符在仓库外的脚本里解析不到。
- 页面首次打开会有一个预览版提示,先点掉 "Continue"。
- 输入框是 `[contenteditable="true"]`,填字后回车发送。
- **判断最终状态前先等 `/Completed in/` 出现**。轮次没结束时过程还是展开的、turnTail 还没出现,这时截图会误判。

## 7.7 CI 里跑什么

`ci.yml`:checkout → `pnpm/action-setup`(版本来自 `packageManager`)→ `setup-node` → `pnpm install --frozen-lockfile` → `pnpm run check`(typecheck ×2 + vitest + tsdown)。发布工作流在 publish 前**再跑一次 `check`**,坏构建发不出去。

## 7.8 还没做、计划做

- **录制回放快照**:一段真实 V4 会话日志 → 按顺序喂节点(含 `tool/call` 与 `tool/result`)→ 快照视图节点。把持久化与客户端契约一次钉住(第 5 章 5.5)。
- **dsh 版本矩阵**:CI 里对最新 dsh 再跑一次 `check`,提前发现契约漂移。7.6 的无头脚本也可以搬进 CI,但它要真模型和 key。
- 覆盖率门:dsh 仓库对一方包要求 per-file 100%,树外插件目前只跑 `vitest run`;若团队要,`vitest --coverage` 加阈值即可。
