# 10. 清单与模板

## 10.1 新插件 checklist

### 设计(动手前)

- [ ] 一句话说清工具/能力给模型带来什么;它与 dsh 内置工具的边界(名字不撞语义,描述里写"for X use Y")。
- [ ] 需要浏览器半边吗?能用内置卡片(`generic`/`read`/`diff`…)表达就不要自定义节点。
- [ ] 需要模型在工具里再被调用吗?先想能否让主模型直出(第 6 章)。
- [ ] 需要持久化什么?对照第 5 章 5.3 的表,确认全部落在原生通道。
- [ ] 包名、scope、行 id、工具名定稿(发布后不再改)。

### 骨架

- [ ] `package.json`:`type: module`、`main`/`exports`、`files: [lib, cordis.patch.yml]`、`engines`、`dsh.bundle.patch`(+ `dsh.client`)、`prepare: tsdown`、`packageManager`、`publishConfig.access`。
- [ ] dsh 包全部 `peerDependencies`;具体版本在 `devDependencies`;测试用 Provider 在 dev。
- [ ] `cordis.patch.yml`:scoped 名加引号;id 有辨识度;注释里给覆盖示例。
- [ ] 双半边:两个 tsconfig;`PLUGIN_ID === name`;`CLIENT_EXTERNALS` 与 dsh `tsdown.client.ts` 对齐;CSS 走 `bundle()`。
- [ ] `pnpm run check` 绿;`pnpm pack` 只含预期文件;`--dump-config` 见到行。

### 宿主

- [ ] Config:每个可调值是字段;schema 缺省;`apply` 断言;坏值启动即炸(试过)。
- [ ] 工具:描述四要素;参数真实类型;`render` 摘要 / `presentationMeta` 大载荷;`presentCall` 纯函数;`isConcurrencySafe`;`exec.signal` 传到底。
- [ ] 提示词段:具名、order、稳定前缀;算过 tokens;大块知识 → skill(可选依赖用 `ctx.inject`)。
- [ ] I/O 走 seam(`ctx.fs`…);重活进 worker 且可取消。

### 浏览器

- [ ] `inject` 与 `dsh.client.inject` 完整。
- [ ] 节点:键 `turn:step`;工具过滤在 State;锚在工具事件;`animation-frame`/`immediate`;从 `tool/result` 单独可重建。
- [ ] 渲染器:ErrorBoundary;主题跟随;样式有作用域;交互只经 `conversation.send`,失败不 throw、不泄 payload。

### 测试

- [ ] 宿主:真实 Context + 真实注册表 + Provider,经 `ctx.tools.execute`;`afterEach dispose`;可选能力两向。
- [ ] 客户端:状态机纯函数用例(含重放);jsdom 渲染 + 交互。
- [ ] `cordis.patch.yml` 解析测试钉住 `{id, name === pkg.name}`。
- [ ] 真机脚本走一遍(第 7 章 7.5)。

### 发布

- [ ] `ci.yml` 与 `release.yml`(模式 A 或 B);secret 类型正确;`pnpm/action-setup` 不写 `version:`。
- [ ] README:安装(npm 主路径 + git 备选)、配置表、Known limitations / Deferred work。
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
    "./package.json": "./package.json"
  },
  "files": ["lib", "cordis.patch.yml"],
  "engines": { "node": ">=22.19" },
  "packageManager": "pnpm@11.12.0",
  "publishConfig": { "access": "public" },
  "dsh": {
    "bundle": { "patch": "./cordis.patch.yml" },
    "client": {
      "inject": ["@deepseek-ai/dsh-client-runtime", "@deepseek-ai/dsh-client-ui-conversation", "@deepseek-ai/dsh-client-ui-theme"],
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
    "@deepseek-ai/cordis": "^4.0.1",
    "@deepseek-ai/dsh-tools": ">=0.1.0-rc.5",
    "@deepseek-ai/dsh-system-prompt": ">=0.1.0-rc.5",
    "@deepseek-ai/dsh-session": ">=0.1.0-rc.5",
    "@deepseek-ai/dsh-client-runtime": ">=0.1.0-rc.5",
    "@deepseek-ai/dsh-client-ui-conversation": ">=0.1.0-rc.5",
    "@deepseek-ai/dsh-client-ui-slots": ">=0.1.0-rc.5",
    "@deepseek-ai/dsh-client-ui-theme": ">=0.1.0-rc.5",
    "react": "^18.2.0 || ^19.0.0"
  },
  "dependencies": { "@deepseek-ai/schemastery": "^3.18.1" },
  "devDependencies": {
    "@deepseek-ai/cordis": "^4.0.1",
    "@deepseek-ai/dsh-tools": "0.1.0-rc.6",
    "@deepseek-ai/dsh-system-prompt": "0.1.0-rc.6",
    "@deepseek-ai/dsh-session": "0.1.0-rc.6",
    "@deepseek-ai/dsh-llm": "0.1.0-rc.6",
    "@deepseek-ai/dsh-client-runtime": "0.1.0-rc.6",
    "@deepseek-ai/dsh-client-ui-conversation": "0.1.0-rc.6",
    "@deepseek-ai/dsh-client-ui-slots": "0.1.0-rc.6",
    "@deepseek-ai/dsh-client-ui-theme": "0.1.0-rc.6",
    "@testing-library/react": "^16.3.0", "@testing-library/dom": "^10.4.0", "jsdom": "^26.1.0",
    "@vitejs/plugin-react": "^5.1.0", "@types/node": "^22.20.0", "@types/react": "^18.3.12", "@types/react-dom": "^18.3.1",
    "react": "^18.3.1", "react-dom": "^18.3.1",
    "lightningcss": "^1.30.1", "tsdown": "^0.22.2", "typescript": "^5.9.3", "vitest": "^3.2.4", "yaml": "^2.8.0"
  }
}
```

单半边:去掉 `./client` 导出、`dsh.client`、`tsconfig.client.json`、client 相关 peers/dev,`main`/`exports` 可带 `types`。

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

const CLIENT_EXTERNALS = [                  // 对照 dsh packages/client/tsdown.client.ts
  'react', 'react/jsx-runtime', 'react-dom', 'react-dom/client',
  '@deepseek-ai/cordis',
  '@deepseek-ai/dsh-client-ui-slots',
  '@deepseek-ai/dsh-client-web-react',
  '@deepseek-ai/dsh-client-ui-primitives',
  '@deepseek-ai/dsh-client-ui-attachment',
  '@deepseek-ai/dsh-client-schema-form',
  '@deepseek-ai/dsh-client-runtime/client',
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

## 10.6 模板:`vitest.config.ts`

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

## 10.7 模板:`tests/host/patch.spec.ts`

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

## 10.8 模板:`.github/workflows/ci.yml`

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

## 10.9 模板:`.github/workflows/release.yml`

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

## 10.10 模板:README 骨架

```
# dsh-xxx
一段:给模型什么能力;建立在什么之上。
## How it works(3–5 条要点)
## Install(npm 主路径;git 备选 + allowBuilds 钥匙;重启 web + 硬刷新)
### Local development(clone → pnpm install → check → dsh plugin add /abs/path)
## The tool(签名、参数、返回)
## Configuration(行 id;字段表:Key / Default / Meaning)
## Known Limitations and Deferred Work(每条:现象 + 原因 + 计划)
## License
```
