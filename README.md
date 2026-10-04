# DeepSeek Harness 插件开发实战手册

> 基于两个真实树外插件的开发、发布与排障经验整理:**dsh-document**(一个工具 + 能力消费)与 **dsh-valuz-genui**(宿主 + 浏览器双半边、流式渲染、完整发布链)。
> 初版写于 2026-08-18,对应 dsh `0.1.0-rc.6`;2026-10-04 按 dsh **`0.2.0-rc.2`**(仓库 HEAD `639ed01539`)全面更新,对应 dsh-valuz-genui v0.2.2、dsh-document v0.1.2。dsh 处于 pre-release,API 可能变化,以官方仓库为准。
>
> 从 0.1.x 升级过来的读者先看:浏览器半边换成了 `uiConversation`(第 4 章)、会话格式升到 v4 且 `assistant/chunk` 已不存在(第 5 章)、安装和启动都会强制检查 peer 版本(0.1.7 起,第 8 章 8.9)。

## 定位

官方仓库 [`docs/user/develop/`](https://github.com/deepseek-ai/deepseek-harness/tree/master/docs/user/develop) 已有一条入门轨道(first plugin / tool / config / publish + lifecycle / service / events + 三角色能力 / LLM adapter),讲**概念与最小示例**。本手册不重复它,而是回答另一个问题:**从零做出一个真实可发布的树外插件,一路上会遇到什么、怎么决策、坑在哪里。**

- 概念处链接官方文档(中文版),不复述。
- 每个结论都能追溯到两个案例仓库里的具体代码。
- 踩过的坑写成"症状 → 根因 → 修法"。

## 目录

| 章 | 文件 | 一句话 |
|---|---|---|
| 0 | [导读](00-intro.md) | 读者、阅读路径、两个案例、约定 |
| 1 | [心智模型](01-mental-model.md) | everything is a plugin:Context / inject / effect、服务、事件、生命周期、宿主与浏览器两个平面 |
| 2 | [树外插件骨架](02-plugin-skeleton.md) | 目录、package.json、cordis.patch.yml、双 tsconfig、tsdown 双目标、CSS 注入 |
| 3 | [宿主半边](03-host-half.md) | 工具、Config、系统提示词段、按需 skill、能力消费、PTC 模式下你的工具 |
| 4 | [浏览器半边](04-client-half.md) | `uiConversation` 注册、Conversation Node 状态机、按调用 id 定键、`assistant/live-chunk` 流式、轮次折叠与 turnTail、slot、主题、动作桥 |
| 5 | [会话与事件模型](05-session-and-events.md) | 会话格式 v4、封闭的事件词汇表与 `ignorable`、model-visible ⟺ logged、持久化通道 |
| 6 | [架构决策复盘](06-design-decisions.md) | 嵌套调用为何失败、DSL vs A2UI、目录进 skill、命名、依赖策略 |
| 7 | [测试](07-testing.md) | 真实运行时宿主测试、jsdom 客户端测试、加载器冒烟、随包文件解析测试、开发 profile + Playwright 真机验证 |
| 8 | [发布与安装](08-publish-and-install.md) | scope、publishConfig 拆分、pnpm publish、release.yml、`dsh plugin add` 三种来源、pnpm 11 的坑、版本兼容检查、插件元数据 |
| 9 | [排障速查表](09-troubleshooting.md) | 症状 → 根因 → 修法 |
| 10 | [清单与模板](10-checklist-and-templates.md) | 新插件 checklist、可复制模板 |

## 案例仓库

| 仓库 | npm | 复杂度 | 覆盖 |
|---|---|---|---|
| [jiaoqsh/dsh-document](https://github.com/jiaoqsh/dsh-document) | `@jiaoqsh/dsh-document` | 简单 | 单一工具、`ctx.fs` 能力消费、worker_threads 跑 wasm、OIDC 发布 |
| [valuz-ai/dsh-valuz-genui](https://github.com/valuz-ai/dsh-valuz-genui) | `@valuz/dsh-valuz-genui` | 复杂 | 宿主 + 浏览器双半边、流式渲染、Conversation Node、skill、npm 依赖上游库、NPM_TOKEN 发布 |
| [valuz-ai/valuz-genui](https://github.com/valuz-ai/valuz-genui) | `@valuz/a2ui` `@valuz/genui-core` | — | 插件依赖的提供者无关上游库(monorepo 发布) |

## 阅读路径

- **第一次写 dsh 插件**:0 → 1 → 2 → 3 → 7 → 8,需要浏览器渲染再读 4、5。
- **已经会写,来查坑**:9 → 10。
- **要理解为什么这么设计**:6。

## 许可证

本手册以 [CC BY 4.0](LICENSE) 发布：可以转载、改编和商用，请注明出处（[jiaoqsh/dsh-plugin-handbook](https://github.com/jiaoqsh/dsh-plugin-handbook)）。手册中的代码片段和模板可以直接复制到你的项目里使用。
