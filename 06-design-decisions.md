# 6. 架构决策复盘

这一章记录 genui 项目里几次真正改变方向的决策:当时的证据、备选、选择、代价。价值不在结论本身,而在"下次遇到类似形状的问题该怎么想"。

## 6.1 嵌套模型调用 → 主模型直写(v0.2 → v0.3)

**v0.2 的设计**:`generate_ui(request, data?, component_names?)` 工具内部通过 `ctx.llm.stream` 再调一次模型,让它专门生成 A2UI 文档;工具把文档写进 `tool/result.meta`,浏览器渲染。

**真机结果**:一次调用跑了 ~200 秒后失败——DeepSeek 流在中途被 TRANSPORT 层断开(`llm-deepseek adapter.ts` 的传输错误,不是 300 秒空闲超时);我们的整体重试(`maxAttempts=2 × maxContinuations=3`)把时间叠到 200 秒还是失败;此外双倍模型成本、界面要等全部生成完才出现、图表质量一般。

**诊断**:根因不是某个参数,而是**形状**——一个工具在内部维持一条很长的模型流,任何一次网络抖动都让整次调用作废,而且天然不流式(主模型看不到、浏览器看不到中间态)。

**对照**:先例插件 dsh-genui 的 `render_ui` 工具执行只要 53 ms、零模型工作;它的流式来自**主模型自己的输出**(围栏文本或工具参数)。再核对 dsh 源码:`tool-call-delta` 在工具执行之前就以 `assistant/chunk` append 并推到浏览器,客户端能折叠 `argsRaw`——**平台完整支持"工具参数流式渲染"**,而且没有 wire 大小上限(只有 256k tokens/step 与 5 分钟无 chunk 的空闲上限)。

**决定**(用户拍板):主模型直接写 A2UI,作为 `generate_ui` 的 `messages[]` 参数;工具只校验 + 持久化。删掉 `streamer.ts`、`route.ts`,`inject` 里去掉 `llm`。

**收益**:无嵌套、一次计费、天生流式、不再有 200 秒断流、界面质量反而**明显更好**(主模型带着完整对话上下文写 UI)。

**代价与缓解**:
- 主模型必须会写 A2UI → 需要常驻教学 → ~9k tokens/请求 → 6.3 的 skill 混合方案缓解到 ~3.1k。
- A2UI 比紧凑 DSL 难内联写好 → 依赖模型能力 + sanitizer 容错(坏组件被丢、兄弟保留)。

**可迁移的教训**:凡是"工具里再调一次模型生成大块内容"的设计,先问一句——能不能让主模型直接产出、工具只做验证?主模型的输出天然流式、天然被记录、天然只付一次钱。仓库里没有任何一方工具在内部调模型,这不是巧合。

## 6.2 紧凑 DSL vs 原生 A2UI

转向后有一个分岔:让模型写一种自定义的紧凑 DSL(更省 token、更好写),还是直接写 A2UI 标准消息?

选了 **A2UI,且以原生 JSON 对象数组传参**(`messages: array of json`),不是转义字符串。理由:
- 上游 `@valuz/a2ui`(76 组件目录、流式清洗器、渲染器)可以原样复用,不必再写一层 DSL → A2UI 编译器和它的测试;
- 原生 JSON 数组让"从不完整参数里抽完整元素"这条流式解析路径成立(第 4 章 4.5),转义字符串做不到;
- 标准格式对未来跨宿主(同一份 A2UI 在 valuz 自己的 server 与 dsh 都能渲染)有价值。

代价是 6.1 说的教学成本与写作难度。这是有意的取舍,不是没想到 DSL。

## 6.3 组件目录挪进按需 skill(混合而非全挪)

问题:完整 A2UI 指南 31,642 字符 ≈ 9k tokens 常驻系统提示词。

评估过三种:全部常驻(贵)、全部挪进 skill(模型可能不加载就动手,质量不可控)、**混合**。选混合:常驻一份紧凑指南(组件名 + 一句话用途 + 消息契约 + "先加载 skill"),10,851 字符 ≈ 3.1k tokens;完整字段签名进 `genui` skill 按需加载。**每请求省 ≈ 5.9k tokens**,没有 skill 能力时自动退回完整指南,`alwaysOnFullGuide` 作为逃生口。

关键实现点:提示词段 `text` 用函数,按 `ctx.get('skills')` 探测;skill 注册用 `ctx.inject(['skills'], …)` 作用域,不把 skill 变成硬依赖(headless 也能加载)。所有改动都在插件,上游 valuz-genui **零改动**——紧凑目录由 `@valuz/a2ui/catalog` 已导出的 `name` + `schema.description` 组合而成。

## 6.4 命名:工具、包、scope、行 id

- **工具名跟模型意图**:`convert_document` → `read_document`(模型把它当读);`render_ui` → `generate_ui`(用户视角是"生成界面")。描述里明确与 `read`/`read_image` 的分工。
- **上游库 scope**:同事定 `@valuz`。包名 `@valuz/a2ui`(不加 genui 前缀——它是独立渲染库,genui-core **依赖它**,是基础层)、`@valuz/genui-core`(先叫 `@valuz/genui`,后为与私有的 `genui-server` / `genui-playground` 对称改成 `genui-core`)。这个不对称是有意的、可解释的。
- **插件 npm 名**:`@valuz/dsh-valuz-genui`——npm 尾名 = GitHub 仓库裸名,与上游 `valuz-genui` 成对(`dsh-<上游名>`)。否决了 `@valuz/dsh-genui`:社区已有 `@omdsh-dev/dsh-genui`(同类功能、同名工具),撞名会长期混淆;否决了 valuz 开头的仓库名:社区惯例是 `dsh-*` 前缀便于聚类;否决无 scope:同事的 token 限定 `@valuz`。
- **cordis 行 id**:`document-tools`、`valuz-genui`——用户覆盖配置时用的名字,要有辨识度。
- 一条元教训:**没发布之前改名是零成本,发布后是永久成本**。scope/包名的讨论值得在第一次 `publish` 前完成。

## 6.5 依赖策略:先发上游库,再让插件依赖 npm

上游 valuz-genui 起初四个包全部 private、无构建产物、包间用 tsconfig `paths` 直引 src。插件有两条路:vendoring(把上游源码复制/别名进插件仓库)或先把上游发布到 npm。选**先发布**(路线 1),中间用相对路径别名过渡。

为此上游做的事(都是让一个 monorepo 包变得"可发布"的通用动作,第 8 章有细节):tsc emit 构建 + `paths:{}` 让包间引用外部化;`publishConfig` 拆分开发(指 src)与发布(指 dist);`add-js-extensions` 补 ESM 扩展名;缩 dts 体积;`pnpm publish` 替换 `workspace:*`。发布后插件删掉全部别名、改成 `dependencies`,并把只为内联上游源码而存在的传递依赖(`@a2ui/*`、recharts、zod…)从插件里去掉——它们由 `@valuz/a2ui` 自带。

教训:别名阶段只该是过渡;**发布上游是让插件仓库变简单的最快方式**。

## 6.6 明确推迟的事

- **recharts 懒加载瘦身**(`client.js` 3.5 MB → 约 1.5 MB):需要上游拆组件、插件产出独立 IIFE 资产、宿主提供路由、还有 React 实例共享的风险(先例插件的资产不需要 React,没有可抄的先例)。改动大、收益是加载时间,用户同意推迟。
- **主题令牌桥**:现在只跟随亮/暗;要把 `--dsw-alias-*` 映射到核心 `--va2-*`(背景/文字/边框/强调),图表色板在 JS 侧需渲染器配合。见 genui README。
- **插件事件持久化 / `ignorable` 写入面**:第 5 章 5.6,等真实用例。

## 6.7 沉淀成原则

1. **形状先于参数**:失败反复出现时先怀疑架构形状(嵌套流、双份 React、跨平面 RPC),再调参数。
2. **主模型直出,工具只验**:能让主模型直接产出的内容,不要在工具里再调模型。
3. **只用一等契约**:文档化事件、公开扩展点、原生持久化通道;不碰 DOM、不写自定义事件、不 inline dsh 包。
4. **量化再决定**:token(字符/3.5)、体积(pack 看 tarball)、时间(真机测)。skill 混合方案是算出来的。
5. **命名与 scope 在首次发布前定**;发布后一切改名都是兼容成本。
6. **推迟要写下来**:推迟的事和推迟的理由进 README 的 Deferred work,别留在聊天记录里。
