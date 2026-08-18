# 8. 发布与安装

这一章是整条链:上游库怎么变得可发布 → 插件包怎么打 → 两种 CI 发布模式 → 用户怎么装 → pnpm 11 的三个坑。所有内容都是真实跑通(并且踩过)的。

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

- `files: ["lib", "cordis.patch.yml"]`;`pnpm pack` 后 tarball 应**只有** `lib/index.js`(+ `lib/client.js`)、`cordis.patch.yml`、`package.json`、`README`、`LICENSE`(genui 642 KB,含 3.5 MB 未压缩的 client.js)。
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

发布流程:bump `version` → `gh release create vX.Y.Z --target main`(**一步同时建 tag 与 Release,Release published 事件触发发布**);或 Actions 里手动 run(`workflow_dispatch`,不产生 tag)。**适合**:团队维护、多包 scope——一个 org 的 Granular Access Token(scope 限定 `@valuz`、read+write)覆盖现有和未来所有包,零逐包配置;私有包(`private: true`)`pnpm -r publish` 自动跳过。secret 建在 **Repository secrets**(不是 Environment secrets——工作流没声明 `environment:` 就读不到;Organization secrets 适合多仓库共享)。

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
5. `npm view <pkg> version` 确认;私有包应 404。
6. profile 里 `dsh plugin add <pkg>@X.Y.Z`(同天升级用精确版本),重启 web,真机过一遍第 7 章 7.5。
