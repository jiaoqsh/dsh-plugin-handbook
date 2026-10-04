# 10. 清单与模板

## 10.1 新插件 checklist

### 设计(动手前)

- [ ] 一句话说清工具/能力给模型带来什么;它与 dsh 内置工具的边界(名字不撞语义,描述里写"for X use Y")。
- [ ] 需要浏览器半边吗?能用内置卡片(`generic`/`read`/`diff`…)表达就不要自定义节点。
- [ ] 需要模型在工具里再被调用吗?先想能否让主模型直出(第 6 章)。
- [ ] 需要持久化什么?对照第 5 章 5.4 的表,确认全部落在原生通道。
- [ ] 包名、scope、行 id、工具名定稿(发布后不再改)。

### 骨架

- [ ] `package.json`:`type: module`、`main`/`exports`、`files: [lib, cordis.patch.yml, locale/*.json, icon.svg]`、`engines`、`dsh.bundle.patch`(+ `dsh.client`)、`prepare: tsdown`、`packageManager`、`publishConfig.access`。
- [ ] 插件页元数据:`locale/en.json`(+ `zh.json`)、`icon`、`dsh.manifestVersion: 1`,`exports` 带 `./locale/*.json` 与 `./package.json`(第 8 章 8.10)。
- [ ] dsh 包全部 `peerDependencies`,范围想清楚封顶与否(第 8 章 8.9);具体版本在 `devDependencies`;测试用 Provider 在 dev;`@deepseek-ai/schemastery` 与 dsh 同版本。
- [ ] `cordis.patch.yml`:scoped 名加引号;id 有辨识度;注释里给覆盖示例。
- [ ] 双半边:两个 tsconfig;`PLUGIN_ID === name`;`CLIENT_EXTERNALS` 与 dsh `PLATFORM_MODULES` 对齐;CSS 走 `bundle()`。
- [ ] `pnpm run check` 绿;`pnpm pack` 只含预期文件;`--dump-config` 见到行。

### 宿主

- [ ] Config:每个可调值是字段;schema 缺省;`apply` 断言;坏值启动即炸(试过)。
- [ ] 工具:描述四要素;参数真实类型;`render` 摘要 / `presentationMeta` 大载荷;`presentCall` 纯函数;`isConcurrencySafe`;`exec.signal` 传到底。
- [ ] 提示词段:具名、order、稳定前缀;算过 tokens;大块知识 → skill(可选依赖用 `ctx.inject`)。
- [ ] I/O 走 seam(`ctx.fs`…);重活进 worker 且可取消。

### 浏览器

- [ ] 服务级 `inject`、`dsh.client.inject`、type-only import 三处对齐(第 4 章 4.1)。
- [ ] 节点:按调用 id 定键;锚取 start 事件;`animation-frame`/`immediate`;没有 live-chunk 时能从 `tool/call` + `tool/result` 重建;能处理没有 State 的 Context。
- [ ] 结果和交互输出在轮次折叠后仍可见(turnTail,第 4 章 4.6)。
- [ ] 渲染器:ErrorBoundary;主题跟随;样式有作用域;文案走 locale;交互只经 `conversation.send`,失败不 throw、不泄 payload。

### 测试

- [ ] 宿主:真实 Context + 真实注册表 + Provider,经 `ctx.tools.execute`;`afterEach dispose`;可选能力两向。
- [ ] 客户端:状态机纯函数用例(含重放);jsdom 渲染 + 交互,`afterEach(cleanup)`。
- [ ] `cordis.patch.yml` 解析测试钉住 `{id, name === pkg.name}`。
- [ ] 真机脚本走一遍(第 7 章 7.5)。

### 发布

- [ ] `ci.yml` 与 `release.yml`(模式 A 或 B);secret 类型正确;`pnpm/action-setup` 不写 `version:`。
- [ ] README:兼容表、安装(npm 主路径 + git 备选)、配置表、Known limitations / Deferred work。
- [ ] 首发后 `npm view` 确认;profile 精确版本安装真机验证。

## 10.2 模板:`package.json`(双半边)

```jsonc
{
  "name": "@scope/dsh-xxx",
  "version": "0.1.0",
  "description": "DeepSeek Harness plugin: …",
  "keywords": ["deepseek-harness", "dsh", "dsh-plugin"],
  "license": "MIT",
  "repository": { "type": "git", "url": "git+https://github.com/owner/dsh-xxx.git" },
  "type": "module",
  "main": "lib/index.js",
  "exports": {
    ".": { "default": "./lib/index.js" },
    "./client": { "default": "./lib/client.js" },
    "./package.json": "./package.json",
    "./locale/*.json": "./locale/*.json"
  },
  "files": ["lib", "cordis.patch.yml", "locale/*.json", "icon.svg"],
  "icon": "./icon.svg",
  "engines": { "node": ">=22.19", "dsh": ">=0.2.0-rc.2 <0.3.0" },
  "packageManager": "pnpm@11.12.0",
  "publishConfig": { "access": "public" },
  "dsh": {
    "manifestVersion": 1,
    "bundle": { "patch": "./cordis.patch.yml" },
    "client": {
      "inject": [
        "@deepseek-ai/dsh-client-locale",
        "@deepseek-ai/dsh-client-ui-chat",
        "@deepseek-ai/dsh-client-ui-conversation",
        "@deepseek-ai/dsh-client-ui-theme"
      ],
      "platform": "web"
    }
  },
  "scripts": {
    "build": "tsdown",
    "prepare": "tsdown",
    "typecheck": "tsc -p tsconfig.json && tsc -p tsconfig.client.json",
    "test": "vitest run",
    "check": "pnpm run typecheck && pnpm run test && pnpm run build"
  },
  "peerDependencies": {
    "@deepseek-ai/cordis": "~4.0.4",
    "@deepseek-ai/dsh-tools": ">=0.2.0-rc.2 <0.3.0",
    "@deepseek-ai/dsh-system-prompt": ">=0.2.0-rc.2 <0.3.0",
    "@deepseek-ai/dsh-session": ">=0.2.0-rc.2 <0.3.0",
    "@deepseek-ai/dsh-llm": ">=0.2.0-rc.2 <0.3.0",
    "@deepseek-ai/dsh-client-ui-chat": ">=0.2.0-rc.2 <0.3.0",
    "@deepseek-ai/dsh-client-ui-conversation": ">=0.2.0-rc.2 <0.3.0",
    "@deepseek-ai/dsh-client-ui-slots": ">=0.2.0-rc.2 <0.3.0",
    "@deepseek-ai/dsh-client-ui-theme": ">=0.2.0-rc.2 <0.3.0",
    "react": "^18.2.0 || ^19.0.0"
  },
  "dependencies": { "@deepseek-ai/schemastery": "~3.18.4" },
  "devDependencies": {
    "@deepseek-ai/cordis": "~4.0.4",
    "@deepseek-ai/dsh-tools": "0.2.0-rc.2",
    "@deepseek-ai/dsh-system-prompt": "0.2.0-rc.2",
    "@deepseek-ai/dsh-session": "0.2.0-rc.2",
    "@deepseek-ai/dsh-llm": "0.2.0-rc.2",
    "@deepseek-ai/dsh-util-values": "0.2.0-rc.2",
    "@deepseek-ai/dsh-api-session-controller": "0.2.0-rc.2",
    "@deepseek-ai/dsh-client-locale": "0.2.0-rc.2",
    "@deepseek-ai/dsh-client-store": "0.2.0-rc.2",
    "@deepseek-ai/dsh-client-ui-chat": "0.2.0-rc.2",
    "@deepseek-ai/dsh-client-ui-conversation": "0.2.0-rc.2",
    "@deepseek-ai/dsh-client-ui-renderer": "0.2.0-rc.2",
    "@deepseek-ai/dsh-client-ui-session": "0.2.0-rc.2",
    "@deepseek-ai/dsh-client-ui-slots": "0.2.0-rc.2",
    "@deepseek-ai/dsh-client-ui-theme": "0.2.0-rc.2",
    "@testing-library/react": "^16.3.0", "@testing-library/dom": "^10.4.0", "jsdom": "^26.1.0",
    "@vitejs/plugin-react": "^5.1.0", "@types/node": "^22.20.0", "@types/react": "^18.3.12", "@types/react-dom": "^18.3.1",
    "react": "^18.3.1", "react-dom": "^18.3.1",
    "lightningcss": "^1.30.1", "tsdown": "^0.22.2", "typescript": "^5.9.3", "vitest": "^3.2.4", "yaml": "^2.8.0"
  }
}
```

依据是 `dsh-valuz-genui:package.json`(v0.2.2)。单半边:去掉 `./client` 导出、`dsh.client`、`tsconfig.client.json`、client 相关 peers/dev,`main`/`exports` 可带 `types`。别忘了 `locale/en.json`:

```json
{ "meta": { "title": "Xxx Tools", "description": "One sentence on what the model gains." } }
```

## 10.3 模板:`cordis.patch.yml`

```yaml
# The layer this bundle contributes when a profile lists it. Config is
# omitted so the plugin's schema defaults apply. Override this row by id in
# your profile's cordis.patch.yml:
#
#   - id: xxx
#     name: '@scope/dsh-xxx'
#     config:
#       someLimit: 123
- insert:
    - id: xxx
      name: '@scope/dsh-xxx'
```

## 10.4 模板:`tsdown.config.ts`(双目标 + CSS)

```ts
import { readFile } from 'node:fs/promises'
import { createRequire } from 'node:module'
import { basename } from 'node:path'
import { defineConfig, type UserConfig } from 'tsdown'
import { bundle, transform } from 'lightningcss'

const resolve = createRequire(import.meta.url).resolve
const PLUGIN_ID = '@scope/dsh-xxx'          // == package.json.name

const CLIENT_EXTERNALS = [                  // 对照 dsh packages/client/web/src/platform.ts 的 PLATFORM_MODULES(0.2.0-rc.2)
  'react', 'react/jsx-runtime', 'react-dom', 'react-dom/client',
  '@deepseek-ai/cordis',
  '@deepseek-ai/dsh-client-store',
  '@deepseek-ai/dsh-client-ui-slots',
  '@deepseek-ai/dsh-client-ui-primitives',
  '@deepseek-ai/dsh-client-ui-dockkit',
]

const CSS_VIRTUAL_PREFIX = '\0plugin-css:'
const CSS_VIRTUAL_SUFFIX = '.mjs'

const cssPlugin = {
  name: 'plugin-css-inline',
  resolveId(source: string, importer: string | undefined): string | null {
    if (!source.endsWith('.css')) return null
    const abs = importer !== undefined && source.startsWith('.')
      ? new URL(source, `file://${importer}`).pathname
      : resolve(source)
    return CSS_VIRTUAL_PREFIX + abs + CSS_VIRTUAL_SUFFIX
  },
  async load(id: string): Promise<string | null> {
    if (!id.startsWith(CSS_VIRTUAL_PREFIX)) return null
    const file = id.slice(CSS_VIRTUAL_PREFIX.length, -CSS_VIRTUAL_SUFFIX.length)
    const isModule = file.endsWith('.module.css')
    const { code, exports: cssExports } = isModule
      ? transform({ filename: file, code: await readFile(file), cssModules: { pattern: '[hash]_[local]' }, minify: true })
      : bundle({ filename: file, minify: true })
    const classMap: Record<string, string> = {}
    for (const [local, exp] of Object.entries(cssExports ?? {})) classMap[local] = exp.name
    return [
      `const css = ${JSON.stringify(code.toString())};`,
      `const tagId = ${JSON.stringify(`${PLUGIN_ID}/${basename(file)}`)};`,
      "if (typeof document !== 'undefined' && document.querySelector('style[data-plugin-css=' + JSON.stringify(tagId) + ']') === null) {",
      "  const tag = document.createElement('style');",
      `  tag.dataset.plugin = ${JSON.stringify(PLUGIN_ID)};`,
      '  tag.dataset.pluginCss = tagId;',
      '  tag.textContent = css;',
      '  document.head.appendChild(tag);',
      '}',
      `export default ${JSON.stringify(classMap)};`,
    ].join('\n')
  },
}

const host: UserConfig = {
  entry: { index: 'src/index.ts' },
  outDir: 'lib', format: ['esm'], platform: 'node', target: 'es2022',
  fixedExtension: false, dts: false, clean: true,
}

const client: UserConfig = {
  entry: { client: 'src/client/index.tsx' },
  outDir: 'lib', format: 'cjs', platform: 'browser', target: 'es2022',
  dts: false, sourcemap: false, clean: false,
  external: CLIENT_EXTERNALS,
  noExternal: (id: string) => (CLIENT_EXTERNALS.includes(id) ? undefined : true),
  define: {
    'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV ?? 'production'),
    'import.meta.env.MODE': JSON.stringify(process.env.NODE_ENV ?? 'production'),
    'import.meta.env': JSON.stringify({ MODE: process.env.NODE_ENV ?? 'production' }),
  },
  plugins: [cssPlugin],
  outputOptions: {
    entryFileNames: 'client.js',
    banner: `window.__ModuleLoader__.load({ id: ${JSON.stringify(PLUGIN_ID)}, factory: (require) => {`,
    footer: 'return module.exports; } });',
    intro: 'var module = { exports: {} }; var exports = module.exports;',
  },
}

export default defineConfig([host, client])
```

## 10.5 模板:tsconfig 一对

`tsconfig.json`(宿主):

```jsonc
{
  "compilerOptions": {
    "target": "es2022", "module": "esnext", "moduleResolution": "bundler",
    "lib": ["es2023", "dom", "dom.iterable"], "jsx": "react-jsx", "types": ["node"],
    "strict": true, "noUncheckedIndexedAccess": true, "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true, "noUnusedLocals": true, "noUnusedParameters": true,
    "allowImportingTsExtensions": true, "resolveJsonModule": true, "esModuleInterop": true,
    "skipLibCheck": true, "noEmit": true
  },
  "include": ["src/index.ts", "src/tool.ts", "src/meta.ts", "tests/host", "css-modules.d.ts"]
}
```

`tsconfig.client.json`:同 compilerOptions,`include: ["src/client", "src/meta.ts", "tests/client", "css-modules.d.ts"]`。

`css-modules.d.ts`:

```ts
declare module '*.module.css' { const classes: Record<string, string>; export default classes }
declare module '@some/lib/styles.css'
```

## 10.6 模板:浏览器入口 `src/client/index.tsx`

```tsx
import type { Context } from '@deepseek-ai/cordis'
import type { SessionId } from '@deepseek-ai/dsh-session/types'
// type-only import:把各服务的 Context 增量声明拉进编译(第 4 章 4.1)
import type {} from '@deepseek-ai/dsh-api-session-controller/client'   // ctx.sessions
import type {} from '@deepseek-ai/dsh-client-locale/client'            // ctx.locale
import type {} from '@deepseek-ai/dsh-client-ui-chat/client'           // ChatNodeDataMap、turnTail slot
import type {} from '@deepseek-ai/dsh-client-ui-conversation/client'   // ctx.uiConversation
import type {} from '@deepseek-ai/dsh-client-ui-renderer/client'       // ctx.slots
import type {} from '@deepseek-ai/dsh-client-ui-session/client'        // 会话级 slot 的 sessionId
import type {} from '@deepseek-ai/dsh-client-ui-theme/client'          // ctx.theme
import { XxxNode, XxxTurnTail } from './XxxView.tsx'
import { en, NS, type XxxKey, zh } from './locales.ts'
import { xxxDefinition } from './node.ts'

declare module '@deepseek-ai/dsh-client-ui-slots' {
  interface LocaleNamespaceMap { xxx: XxxKey }
}

export const inject = ['uiConversation', 'slots', 'sessions', 'theme', 'locale']

export function apply(ctx: Context): void {
  ctx.uiConversation.events.register(xxxDefinition)
  ctx.effect(() => ctx.locale.register(NS, { zh, en }), 'xxx: dictionaries')
  ctx.slots.inject('conversation.chat.node', () => ctx.slots.register({
    name: 'conversation.chat.node', key: 'xxx-node', locale: NS,
    inject: () => ({ colorScheme: ctx.theme.getTheme().active.colorScheme }),
  }, XxxNode))
  ctx.slots.inject('conversation.chat.turnTail', () => ctx.slots.register({
    name: 'conversation.chat.turnTail', id: 'xxx-results', locale: NS,
    inject: (sessionId: SessionId) => {
      const binding = ctx.sessions.binding(sessionId)
      if (binding === undefined) throw new Error(`xxx: unknown session "${sessionId}"`)
      const chat = ctx.uiConversation.binding(binding).target('chat')
      return {
        keyedHooks: {
          results: (turn: string) => {
            const snapshot = chat.getSnapshot()
            if (snapshot === undefined) throw new Error('xxx: Chat target is unavailable')
            return snapshot.nodes.turnDataSource(Number(turn), 'xxx-node')
          },
        },
      }
    },
  }, XxxTurnTail))
}
```

节点定义的写法见第 4 章 4.2–4.5,完整实现参考 `dsh-valuz-genui:src/client/node.ts`。不需要 turnTail 的插件(结果本来就不是交互输出)删掉第二个 slot 和 `ui-session`、`api-session-controller` 两行 import。

## 10.7 模板:`vitest.config.ts`

```ts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  resolve: { dedupe: ['react', 'react-dom'] },
  test: {
    include: ['tests/**/*.spec.{ts,tsx}'],
    pool: 'forks',
    server: { deps: { inline: [/@some-lib\//] } },   // 让第三方包的 CSS 导出过 vite 管线
  },
})
```

## 10.8 模板:`tests/host/patch.spec.ts`

```ts
import { readFileSync } from 'node:fs'
import { describe, expect, it } from 'vitest'
import { parse } from 'yaml'
import pkg from '../../package.json'

describe('cordis.patch.yml', () => {
  const patch = parse(readFileSync(new URL('../../cordis.patch.yml', import.meta.url), 'utf8')) as unknown
  it('inserts the row resolving this package', () => {
    expect(patch).toEqual([{ insert: [{ id: 'xxx', name: pkg.name }] }])
  })
})
```

## 10.9 模板:`.github/workflows/ci.yml`

```yaml
name: CI
on:
  push: { branches: [main] }
  pull_request:
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4   # version comes from package.json "packageManager"
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm run check
```

## 10.10 模板:`.github/workflows/release.yml`

模式 B(NPM_TOKEN,团队 / 多包):

```yaml
name: Release
on:
  release: { types: [published] }
  workflow_dispatch:
jobs:
  publish:
    runs-on: ubuntu-latest
    permissions: { contents: read, id-token: write }
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4   # version comes from package.json "packageManager"
      - uses: actions/setup-node@v4
        with: { node-version: 22, registry-url: https://registry.npmjs.org, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm run check
      - run: pnpm publish --access public --no-git-checks --provenance   # monorepo: pnpm -r publish
        env: { NODE_AUTH_TOKEN: "${{ secrets.NPM_TOKEN }}" }
```

模式 A(OIDC,个人 / 单包):见第 8 章 8.4 的 dsh-document 版本(tag 触发、tag 与 version 校验、`npm install -g npm@latest`、`npm publish`、`gh release create`)。

## 10.11 模板:README 骨架

```
# dsh-xxx
一段:给模型什么能力;建立在什么之上。
## How it works(3–5 条要点)
## Install
### Compatibility(插件版本 × dsh 版本表;不兼容时 dsh 会拒装,见第 8 章 8.9)
(npm 主路径;git 备选 + allowBuilds 钥匙;重启 web + 硬刷新)
### Local development(clone → pnpm install → check → dsh plugin add /abs/path)
## The tool(签名、参数、返回)
## Configuration(行 id;字段表:Key / Default / Meaning)
## Known Limitations and Deferred Work(每条:现象 + 原因 + 计划)
## License
```
