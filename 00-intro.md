# 0. 导读

## 这本手册给谁看

- 要为 DeepSeek Harness(下称 dsh)写**树外插件**(自己的仓库、自己发 npm、用户 `dsh plugin add` 安装)的开发者。
- 已经读过官方入门轨道,想知道"真实项目长什么样、坑在哪"的人。
- 需要给团队交接一个 dsh 插件项目的人。

不适合完全没接触过 dsh 的读者——请先花 30 分钟读官方 [Your first plugin](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/index.zh.md) 和 [Cordis Primer](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-primer.zh.md),再回来。

## 两个贯穿案例

手册所有结论都来自这两个插件的真实开发过程,代码引用一律注明仓库与路径。

### dsh-document(简单案例)

一个工具 `read_document(file_path, offset?, limit?, pages?)`:把 Word / PowerPoint / Excel / OpenDocument / RTF / EPUB / CSV / PDF 转成带行号的 Markdown 给模型读。

- **只有宿主半边**(Node),没有浏览器代码。
- 通过 `ctx.fs` 能力读文件(不直接碰 `node:fs`),所以自动受 dsh 的文件系统策略约束。
- Office 格式走 `@firecrawl/anydoc`(本地 Rust napi),PDF 走 `@firecrawl/pdf-inspector-wasm`,在每次调用的 `worker_threads` Worker 里跑,abort 即 terminate。
- 用 npm **trusted publishing(OIDC)** 发布,tag 触发。

它展示的是**最小完整插件**:一个 `apply(ctx, config)`、一个 Config schema、一个工具、一个 `cordis.patch.yml`、一条发布链。

### dsh-valuz-genui(复杂案例)

一个工具 `generate_ui(messages, title?)`:主模型自己写 [A2UI](https://a2ui.org) v0.9.1 文档,作为工具参数传入;浏览器把它渲染成对话里的交互式界面(图表、KPI 卡、表格、表单、仪表盘),**模型边写参数、浏览器边渲染**,界面上的点击/提交再回流给模型。

- **宿主 + 浏览器双半边**:宿主注册工具、系统提示词段、按需 skill;浏览器注册一个 Conversation Node 与渲染器。
- 依赖上游 npm 库 `@valuz/genui-core`(提示词/校验)与 `@valuz/a2ui`(76 组件目录 + 流式清洗 + React 渲染器),这两个库本身也是这次实战里从零发布到 npm 的。
- 经历过一次架构转向(嵌套模型调用 → 主模型直写),第 6 章复盘。
- 用 NPM_TOKEN 发布,GitHub Release 触发。

它展示的是**树外插件能触到的全部面**:双半边打包、纯度门、Conversation Node、会话事件模型、skill、主题、动作回流、npm 依赖管理、双仓库发布链。

## 阅读约定

- 代码引用格式:`仓库:路径:行号`,例如 `dsh-valuz-genui:src/client/node.ts:117`。
- "官方文档"指 [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) 仓库 `docs/`,链接指向中文版(`.zh.md`)。
- "宿主"= Node 进程里的 dsh(agent loop、工具、会话),"浏览器"= `dsh web` 的前端 shell。一个 bundle 可以只有宿主半边,也可以两半都有。
- 版本:dsh `0.1.0-rc.6`、pnpm 11、Node 22/24、React 18(dsh 浏览器平台)。dsh 处于 pre-release 且明确声明"foundation over blast radius"——旧格式不做兼容,升级要看变更。

## 一句话版本的核心结论

如果只记住五件事:

1. **一切都是插件,注册都是效果**:你写的每个贡献(工具、提示词段、事件监听、slot)都通过 `ctx.xxx.register()` / `ctx.effect()` / `ctx.on()` 挂上,返回的 disposer 让卸载和热替换自动正确。
2. **peer 而非 dependency 引用 dsh 包**:dsh 的包由 profile 提供一份;你若自带一份就会有两个注册表、两个 Context 类型,一切都对不上。
3. **模型可见 ⟺ 已记录**:凡是进入模型请求的东西都必须能从会话日志重建。你不能随意写自定义会话事件(见第 5 章),但平台原生通道(`user/message`、`tool/result.meta`、`assistant/chunk`)已经够用。
4. **不要在工具里嵌套调模型**:主模型的输出本身就是流式的、已记录的、一次计费的。让主模型直接产出你要的东西,工具只做校验与持久化(第 6 章有一次 200 秒失败的完整复盘)。
5. **发布链上的坑比代码多**:YAML 里 `@` 要加引号、`workspace:*` 只有 `pnpm publish` 会替换、pnpm 11 有发布冷却期和元数据缓存、`pnpm/action-setup` 与 `packageManager` 不能同时指定版本——第 8、9 章都是这类。
