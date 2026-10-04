# 8. 发布与安装

这一章是整条链:上游库怎么变得可发布 → 插件包怎么打 → 两种 CI 发布模式 → 用户怎么装 → pnpm 11 的三个坑 → dsh 的版本兼容检查 → 插件页元数据。所有内容都是真实跑通(并且踩过)的。

## 8.1 发布前先定名字

第 6 章 6.4 的结论:scope、包名、行 id 在**第一次 `publish` 之前**定完。发布后改名 = 兼容包 + 迁移说明 + 永久的搜索混淆。`package.json` 里 `"publishConfig": { "access": "public" }`——scoped 包默认 private,不写会 402。

## 8.2 让 monorepo 里的库可发布(valuz-genui 的做法)

上游起初四个 workspace 全 private、包间用 tsconfig `paths` 直引 src、`build` 只是 `tsc --noEmit`。变成可发布走了六步,每步都是通用的:

1. **`tsconfig.build.json`**:`extends` 主配置,`noEmit:false`、`declaration:true`、`outDir: dist`、`rootDir: src`、**`paths: {}`**(关键——否则 tsc 会顺着 `paths` 把姊妹包的源码一起 emit 进你的 dist,第一次就把 a2ui 的 `.d.ts`/`.js` 吐进了 `packages/a2ui/src`,8250 个 lint 错误);exclude 测试与 demo。
2. **`publishConfig` 拆分开发/发布视图**:开发时 `main/types/exports` 指 `src/*.ts`(vite dev、vitest 直接吃源码);`publishConfig.main/types/exports` 指 `dist/*`,`pnpm publish` 时替换。这样本地 `pnpm dev` 不需要先 build——我们曾把 `exports` 直接指到 dist 导致 `pnpm dev` 报 "Failed to resolve import @…/styles.css",拆分后修复。
3. **`files: ["dist"]`**,CSS 与主题文件 `cpSync` 进 dist(tsc 不搬静态资源),`sideEffects: ["**/*.css"]` 防 tree-shake 掉样式。
4. **补 ESM 扩展名**:tsc `moduleResolution: bundler` emit 的相对导入没有 `.js`,纯 Node 解析不了(bundler 无所谓,但"正经的包"要能被 Node 直接 import)。`scripts/add-js-extensions.mjs` 走一遍 dist,按实际文件系统把 `./x` 改成 `./x.js` 或 `./x/index.js`。
5. **控 dts 体积**:`as const` 的大数组会生成巨型 `.d.ts`(a2ui 的 `catalog/index.d.ts` 1.1 MB);改成 `readonly ComponentApi[]` 注解 → 553 B。只保留真正需要窄类型的地方用 `satisfies`。
6. **用 `pnpm publish` / `pnpm pack`,不要 `npm pack`**:`workspace:*` 只有 pnpm 会替换成实际版本;`npm pack` 打出来的 tarball 里 `"@valuz/a2ui": "workspace:*"` 无法安装。

验证方法(每次改发布配置都跑):`pnpm -r build && pnpm pack` → 解包看 `dependencies` 已无 `workspace:*`、`dist/` 齐全 → 在一个空目录 `npm i <tgz>` 后用 `node --input-type=module -e "import {...} from '@valuz/genui-core'"` 纯 Node 导入冒烟。

## 8.3 插件包本身

- `files: ["lib", "cordis.patch.yml", "locale/*.json", "icon.svg"]`;`pnpm pack` 后 tarball 应**只有** `lib/index.js`(+ `lib/client.js`)、`cordis.patch.yml`、`locale/*.json`、`icon.svg`、`package.json`、`README`、`LICENSE`(genui 642 KB,含 3.5 MB 未压缩的 client.js)。后两项是插件页元数据,见 8.10。
- npm tarball 自带 `lib/` → 用户安装**不跑 `prepare`、不需要 allowBuilds**。这是相对 git 安装的主要体验优势。
- 依赖分层复查:dsh 包 peer;上游库与运行时库 dependency;测试用 Provider 与工具链 devDependency;**别留只为内联上游源码而存在的传递依赖**(切到 npm 上游后 genui 删掉了 `@a2ui/*`、recharts、zod 等 8 个)。

## 8.4 CI 发布:两种模式

两个案例用了两种,各有适用场景。

### 模式 A:npm trusted publishing(OIDC)——dsh-document

```yaml
on: { push: { tags: ['v*'] } }
permissions: { contents: write, id-token: write }
steps:
  - uses: pnpm/action-setup@v4          # 版本来自 packageManager
  - uses: actions/setup-node@v4
    with: { node-version: 22, cache: pnpm, registry-url: https://registry.npmjs.org }
  - run: npm install -g npm@latest       # trusted publishing 需要 npm ≥ 11.5.1,Node 22 自带的旧
  - name: check tag against package.json # tag 必须等于 v<version>,否则失败
  - run: pnpm install --frozen-lockfile
  - run: pnpm run check
  - run: npm publish                     # 无 token;npmjs.com 上配置信任本仓库的 release.yml
  - run: gh release create "$GITHUB_REF_NAME" --generate-notes ...
```

发布流程:bump `version` → `git tag vX.Y.Z && git push origin vX.Y.Z`。无长期 secret,provenance 自动。**适合**:单包、仓库 owner 就是 npm 包 owner(每个包要在 npm 上单独配置 trusted publisher 与仓库/工作流的绑定,首发通常要手动一次)。

### 模式 B:NPM_TOKEN——valuz-genui 与 dsh-valuz-genui

```yaml
on: { release: { types: [published] }, workflow_dispatch: }
permissions: { contents: read, id-token: write }   # id-token 仍要:--provenance
steps:
  - uses: pnpm/action-setup@v4          # 不要写 version:,见下
  - uses: actions/setup-node@v4
    with: { node-version: 22, registry-url: https://registry.npmjs.org, cache: pnpm }
  - run: pnpm install --frozen-lockfile
  - run: pnpm run check                 # 或 pnpm check + pnpm -r build(monorepo)
  - run: pnpm publish --access public --no-git-checks --provenance    # monorepo: pnpm -r publish
    env: { NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }} }
```

发布流程:bump `version` → `gh release create vX.Y.Z --target main`(**一步同时建 tag 与 Release,Release published 事件触发发布**)。`--target` 写分支名或完整 SHA;写短 SHA 会被 GitHub 拒绝,报 "target_commitish is invalid";或 Actions 里手动 run(`workflow_dispatch`,不产生 tag)。**适合**:团队维护、多包 scope——一个 org 的 Granular Access Token(scope 限定 `@valuz`、read+write)覆盖现有和未来所有包,零逐包配置;私有包(`private: true`)`pnpm -r publish` 自动跳过。secret 建在 **Repository secrets**(不是 Environment secrets——工作流没声明 `environment:` 就读不到;Organization secrets 适合多仓库共享)。

两种模式都附 provenance。选择原则:**谁维护、几个包**——个人单包 OIDC 更省心,团队多包 token 更通用。

### 两个模式共同的坑

- **`pnpm/action-setup` 里写了 `version:` 又在 `package.json` 里有 `packageManager`** → `ERR_PNPM_BAD_PM_VERSION`,首次 dispatch 直接挂在 setup 步(在 publish 之前,版本号没被占用)。二选一,推荐只留 `packageManager`。
- "Node.js 20 is deprecated" 的 annotation 来自 actions 本身,无害。
- 发布失败在 `check` 之前 → 版本号未占用,修了重跑即可;失败在 `publish` 之后半截(monorepo 发了一个没发另一个)→ 只能 bump patch。

## 8.5 用户安装的三条路

| 来源 | 命令 | 需要 | 适合 |
|---|---|---|---|
| **npm** | `dsh plugin --profile web add @valuz/dsh-valuz-genui` | 无 | 默认;预构建 lib,最快 |
| **git 按 sha** | `dsh plugin --profile web add github:valuz-ai/dsh-valuz-genui#<sha>` | profile 的 `pnpm-workspace.yaml` 加 `allowBuilds: '<name>@https://codeload.github.com/<owner>/<repo>/tar.gz/<sha>': true`(首次失败时 pnpm 会打印这把钥匙,原样抄) | 试未发布的提交 |
| **本地路径** | `dsh plugin --profile web add /abs/path` | 无(`link:`) | 开发;改代码后 `pnpm run build` + 重启 web + 硬刷新 |

`dsh plugin --profile <name> <pnpm args>` 把参数转发给 profile 目录的 pnpm 并维护 `dsh.profile.bundles`。它改的文件在 `~/.dsh/profiles/<name>/`:`package.json`(dependencies + bundles)、`pnpm-lock.yaml`、`pnpm-workspace.yaml`(策略)。安装完**重启 `dsh web` + 硬刷新**(浏览器半边由 shell 加载,不热更)。

## 8.6 pnpm 11 的三个坑(profile 侧)

1. **发布冷却期 `minimumReleaseAge`**:刚发布的版本在窗口期内不参与解析,除非 `minimumReleaseAgeExclude` 里豁免。`dsh plugin add` 时 pnpm 会提示并把 `<pkg>@<ver>` 写进 profile 的 `pnpm-workspace.yaml`(截图里 "Added 2 entries to minimumReleaseAgeExclude" 就是它)。上游库刚发、插件依赖它,也要豁免。
2. **元数据缓存导致"装到旧版本"**:同一天先装了 0.1.0、再发 0.1.1,`add @scope/pkg` 仍解析到 0.1.0——不是冷却期,是 `~/Library/Caches/pnpm/v11/metadata/registry.npmjs.org/<pkg>.jsonl` 缓存的 packument 里还没有 0.1.1(解析器看不见就选不到);加豁免不管用。**修法**:精确版本 `add @scope/pkg@0.1.1`(强制刷新元数据),或删那个 `.jsonl`。别的机器(无缓存)直接就能拿到新版。
3. **"Lockfile passes supply-chain policies (verified Ns ago)"** 只是缓存的 lockfile 校验结论,与版本解析无关,别被它带偏。

还有一条不是 pnpm 的问题:**npm registry 本身有延迟**。发布成功后几分钟内,`npm view <pkg> version` 可能还是旧版本,`add` 也解析不到新版。等几分钟,并且刚发布后一律用精确版本安装(第 2 条)。

另外:`dsh plugin … remove` 曾在 pnpm 尾部挂住并留下过期的 `bundles` 条目,手动编辑 profile `package.json` 删掉即可;`WARN Issues with peer dependencies` 在 rc 期常见,`pnpm peers check` 看一眼,一般是 dsh 内部包间的范围。

## 8.7 插件从"源码别名"切到"npm 依赖"的迁移清单

发布上游后,插件里删这些(genui 提交 `refactor: consume @valuz/genui-core + @valuz/a2ui from npm`):

- `aliases.ts`(别名表)与所有 import 它的地方;
- 两个 tsconfig 的 `baseUrl` + `paths`;
- tsdown 的 alias 插件,cssPlugin 改用 `createRequire(import.meta.url).resolve(source)` 解析裸 `.css` 说明符;
- vitest 的 `resolve.alias`,改为 `server.deps.inline: [/@valuz\//]`;
- 上游内联所需的传递依赖;
- 加 `dependencies: { "@valuz/a2ui": "^0.1.0", "@valuz/genui-core": "^0.1.0" }`;
- README 里"克隆姊妹仓库"的开发说明。

然后 `pnpm install`(注意冷却期豁免)→ `check` → 看 `lib/index.js` 的 import 列表里出现 `@valuz/*`(宿主运行时外部化)、`lib/client.js` 里**没有** `require("@valuz` (浏览器仍内联)→ 令牌定义数量不掉(第 9 章那条 CSS 坑的回归检查)。

## 8.8 一次完整发布的操作单

1. bump `package.json.version`(monorepo 两个包保持一致),提交到 main。
2. `pnpm run check`(或 `pnpm check && pnpm -r build`)本地绿;`pnpm pack` 看 tarball。
3. 模式 A:`git tag vX.Y.Z && git push origin vX.Y.Z`;模式 B:`gh release create vX.Y.Z --target main --title vX.Y.Z --notes "..."`。
4. `gh run watch <id> --exit-status`;失败看 `gh run view <id>`——判断失败在 publish 前还是后。
5. `npm view <pkg> version` 确认(可能要等几分钟,见 8.6);私有包应 404。
6. profile 里 `dsh plugin add <pkg>@X.Y.Z`(刚发布一律用精确版本),重启 web,真机过一遍第 7 章 7.5。
7. 如果这一版改了支持的 dsh 范围,同步更新 README 的兼容表(8.9)。

## 8.9 版本兼容检查:peer 范围是强制的

**从 dsh 0.1.7 起,peer 范围会被强制检查**(app-boot README 从 `dsh-v0.1.7-rc.1` 开始写入这条规则),0.2.0-rc.2 同样如此。 安装和 profile 启动都会拿插件声明的 dsh peer 范围,去比 `dsh --version` 显示的运行时版本(`deepseek-harness:apps/cli/README.md:39`):

- 只看 `@deepseek-ai/dsh` 与 `@deepseek-ai/dsh-*` 这些 peer;每个范围都必须满足;**prerelease 参与范围匹配**;没写 dsh peer 就不受约束;范围写坏了算不兼容。`engines.dsh` 不参与检查(`deepseek-harness:packages/boot/app-boot/README.md:52`)。`@deepseek-ai/cordis` 也不在检查之列。
- `dsh plugin add` 在 pnpm 运行**之前**就检查。本地路径读它自己的 `package.json`,registry 包按 pnpm 选中的版本查它声明的 peer。不兼容就拒绝,错误码 `incompatible-version`,会列出包名、版本、运行时版本和不满足的 peer(`deepseek-harness:packages/boot/plugin-manager/README.md:63,69`)。
- 启动时不兼容的插件不加载,bundle 被跳过(`app-boot/README.md:54`)。这时 `--dump-config` 里看不到它的行,因为被跳过的 bundle 不贡献任何行;反过来,profile patch 里直接写的插件行被拒时仍会出现在 `--dump-config` 里(同上)。

**例外**只能一对一地给:某个插件的精确版本 × 某个 dsh 运行时的精确版本。

```sh
dsh plugin --profile web allow-version @valuz/dsh-valuz-genui@0.2.2 --dsh-version <dsh --version 的输出> --accept-risk
dsh plugin --profile web version-exemptions                       # 列出
dsh plugin --profile web revoke-version @valuz/dsh-valuz-genui@0.2.2 --dsh-version <同上>
```

CLI 拒绝安装时会把对应的 `allow-version` 命令原样打印出来。例外存在 profile 自己的 `compatibility.json` 里;插件升级和 dsh 升级都不继承(`plugin-manager/README.md:65-69`)。

**范围怎么写。** 两种取舍:

| 写法 | 例子 | 好处 | 代价 |
|---|---|---|---|
| 封顶到一个 minor | genui:dsh 包 `>=0.2.0-rc.2 <0.3.0`,cordis `~4.0.4`(`dsh-valuz-genui:package.json`) | 不兼容的宿主在加载前就被拒,用户看到明确的错误 | dsh 每升一个 minor,插件都要发一版 |
| 只写下限 | dsh-document:`>=0.1.0-rc.5`(`dsh-document:package.json`) | 宿主升级不用发版 | 宿主改了 API 也照样加载,出问题时是运行期错误 |

API 面大的插件(尤其有浏览器半边的)建议封顶:genui 0.1.x 在 dsh 0.2 上的失败是"浏览器服务等不到",排查成本远高于一条安装期报错。只用工具、文件系统这类稳定接口的插件可以只写下限。不管哪种,README 里都放一张兼容表:

```md
| Plugin | DeepSeek Harness |
|---|---|
| 0.2.x | 0.2.x (from 0.2.0-rc.2) |
| 0.1.x | 0.1.0-rc.6 to 0.1.1 |
```

(`dsh-valuz-genui:README.md:17-24`。)

## 8.10 插件元数据:插件页上的标题、描述与图标

dsh 的插件管理页和设置里的插件清单会读包的显示元数据。字段都可选,读取时不激活插件(`deepseek-harness:docs/cookbook/adding-a-package.md:112-150`,中文版[添加包 › 插件显示元数据](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cookbook/adding-a-package.zh.md#plugin-display-metadata))。

- **标题与描述**:`locale/en.json`(必须有,作为发现入口)和 `locale/zh.json` 等,内容是 `{ "meta": { "title": "…", "description": "…" } }`。标题回退顺序是 locale `meta.title` → `package.json.name` → Cordis 插件名;描述是 locale `meta.description` → `package.json.description`。
- **导出与发布**:`exports` 加 `"./locale/*.json": "./locale/*.json"` 和 `"./package.json": "./package.json"`,`files` 里列 `locale/*.json`。
- **图标**:`package.json` 顶层 `"icon": "./icon.svg"`,文件放进 `files`。支持 SVG / PNG / JPEG / WebP,不超过 256 KiB;路径不能是绝对路径、URL 或包目录之外。
- **清单版本**:`dsh.manifestVersion: 1` 声明清单格式;`engines.dsh` 是作者声明的兼容范围,只作说明,不强制(8.9)。两者都可选(`deepseek-harness:packages/util/package-manifest/README.md:48-55`)。

genui 与 dsh-document 都按这个补齐了(`dsh-valuz-genui` 提交 `7c5f28f`、`dsh-document` 提交 `3136682`),在插件页上核对过显示效果。
