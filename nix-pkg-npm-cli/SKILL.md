---
name: nix-pkg-npm-cli
description: Use when packaging an npm/Node-distributed CLI, MCP server, or agent tool as a Nix package in the sysadmin flake (systems/nixos/pkgs/*). Covers the buildNpmPackage-vs-prebuilt decision, the two hashes (src hash, npmDepsHash) and how to compute them without network surprises, the repo's default.nix + update.py contract, and build verification. Prefer this over reaching for a global npx wrapper.
agent_created: true
---

# 把 npm 分发的 CLI / MCP server 打成 flake 包

目标是让工具以声明式方式进入 `home.packages`，而不是靠 `npx`（会写 `~/.npm/_npx`，非声明式）。

## 第 0 步：先摸清上游形态，再决定打法

不要先写 nix，先查 npm registry 的元数据。要拿到的字段：

```bash
# 包本体：dist-tags / bin / dependencies / scripts / dist.unpackedSize / fileCount
curl -s https://registry.npmjs.org/<pkg> | python3 -c "import sys,json; d=json.load(sys.stdin); v=d['dist-tags']['latest']; dd=d['versions'][v]; print('bin:',dd.get('bin')); print('deps:',dd.get('dependencies')); print('optDeps:',dd.get('optionalDependencies')); print('scripts:',dd.get('scripts')); print('unpacked:',dd['dist'].get('unpackedSize')); print(dd['dist']['integrity'])"

# 具体版本同上：https://registry.npmjs.org/<pkg>/<version>
# 文件树（查有没有预编译 .node / bin/）：https://data.jsdelivr.com/v1/packages/npm/<pkg>@<version>
```

**决策规则：**

| 上游形态 | 打法 |
|---|---|
| npm tarball 里是**预编译产物** + 依赖少 + 有 lock | 照 `pkgs/codebuddy-code` 那套：`fetchurl` 抓 tarball + 手工装配 `node_modules` |
| npm tarball 无 `package-lock.json`、依赖树宽（几十上百包） | `buildNpmPackage`，src 取 **GitHub 源码 tag**（lock 齐全），依赖树交给 `fetchNpmDeps` |
| **npm tarball 里有编译产物但没有 lock，而源码树里有 lock 却没有产物** | `buildNpmPackage`，src 取 **npm tarball** + lock 单独 vendored + **显式传 `npmDeps`**（见「tarball + vendored lock 模式」） |
| 需要 `tsc` 编译（源码只有 `src/`，无 `lib/`） | `buildNpmPackage`，**不要**加 `--omit=dev`（编译器在 devDependencies） |

先例：`mobile-mcp` 走 buildNpmPackage（无 lock、80+ 依赖、需 tsc）；
`codebuddy-code` / `zcode` / `workbuddy` 走预编译 tarball。
（tarball + vendored lock 模式目前没有在用先例，做法见下节。）

## tarball + vendored lock 模式（上游 `npm publish` 已经编好了产物时）

判断依据：`tar tzf <pkg>.tgz` 里能看到 `dist/`（或 `lib/`）且文件数很多，同时**没有** `package-lock.json`。
上游的发布流程通常是 `npm run compile` → `npm publish`，所以产物只在 registry 那份 tarball 里，
源码 tag 反而要你重装几百个 devDependencies 再跑一遍编译器。能让上游编就别自己编。

三件事必须同时做，缺一就失败：

1. **`src` 取 registry tarball**：`fetchurl { url = "https://registry.npmjs.org/<pkg>/-/<pkg>-${version}.tgz"; hash = ...; }`
   hash 是**普通文件 sha256**（`nix hash file --type sha256 --sri x.tgz`），不是 fetchzip 语义。
2. **lock 单独 vendored 并补进 src**：`npmConfigHook` 会 `diff` src 里的 lock 和 `$npmDeps/package-lock.json`，
   两边必须是同一份。发布 tarball 里没有，就从对应 tag 抓一份，`postPatch = "cp ${lockfile} package-lock.json";`
   （nixpkgs 的报错信息自己就提示了这条：「You can copy a vendored package-lock.json file via postPatch」）。
3. **显式传 `npmDeps`**：`buildNpmPackage` 默认是 `npmDeps ? fetchNpmDeps { inherit src; ... }`，
   而 src（tarball）里没有 lock，**在取依赖那一步就会以 「No lock file!」失败 —— `postPatch` 救不了，
   它跑在构建期，那时依赖已经取完了**。所以必须：

   ```nix
   let lockOnly = runCommand "x-lockfile-${version}" { } ''
         mkdir -p $out && cp ${lockfile} $out/package-lock.json
       '';
   in buildNpmPackage {
     npmDeps = fetchNpmDeps { src = lockOnly; hash = "..."; };
   }
   ```

   `fetchNpmDeps` 只从 src 读 `package-lock.json`，给一个仅含 lock 的目录就够，不必拉整个源码仓库。

另加两条：

- **前置核对**：把源码 tag 的 `package.json` 与 tarball 里的逐字段 diff 一遍
  （`dependencies` / `devDependencies` / `optionalDependencies`）。全一致才说明 vendored lock 与
  tarball 的 package.json 能过 `npm ci` 的一致性校验；有一项不同就得回到源码构建那条路。
- **`dontNpmBuild = true`**：产物已在 tarball 里。注意别被脚本名骗了 —— 上游的 `build` 脚本
  可能只生成元数据，真正的编译叫 `compile`；先 `tar xzOf pkg.tgz package/package.json` 看 scripts。


## 第 1 步：确认构建期没有隐藏网络副作用

这是能不能打成纯声明式包的关键。逐个检查依赖树的 **postinstall / install / prepare**：

先做最省事的一步：**查 lock 里的 `hasInstallScript`**，谁有脚本一目了然。

```bash
python3 -c "
import json
d=json.load(open('package-lock.json'))
for k,v in d['packages'].items():
    if v.get('hasInstallScript'):
        print(f\"{k:70} dev={v.get('dev',False)} optional={v.get('optional',False)}\")
"
```

**别以为 `--ignore-scripts` 是默认开着的 —— 它不是。** `npmConfigHook` 里只有一条命令
硬编码了这个 flag：

```sh
npm ci --ignore-scripts $npmInstallFlags …       # 只有这一步安全
patchShebangs node_modules
npm rebuild $npmRebuildFlags …                   # 这一步会跑所有包的 install 脚本！
```

而 `npmRebuildFlags ? [ ]`（`npmInstallFlags` 同样是**空**的，见
`build-npm-package/default.nix`）。所以**任何生产依赖带 install 脚本，都会在 `npm rebuild`
时真的执行**。上面那条锁文件查询就是用来提前发现它的，别等项目跑了几十分钟才在
`npm error path /build/package/node_modules/<x>` 那里撞上。

分情况处理：

- `prepare: husky`、`esbuild` 的 fallback 脚本之类 → 无害，被 `npm ci` 的
  `--ignore-scripts` 跳过，`npm rebuild` 里也通常是空转
- **optionalDependency 带下载脚本**（如 `frida` 的 `install.js` 去 GitHub 下载 toolchain）
  → 上游本来就设计成"装不上也能用"。加 `npmRebuildFlags = [ "--ignore-scripts" ]`，
  再去产物里确认那个包只剩 JS 没有原生 binding，并在文档里写明哪些功能因此不可用
- **生产依赖强依赖下载的预编译二进制** → 这包很难打干净，先告诉用户，别硬做
  （要给的话，得把那二进制单独做成 fixed-output derivation，再塞进包期望的位置）
- playwright 的 postinstall **会下载浏览器** → 别解开这个开关，本机也不需要浏览器

顺带：**预编译二进制如果走 `optionalDependencies` 按平台分发**（如
`@mobilenext/mobilecli-linux-amd64`，Go 静态链接产物）→ 干净，包管理器直接装下来，
连 `autoPatchelfHook` 都不需要（`ldd` 会报"不是动态可执行文件"）。

改完记得**去看构建日志确认**，别只看它通过了：`nix log <drv>` 里 `npm rebuild` 前后各有一条
`patching script interpreter paths in node_modules`，卡在两条之间的报错就是脚本在做事。

## 第 2 步：算两个 hash

```bash
# src hash：nur l / nix 下载走代理常失败，直接用 curl 抓再本地算（fetchzip 语义）
mkdir -p ~/.cache/<pkg>-hash && cd ~/.cache/<pkg>-hash
curl -sL -o src.tar.gz https://codeload.github.com/<owner>/<repo>/tar.gz/refs/tags/<version>
tar xzf src.tar.gz
nix hash path --type sha256 --sri ./<repo>-<version>

# npmDepsHash：扫 lock 的全部条目，必须真联网（条目多时起后台跑）
nix run nixpkgs#prefetch-npm-deps -- ./<repo>-<version>/package-lock.json
```

- 上游 tag **未必带 `v` 前缀**，先 `curl .../api.github.com/repos/<owner>/<repo>/tags` 确认
- `nurl` 在受限网络下会挂，别把它当唯一路径
- `npmDepsHash` 慢，起后台任务，别阻塞

### `prefetch-npm-deps` 会无限挂起（它不是可靠的取 hash 手段）

它是 Rust 二进制、**没有设任何超时**。网络里只要有一条连接中途断掉，它就会永久停在
`futex_do_wait`，既不报错也不退出。表现为「跑了几十分钟、一点输出都没有」。

判断它是在干活还是卡死 —— 采两次 `/proc/<pid>/io`，看 `write_bytes` 有没有增长
（注意 `nix run` 外层还有一层 bash 包着，`cmdline` 里带 lock 路径的那个才是真进程）：

```bash
P=$(pgrep -f 'prefetch-npm-deps' | tail -1)   # 或按 cmdline 里是否含 lock 文件名筛
grep write_bytes /proc/$P/io; sleep 15; grep write_bytes /proc/$P/io
```

`write_bytes` 不动 + 全线程 `futex_do_wait` + 只剩 1 个 socket = 卡死了，别等。

**更稳的做法：让 nix 自己取。** 把 `npmDeps` 的 hash 先写成一个**格式合法但错误**的 SRI
（`sha256-` + 43 个 `A` + `=`，正好 32 字节 —— `lib.fakeHash` 就是这个值），然后直接
`nix build`：`fetchNpmDeps` 走 nix 自己的 fetcher（有超时和重试），跑完会以
`hash mismatch ... specified: <你写的> got: <真值>` 的形式把真 hash 打印出来。

两个坑：

- **`sha256-PLACEHOLDER` 这类短串会在求值阶段就被拒**（`invalid SRI hash ... length 8 != expected length 32`），
  根本走不到下载那一步。必须写足长度的 SRI。
- 如果想继续用 `update.py`，**别写 `lib.fakeHash`**（它是表达式，脚本的「引号里的字符串」正则匹配不到）。
  写那个 43 个 `A` 的字面量。

## 第 3 步：写文件（按仓库契约）

位置 `systems/nixos/pkgs/<name>/`，必须同时给 `default.nix` + `update.py`（CLI 契约见
`systems/nixos/pkgs/README.md`）。`default.nix` 的关键点：

- 钉 `nodejs`：看**依赖树里最严的那条** engines 要求，不是包本体自己声明的
- **自己造入口**，不要依赖 `buildNpmPackage` 生成的 bin 的形态（它装到
  `$out/lib/node_modules/<scope>/<name>/`，bin 形态不稳定）：

  ```nix
  postInstall = ''
    makeWrapper ${nodejs}/bin/node $out/bin/<name> \
      --add-flags "$out/lib/node_modules/<scope>/<name>/lib/index.js" \
      --prefix PATH : ${lib.makeBinPath [ ... ]}
  '';
  ```

- 需要的外部命令（如 android-tools 的 adb）用 `--prefix PATH` 塞进去，而不是指望用户的 PATH
- `update.py` 抄同目录兄弟脚本的结构（严格解析 → 锚定替换 → 往返自检 → 降级/预发布提示），别另起一套

## 第 4 步：接线

- `systems/nixos/flake.nix` 的 overlay：`<name> = final.callPackage ./pkgs/<name> { };`
- `hosts/common/home/devtools/default.nix` 的 `home.packages`
- `pkgs/README.md`：带脚本的包列表要补上

## 第 5 步：验证（不要只做静态检查）

```bash
# 1. 先求值，查参数名 / 属性名写错
nix eval --impure --raw --expr 'let p = import (builtins.getFlake "nixpkgs") { system = "x86_64-linux"; }; in (p.callPackage ./systems/nixos/pkgs/<name> { }).name'

# 2. 真构建
nix build --impure --no-link --print-out-paths --expr '...同上的 callPackage...'

# 3. 跑产物入口 + 检查平台二进制
$OUT/bin/<name> --help
ldd <平台二进制>     # 静态链接则不需要 autoPatchelfHook
```

## 已知环境坑

- 本会话沙箱内：`/tmp` 解压被拦（用 `$HOME/.cache/`）；GitHub 直连不通、npm registry 通；
  nix 走 ustc 镜像可正常下载
- 沙箱里 `python3` 报 SSL CERTIFICATE_VERIFY_FAILED → 是自带 python 缺 CA，不是脚本问题：
  `SSL_CERT_FILE=$(nix eval --impure --raw --expr '... p.cacert ...') ` 绕过
- **`just switch` 的激活阶段一定会在 agent 沙箱里失败**（沙箱设了 `no_new_privs`，
  `NoNewPrivs: 1`，sudo 无法提权）。这是正常现象，不是配置错误。构建能验证，
  激活要用户在**自己的终端**里跑。别再花时间在沙箱里试提权。
