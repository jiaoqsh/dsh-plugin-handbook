# 9. 排障速查表

每一条都是真实发生过的。按"你看到的现象"查。

## 安装与加载

| 现象 | 根因 | 修法 |
|---|---|---|
| `dsh` 启动报 `failed to parse overlay .../cordis.patch.yml: bad indentation of a mapping entry` | `name: @scope/pkg` 没加引号,`@` 是 YAML 保留起始字符 | `name: '@scope/pkg'`;仓库里加 patch 解析测试(第 7 章 7.3) |
| `dsh plugin add github:…` 首次失败,提示 allowBuilds | pnpm ≥ 10 禁止依赖的 `prepare` 构建脚本 | 把 pnpm 打印的钥匙 `'<name>@https://codeload.github.com/…/tar.gz/<sha>': true` 原样抄进 profile 的 `pnpm-workspace.yaml`;裸包名不匹配。或改用 npm 安装(自带 lib) |
| 装的是旧版本(`+ pkg ^0.1.0` 而不是 0.1.1) | 同一天先装过旧版,pnpm 11 元数据缓存里没有新版本 | `add pkg@0.1.1` 精确版本;或删 `~/Library/Caches/pnpm/v11/metadata/registry.npmjs.org/<pkg>.jsonl` |
| pnpm 提示 minimumReleaseAge / 新版本装不上 | pnpm 11 发布冷却期 | 按提示把 `<pkg>@<ver>` 加进 profile `pnpm-workspace.yaml` 的 `minimumReleaseAgeExclude` |
| 装完但 `--dump-config` 里没有你的行 | `dsh.profile.bundles` 没加上(如 `remove` 半途挂住)/ `dsh.bundle.patch` 路径不对 / patch 文件不在 `files` 里 | 看 profile `package.json` 的 `bundles`;`pnpm pack` 检查 tarball 含 `cordis.patch.yml` |
| 加载时 schema 报错指向你的行 id | 配置值不合法(这是**想要**的 fail-loud) | 修 profile 的 `cordis.patch.yml` 覆盖值 |
| `dsh-valuz-genui` 装了但 `require`/`instanceof` 报错、工具重复 | 把 dsh 包写成了 `dependencies`,装了第二份 | 改成 `peerDependencies`(devDependencies 里放具体版本) |

## 浏览器半边

| 现象 | 根因 | 修法 |
|---|---|---|
| 控制台 `Cannot find module 'xxx'`(client.js 加载失败) | 该模块不在平台外部清单里却被标成 external;或宿主升级改了平台模块 | 对照 dsh `packages/client/tsdown.client.ts` 更新 `CLIENT_EXTERNALS`;非平台模块必须 inline;重构建 |
| 改了浏览器代码不生效 | shell 不热更插件的 client.js;或 `PLUGIN_ID` 与包名不一致导致 loader 拿错模块 | `pnpm run build` → 重启 `dsh web` → **硬刷新**;`PLUGIN_ID === package.json.name` |
| 渲染的界面出现在**用户消息上面** | 节点锚在 `step/start`(低 seq),chat 按 `anchorSeq` 排 | 在 State 里记你的工具事件(首片 `tool-call-delta` 或 `tool/result`)的 seq/location,`buildViewNode` 用它做锚 |
| 引擎抛 "start after matches" | `match` 的键设计让同一 id 在 update 之后又 start | 键用 `turn:step`,工具过滤放 State(第 4 章 4.3) |
| 流式时界面不动、settle 后才出现 | 参数是转义字符串而非原生数组;或没折叠 `assistant/chunk`;或 `publication` 返回 `none` | 参数 `type:'array', items:{type:'json'}`;`match` 把 `assistant/chunk` 当 update;`animation-frame` |
| Recharts tooltip 黑块 / 任何 `var(--xxx)` 落空 | 第三方 CSS 的 `@import` 没被内联(lightningcss `transform()` 不处理 `@import`) | cssPlugin 对普通 CSS 用 `bundle()`;检查 `grep -c -- '--va2-bg:#' lib/client.js` 之类的令牌**定义**数量 |
| 亮暗切换不跟随 | 没把宿主 `colorScheme` 传进渲染器 | slot `inject` 里取 `ctx.theme.getTheme().active.colorScheme`,组件用 `useEffect` 跟随 |
| 双 React 报错(hooks 报 invalid) | 测试或打包里 React 被装了两份 | vitest `resolve.dedupe: ['react','react-dom']`;打包时 `react*` 必须在 externals |
| `@a2ui/react` peer 报 react ^19 | 库声明比实际用的 API 新 | profile/仓库 `pnpm-workspace.yaml` 的 `peerDependencyRules.allowedVersions` 放行 |
| 刷新后界面消失 | 只从流式 chunk 建状态,没处理 `tool/result` 单独到达 | 采纳 `tool/result.meta`;`anchorSeq ?? event.seq` |
| 换到 PTC 模式后自定义界面完全不出现,只剩 run_code 的文本收据 | `mode: code` 下模型只能在程序里调你的工具:流式输出的是 TypeScript 不是你的参数;子调用结果记在 `tool/code-dispatch`,**无 `meta`** | 用 Standard 或 `mode: both` 的 preset;README 写明。`code` 下支持需节点匹配 `tool/code-dispatch` + 上游给该事件带 `presentationMeta` |

## 宿主半边

| 现象 | 根因 | 修法 |
|---|---|---|
| 工具调用 ~200 s 后失败,`stream from api.deepseek.com failed`(TRANSPORT) | 工具内部嵌套了一条长模型流,网络抖动 + 整体重试叠加 | 换形状:主模型直出、工具只验(第 6 章 6.1) |
| 模型不用你的工具 / 用错工具 | 描述没说清何时用、与谁区分;名字与内置工具语义冲突 | 重写描述("use X not Y";"for A use read");名字跟模型意图(`read_document`) |
| 模型写的组件被丢 / 界面残缺 | 模型没加载 skill 就动手,字段靠猜 | 常驻段明确"先加载 skill";或 `alwaysOnFullGuide: true`;sanitizer 保留兄弟组件 |
| 工具结果把上下文撑爆 | 大载荷走了 `render` | 走 `presentationMeta`(`tool/result.meta`),`render` 只回摘要 |
| 会话打不开 / 重建被拒绝 | 你 append 了自定义事件类型(不在 `KNOWN_SESSION_EVENT_TYPES` 且不 ignorable) | 删掉;用第 5 章 5.3 的原生通道 |
| worker 在源码模式下语法错误 | worker 以 `.ts` + `--experimental-strip-types` 跑,用了 TS 专属运行时语法 | worker 文件避免 enum/参数属性等 |
| 测试后有句柄泄漏 / 第二个测试受影响 | 注册没走效果,或 worker/timer 没随 fiber 撤销 | 全部 `ctx.effect()`;`afterEach` `fiber.dispose()` 暴露它 |

## 上游库与发布

| 现象 | 根因 | 修法 |
|---|---|---|
| 首次 build 后姊妹包 `src/` 里冒出大量 `.d.ts/.js`,lint 上千错误 | `tsconfig.build.json` 继承了 `paths`,tsc 顺着 paths 一起 emit | build 配置里 `paths: {}`;清理 `git status` 里的未跟踪产物 |
| `pnpm dev` 报 `Failed to resolve import "@x/y/styles.css"` | `exports` 指向 dist,而开发时没有 dist | `exports` 指 src,`publishConfig.exports` 指 dist |
| 消费者安装报 `workspace:*` 无法解析 | 用 `npm pack`/`npm publish` 打的包 | 用 `pnpm publish` / `pnpm pack` |
| 纯 Node `import` 发布包报 ERR_MODULE_NOT_FOUND `./x` | tsc bundler 模式 emit 无扩展名 | 后处理补 `.js`(`add-js-extensions.mjs`),或 tsc 用 `nodenext` 并在源码写 `.js` |
| `.d.ts` 巨大(1 MB) | `as const` 大数组 | 数组注解为 `readonly T[]`,只在需要窄类型处 `satisfies` |
| Release 工作流挂在 setup:`ERR_PNPM_BAD_PM_VERSION` | `pnpm/action-setup` 的 `version:` 与 `packageManager` 同时指定 | 删 action 的 `version:` |
| `npm publish` 402 / 403 | scoped 包没 `publishConfig.access: public`;或 token 的 scope 不含该包 | 加 access;token 用 org Granular、scope 匹配 |
| 建了 GitHub Release 却重跑发布报版本已存在 | Release published 会触发模式 B 工作流 | 只想留档就 `git tag` + push,不建 Release |
| eslint 对 `scripts/*.mjs` 报 no-undef `process` | 脚本目录没进 lint 环境配置 | eslint ignores 加 `scripts/**`,或给它 node globals |
