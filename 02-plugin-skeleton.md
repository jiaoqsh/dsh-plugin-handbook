# 2. 树外插件骨架

这一章是"仓库长什么样"。先给单半边(dsh-document)的最小骨架,再给双半边(dsh-valuz-genui)的完整骨架,逐个文件说清楚**为什么这样写**。官方对 bundle / profile 两种清单的定义见[打包与安装插件](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/publish.zh.md)。

## 2.1 单半边骨架(dsh-document)

```
dsh-document/
├── package.json            dsh.bundle.patch → cordis.patch.yml;peers = dsh 包
├── cordis.patch.yml        往组合里 insert 一行 {id, name}
├── tsconfig.json           一个程序,noEmit,src + tests
├── tsdown.config.ts        单目标:ESM for Node,dts,一切 external
├── vitest.config.ts
├── src/
│   ├── index.ts            name / inject / Config / apply
│   ├── tool.ts             read_document 的 defineTool
│   ├── converter.ts        DocumentConverter seam + anydoc 实现 + 路由
│   ├── pdf-inspector.ts    PDF 实现:spawn worker
│   ├── pdf-worker.ts       worker 入口(按路径 spawn,不 import)
│   ├── pages.ts / window.ts
├── tests/
│   ├── read-document.spec.ts
│   └── fixtures/
└── .github/workflows/{ci,release}.yml
```

### package.json 里真正要紧的字段

```jsonc
{
  "name": "@jiaoqsh/dsh-document",
  "type": "module",
  "main": "lib/index.js",
  "types": "lib/index.d.ts",
  "exports": { ".": { "types": "./lib/index.d.ts", "default": "./lib/index.js" }, "./package.json": "./package.json" },
  "files": ["lib", "cordis.patch.yml"],          // 发布产物:预构建 lib + 补丁文件,别忘了后者
  "engines": { "node": ">=22.19" },
  "dsh": { "bundle": { "patch": "./cordis.patch.yml" } },   // 这行让它成为一个 bundle
  "scripts": {
    "build": "tsdown",
    "prepare": "tsdown",                          // git 安装时没有 lib/,靠 prepare 现场构建
    "typecheck": "tsc -p tsconfig.json",
    "test": "vitest run",
    "check": "pnpm run typecheck && pnpm run test && pnpm run build"
  },
  "peerDependencies": {                            // ← dsh 的包一律 peer
    "@deepseek-ai/cordis": "^4.0.1",
    "@deepseek-ai/dsh-fs": ">=0.1.0-rc.5",
    "@deepseek-ai/dsh-system-prompt": ">=0.1.0-rc.5",
    "@deepseek-ai/dsh-tools": ">=0.1.0-rc.5"
  },
  "dependencies": {                                // ← 只有你自己的运行时依赖
    "@deepseek-ai/schemastery": "^3.18.1",
    "@firecrawl/anydoc": "0.1.9",
    "@firecrawl/pdf-inspector-wasm": "1.14.2"
  },
  "devDependencies": {                             // ← peers 的具体版本 + 测试要的 Provider + 工具链
    "@deepseek-ai/dsh-fs": "0.1.0-rc.6",
    "@deepseek-ai/dsh-fs-local": "0.1.0-rc.6",   // 测试跑真实组合要有 Provider
    ...
  }
}
```

**为什么 dsh 包必须是 peer 而不是 dependency。** profile 的 `node_modules` 里已经有一份 dsh(`dsh-tools`、`dsh-fs`……)。如果你把 `dsh-tools` 写成 dependency,pnpm 会给你装第二份——你的 `defineTool` 来自副本 A,profile 的 `ctx.tools` 来自副本 B,类型对不上、`instanceof` 失败、注册表是两个。peer 表示"我用宿主的那一份"。profile 端把 `autoInstallPeers` 关掉(`~/.dsh/profiles/*/pnpm-workspace.yaml` 里 `autoInstallPeers: false`),就是为了不让 pnpm 替你补第二份。

peer 范围我们写成 `>=0.1.0-rc.5`(开放上限)是 pre-release 期的权宜——dsh 出稳定版后应收成 `^0.1.0`,让不兼容的宿主在安装期就报错(第 5 章末尾"宿主契约加固")。

**`prepare: tsdown` 的代价。** 用户用 `github:owner/repo#sha` 安装时,包里没有 `lib/`,pnpm 会跑 `prepare` 现场构建——但 pnpm ≥ 10 默认**禁止依赖的构建脚本**,第一次 `add` 会失败并打印一把精确的钥匙,用户要把它抄进 profile 的 `pnpm-workspace.yaml`:

```yaml
allowBuilds:
  '@jiaoqsh/dsh-document@https://codeload.github.com/jiaoqsh/dsh-document/tar.gz/<sha>': true
```

注意钥匙是 `<name>@<codeload tarball url>` 整串,写裸包名**不匹配**。从 npm 装则 tarball 自带 `lib/`,不跑 prepare、不要钥匙——这是优先发 npm 的实际理由之一。

### cordis.patch.yml

```yaml
- insert:
    - id: document-tools              # 用户在 profile 里按这个 id 覆盖你
      name: '@jiaoqsh/dsh-document'   # 加了 scope 的名字必须加引号!(YAML 里 @ 是保留起始字符)
      # config 省略 → 走 schema 缺省
```

`@` 开头的裸标量不是合法 YAML。genui 0.1.0 就是漏了这对引号,`dsh plugin add` 之后 profile 加载直接炸("bad indentation of a mapping entry"),只能发 0.1.1。**这个文件是原样发布、由宿主解析的,仓库里必须有一个测试去 parse 它**(第 7 章)。

### tsdown.config.ts(单目标)

```ts
export default defineConfig({
  entry: ['src/index.ts', 'src/pdf-worker.ts'],   // worker 按路径 spawn,所以也得是产物
  outDir: 'lib', format: ['esm'], platform: 'node', target: 'es2024',
  fixedExtension: false, dts: true, clean: true,
})
```

tsdown 默认把 `dependencies` 和 `peerDependencies` 都留作 external——正合适:运行时由 profile 提供 dsh 包,由你的 `dependencies` 提供引擎。**不要**把 dsh 包 inline 进 `lib/`,理由同 peer。

### tsconfig.json

一个程序,`moduleResolution: bundler` + `allowImportingTsExtensions`(源码里 `import './tool.ts'`),`noEmit`(产物交给 tsdown),`include: [src, tests]`。dsh 自身的规范是 `.ts` 显式扩展名的相对导入,树外插件跟着走省事。

## 2.2 双半边骨架(dsh-valuz-genui)

```
dsh-valuz-genui/
├── package.json            + exports["./client"] + dsh.client {inject, platform}
├── cordis.patch.yml
├── tsconfig.json           宿主程序:src/index.ts, tool, prompt-section, meta, tests/host
├── tsconfig.client.json    浏览器程序:src/client, meta, tests/client
├── tsdown.config.ts        双目标 + CSS 插件
├── vitest.config.ts        jsdom + react 插件;inline @valuz/*
├── css-modules.d.ts        `*.module.css` 与 `@valuz/a2ui/styles.css` 的模块声明
├── src/
│   ├── index.ts            宿主入口
│   ├── tool.ts / prompt-section.ts
│   ├── meta.ts             两半共享:tool/result.meta 的类型与守卫、<ui_action> 编解码
│   └── client/
│       ├── index.tsx       浏览器入口:注册 node + slot
│       ├── node.ts         ConversationNodeDefinition
│       ├── GenuiSurface.tsx  渲染器(用 @valuz/a2ui 的 A2UIRenderer)
│       ├── action-bridge.ts  交互 → conversation.send
│       └── partial-args.ts   从不完整 JSON 里抽完整的数组元素
├── tests/host/*.spec.ts    真实 Cordis 组合
├── tests/client/*.spec.ts(x)  jsdom
└── .github/workflows/{ci,release}.yml
```

### package.json 增加的部分

```jsonc
"exports": {
  ".":         { "default": "./lib/index.js" },     // 宿主
  "./client":  { "default": "./lib/client.js" },    // 浏览器
  "./package.json": "./package.json"
},
"dsh": {
  "bundle": { "patch": "./cordis.patch.yml" },
  "client": {
    "inject": [                                      // 浏览器 shell 里先要有这些包
      "@deepseek-ai/dsh-client-runtime",
      "@deepseek-ai/dsh-client-ui-conversation",
      "@deepseek-ai/dsh-client-ui-theme"
    ],
    "platform": "web"
  }
},
"scripts": { "typecheck": "tsc -p tsconfig.json && tsc -p tsconfig.client.json", ... },
"peerDependencies": { ...宿主 dsh 包..., "@deepseek-ai/dsh-client-runtime": ">=0.1.0-rc.5", "...ui-conversation", "...ui-slots", "...ui-theme", "react": "^18.2.0 || ^19.0.0" }
```

`dsh.client` 让 `dsh web` 知道要加载你的 `./client` 导出,`inject` 是浏览器侧的加载顺序依赖。浏览器入口自己再声明服务级 `inject`:

```ts
// dsh-valuz-genui:src/client/index.tsx
export const inject = ['conversationEvents', 'slots', 'sessions', 'theme']
export function apply(ctx: ClientContext): void { ... }
```

### 为什么是两个 tsconfig

宿主的 `@deepseek-ai/dsh-session` 与浏览器的 `@deepseek-ai/dsh-client-runtime` **都**对 cordis 的 `Context.sessions` 做了 declaration merging,含义不同。放在一个 tsc 程序里,两份增量声明互相冲突。所以:

- `tsconfig.json`(宿主):`include: [src/index.ts, src/tool.ts, src/prompt-section.ts, src/meta.ts, tests/host, css-modules.d.ts]`
- `tsconfig.client.json`(浏览器):`include: [src/client, src/meta.ts, tests/client, css-modules.d.ts]`,多 `jsx: react-jsx` 与 `dom` lib

`src/meta.ts` 是两半共享的纯类型/纯函数模块,两个程序都包含它;它**不能** import 任一平面的包。

### tsdown 双目标

宿主目标与单半边一样(ESM、Node、dsh 包 external)。浏览器目标是这套骨架里最有"dsh 味"的部分,逐项解释:

```ts
// dsh-valuz-genui:tsdown.config.ts(节选)
const CLIENT_EXTERNALS = [
  'react', 'react/jsx-runtime', 'react-dom', 'react-dom/client',
  '@deepseek-ai/cordis',
  '@deepseek-ai/dsh-client-ui-slots',
  '@deepseek-ai/dsh-client-web-react',
  '@deepseek-ai/dsh-client-ui-primitives',
  '@deepseek-ai/dsh-client-ui-attachment',
  '@deepseek-ai/dsh-client-schema-form',
  '@deepseek-ai/dsh-client-runtime/client',
]

const client: UserConfig = {
  entry: { client: 'src/client/index.tsx' },
  outDir: 'lib', format: 'cjs', platform: 'browser', target: 'es2022',
  external: CLIENT_EXTERNALS,
  noExternal: (id) => (CLIENT_EXTERNALS.includes(id) ? undefined : true),   // 纯度门:其余全部 inline
  define: {
    'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV ?? 'production'),
    'import.meta.env.MODE': ..., 'import.meta.env': ...,   // 让 Vite 系库在非 Vite 环境下也能跑
  },
  plugins: [cssPlugin],
  outputOptions: {
    entryFileNames: 'client.js',
    banner: `window.__ModuleLoader__.load({ id: ${JSON.stringify(PLUGIN_ID)}, factory: (require) => {`,
    footer: 'return module.exports; } });',
    intro: 'var module = { exports: {} }; var exports = module.exports;',
  },
}
```

- **加载器包装**:`dsh web` 不是用 `<script type=module>` 加载插件,而是执行一段 CJS,包在 `window.__ModuleLoader__.load({ id, factory: (require) => {...} })` 里;`require` 由 shell 提供,**只认平台模块**。所以 `id` 必须是你的**包名**(dsh 的 `tsdown.client.ts` 文档明写 "plugin id (package name)"),改包名时同步改。
- **纯度门**:`CLIENT_EXTERNALS` 就是 shell 会提供的全部模块;任何别的 import(a2ui、recharts、markdown-it、你自己的依赖)都得 inline 进 `client.js`。这清单是**手抄** dsh 的 `packages/client/tsdown.client.ts`——那份预设不在已发布包的导出里。宿主升级若增删平台模块,你要重构建(第 5 章末尾)。
- 结果:`client.js` 3.5 MB(recharts + A2UI 渲染器 + markdown-it 都在里面),这是当前接受的代价。

### CSS 怎么进浏览器

浏览器半边 import 了两种 CSS:CSS Modules(`./GenuiSurface.module.css`,要类名映射)和第三方整包样式(`@valuz/a2ui/styles.css`,内部还 `@import` 主题令牌和分组件样式表)。tsdown 自带的 CSS 管线不适用(依赖 `@tsdown/css`,且平台模块要求样式随 JS 一起走加载器),所以自写一个 rolldown 插件:

```ts
// dsh-valuz-genui:tsdown.config.ts(节选)
const cssPlugin = {
  name: 'genui-css-inline',
  resolveId(source, importer) {
    if (!source.endsWith('.css')) return null
    const abs = importer !== undefined && source.startsWith('.')
      ? new URL(source, `file://${importer}`).pathname
      : resolve(source)                       // 裸说明符走 node 解析(包的 exports map)
    return CSS_VIRTUAL_PREFIX + abs + CSS_VIRTUAL_SUFFIX   // '\0genui-css:' + path + '.mjs'
  },
  async load(id) {
    if (!id.startsWith(CSS_VIRTUAL_PREFIX)) return null
    const file = ...
    const isModule = file.endsWith('.module.css')
    const { code, exports: cssExports } = isModule
      ? transform({ filename: file, code: await readFile(file), cssModules: { pattern: '[hash]_[local]' }, minify: true })
      : bundle({ filename: file, minify: true })   // ← 关键:bundle() 会内联 @import,transform() 不会
    // 生成一个模块:执行时若无同 id 的 <style> 就插一个,default 导出类名映射
    return `const css = ${JSON.stringify(code.toString())}; ...
            if (document.querySelector('style[data-plugin-css="..."]') === null) { ...appendChild... }
            export default ${JSON.stringify(classMap)};`
  },
}
```

- 虚拟 id 加 `.mjs` 后缀是为了**躲开** tsdown 对 `.css` 结尾 id 的内置处理。
- `bundle()` vs `transform()` 是一个真踩过的坑:第一版用 `transform()`,A2UI 的 `styles.css` 里 `@import './themes/default.css'` 没被内联,注入的 `<style>` 里留着相对 `@import`,浏览器解析不了 → **163 个 `--va2-*` 令牌一个定义都没进来**,所有 `var(--va2-*)` 落空,Recharts tooltip 变成黑块。换成 `bundle()`(lightningcss 从磁盘内联)后修复。
- `<style>` 打 `data-plugin` 标记并按 id 幂等,是为了热替换和多次 factory 执行不重复插入。
- 类型侧要有 `css-modules.d.ts`:`declare module '*.module.css' { const classes: Record<string,string>; export default classes }` 和 `declare module '@valuz/a2ui/styles.css'`。

### 客户端依赖的两条规矩

- 你 inline 的库若声明 `react` peer 为 `^19` 而实际只用 React 18 API(`@a2ui/react` 就是),用 `pnpm-workspace.yaml` 的 `peerDependencyRules.allowedVersions` 放行,别去改上游。
- 上游库发到 npm 之前,我们用相对路径别名(tsconfig `paths` + tsdown/vitest alias)指向姊妹仓库的 `src`;发到 npm 之后**全部删掉**,改成普通 `dependencies`——第 8 章有迁移清单。别名阶段只该是过渡。

## 2.3 pnpm 侧的三个文件

- `.npmrc`:`auto-install-peers=false`(与 profile 一致;peers 的具体版本你在 `devDependencies` 里显式列)。
- `pnpm-workspace.yaml`:`allowBuilds`(pnpm 10+ 默认禁止依赖的 install 脚本,dsh dev 包的传递依赖 `koffi` 自带预编译二进制,标 `false` 拒绝其脚本即可;vitest 要 `esbuild: true`)、`minimumReleaseAgeExclude`(pnpm 11 的发布冷却期,刚发的上游版本要显式豁免,第 8 章)、`peerDependencyRules`。
- `packageManager: "pnpm@11.12.0"` 写进 package.json:CI 的 `pnpm/action-setup` 从这里读版本;**如果同时在 action 里写 `version:`,会报 `ERR_PNPM_BAD_PM_VERSION`**(两个仓库都踩了一次)。

## 2.4 骨架层面的检查点

写完骨架、还没写业务代码时,先过这几关:

1. `pnpm run check` 全绿(两个 tsc 程序 + vitest + tsdown)。
2. `pnpm pack` 看 tarball:应该**只有** `lib/`、`cordis.patch.yml`、`package.json`、`README`、`LICENSE`。
3. 用一个一次性 profile 装它:`dsh plugin --profile scratch add /absolute/path/to/repo`(`dsh plugin` 把后面的参数转发给 profile 目录里的 pnpm,并维护 `bundles` 列表),然后 `dsh --profile scratch --dump-config` 看到你的行。用完 `rm -rf ~/.dsh/profiles/scratch`。
4. 双半边的要装进带 web app 的 profile 才能看浏览器半边:`dsh plugin --profile web add /absolute/path/to/repo`,再 `dsh web`(等于 `dsh --profile web`),浏览器控制台无加载错误(第 9 章有"client.js 加载失败"的排查)。本地路径安装是 `link:`,改代码后 `pnpm run build` + 重启 `dsh web` + 硬刷新即可,不用重装。
