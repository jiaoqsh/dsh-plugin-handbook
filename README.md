# DeepSeek Harness 插件开发实战手册

> 基于两个真实树外插件的开发、发布与排障经验整理:**dsh-document**(一个工具 + 能力消费)与 **dsh-valuz-genui**(宿主 + 浏览器双半边、流式渲染、完整发布链)。
> 记录时间 2026-08,对应 dsh `0.1.0-rc.6` 时代;dsh 处于 pre-release,API 可能变化,以官方仓库为准。

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
| 3 | [宿主半边](03-host-half.md) | 工具、Config、系统提示词段、按需 skill、能力消费、PTC(Code Mode)下你的工具 |
| 4 | [浏览器半边](04-client-half.md) | Conversation Node 状态机、键与锚、折叠流式增量、slot、主题、动作桥 |
| 5 | [会话与事件模型](05-session-and-events.md) | SessionEventMap、`ignorable`、model-visible ⟺ logged、持久化通道 |
| 6 | [架构决策复盘](06-design-decisions.md) | 嵌套调用为何失败、DSL vs A2UI、目录进 skill、命名、依赖策略 |
| 7 | [测试](07-testing.md) | 真实运行时宿主测试、jsdom 客户端测试、加载器冒烟、随包文件解析测试 |
| 8 | [发布与安装](08-publish-and-install.md) | scope、publishConfig 拆分、pnpm publish、release.yml、`dsh plugin add` 三种来源、pnpm 11 的坑 |
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
