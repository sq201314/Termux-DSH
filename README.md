# Termux 手动安装 DSH 完整流程与运行时修复手册

> 适用环境：Termux aarch64 / Android 11+（API 30+）  
> 已验证版本：Node.js v24.18.0、npm 11.21.1、dsh 0.2.1-rc.2、pnpm 12.14.0  
> 设备实测：`android | arm64 | v24.18.0`，内核 `Linux 4.9.186 aarch64 Android`  
> 目标：不使用第三方预编译包，基于官方 npm 包手动编译、修补，最终 `dsh web` 完整可用（含终端、grep/glob、沙箱、shell 选择）

---

## 目录

- [0. 摘要与总览](#0-摘要与总览)
- [1. 环境事实与重要原则](#1-环境事实与重要原则)
- [2. 阶段 A：从零安装 Node 与工具链](#2-阶段-a从零安装-node-与工具链)
- [3. 阶段 B：原生模块编译（node-pty / koffi / sharp）](#3-阶段-b原生模块编译node-pty--koffi--sharp)
- [4. 阶段 C：绕过 Android 不支持的原生绑定（require-builtin / flock）](#4-阶段-c绕过-android-不支持的原生绑定require-builtin--flock)
- [5. 阶段 D：绕过 Android 禁止的系统调用（link → rename）](#5-阶段-d绕过-android-禁止的系统调用link--rename)
- [6. 阶段 E：启动方式固化（shebang / 包装脚本 / 符号链接）](#6-阶段-e启动方式固化shebang--包装脚本--符号链接)
- [7. 阶段 F：禁用无法加载的实验插件（cot-translation）](#7-阶段-f禁用无法加载的实验插件cot-translation)
- [8. 阶段 G：运行时问题 1 — terminal inspection unsupported](#8-阶段-g运行时问题-1--terminal-inspection-unsupported)
- [9. 阶段 H：运行时问题 2 — ripgrep launch failed](#9-阶段-h运行时问题-2--ripgrep-launch-failed)
- [10. 阶段 I：运行时问题 3 — sandbox workspace-write 无后端](#10-阶段-i运行时问题-3--sandbox-workspace-write-无后端)
- [11. 阶段 J：运行时问题 4 — cmd 被当成 shell，退出码 20](#11-阶段-j运行时问题-4--cmd-被当成-shell退出码-20)
- [12. 过程中遇到的其他问题与解决](#12-过程中遇到的其他问题与解决)
- [13. 全部补丁与验证总结](#13-全部补丁与验证总结)
- [14. 局域网访问](#14-局域网访问)
- [15. 文件、备份与回滚](#15-文件备份与回滚)
- [16. 升级后如何重新应用](#16-升级后如何重新应用)
- [17. Android 上的其他限制（非本次 bug）](#17-android-上的其他限制非本次-bug)
- [18. 常见问题速查表](#18-常见问题速查表)
- [附录 A：全部对话记录](#附录-a全部对话记录)
- [附录 B：独立审计报告](#附录-b独立审计报告)
- [附录 C：验证输出原文](#附录-c验证输出原文)

---

## 0. 摘要与总览

本手册记录了在 Android/Termux 下从零手动安装 DSH，并将其修补到可用状态的全过程。安装和修补分成 **10 个阶段（A~J）**，其中前 7 个（A~G）是"从无到能启动"的编译与桩替换，后 4 个（H~J 加一些附录场景）是"启动后的运行时问题"修复。

### 0.1 全部问题清单

| # | 阶段 | 用户可见问题 | 根因 | 状态 |
|---|---|---|---|---|
| 1 | B | `koffi` 编译报 `statx` / `__u32` | 编译目标低于 Android API 30 | ✅ 用 `-target aarch64-linux-android30` 修复 |
| 2 | B | `node-pty` 报 `android_ndk_path` 未定义 | Termux 下 `gyp` 未定义该变量 | ✅ 修补 `$PREFIX/include/node/common.gypi` |
| 3 | B | `sharp` 加载失败 `Could not load the "sharp" module` | 无 Android 预编译包 | ✅ 系统 `libvips` + `node-gyp configure` + `make` 原生编译 |
| 4 | C | `No usable native binding found for node-addon-require-builtin-android-arm64` | 该包无 Android 原生绑定 | ✅ 写 Android 桩：`requireBuiltin(id) → require(id)` |
| 5 | C | `flock is not supported on android-arm64` | Android 不支持 `flock(2)` 系统调用 | ✅ 纯 JS 进程内软锁替换 `flock.js` |
| 6 | D | `EACCES: permission denied, link '...tmp' -> '...zstd'` | Android SELinux 禁止 `link()` | ✅ `link` → `rename` 全面替换 |
| 7 | E | `ERR_MODULE_NOT_FOUND @deepseek-ai/dsh-app-boot` | `dsh` 是普通文件，模块解析基准错误 | ✅ 改为符号链接指向包装脚本 |
| 8 | E | `--expose-internals is not allowed in NODE_OPTIONS` | Node 禁止该标志进 NODE_OPTIONS | ✅ 用命令行参数 + 包装脚本 |
| 9 | F | `cot-translation: failed to import` | 实验插件不兼容 Android | ✅ 移走目录 + 删除 bundle patch 条目 |
| 10 | G | **`终端错误：subprocess-local: terminal inspection is unsupported on platform android`** | `createProcessInspector()` 只认 linux/darwin/win32 | ✅ 3 处平台判断扩展 `android → linux`，**实测终端可用** |
| 11 | H | **`grep could not start its search command (ripgrep launch failed)`** | `@vscode/ripgrep` 无 `android-arm64` 二进制 | ✅ 系统 `rg` + PATH 回退，**实测 grep/glob 可用** |
| 12 | I | **`sandbox mode "workspace-write" is requested but no sandbox backend is usable`** | Android 无 bubblewrap/Landlock | ✅ profile 持久化 `danger-full-access` |
| 13 | J | **`进程已退出（20）cmd: No service specified`** | `$PREFIX/bin/cmd`（Android 服务管理器）被当成 shell | ✅ POSIX 过滤 `cmd`/`powershell` + profile 限定候选 |

### 0.2 关键约束（务必先读）

1. **Termux 绝对不要用 nvm**。`nvm` 与 `$PREFIX` 冲突，会出现 `nvm is not compatible with "PREFIX"`、`No such file or directory`。用 `pkg install nodejs-lts`。
2. **需要 Android 11 / API 30 或更高**。`koffi` 依赖的 `statx()` 在 API 30 才可用。
3. **不要用 `NODE_OPTIONS=--expose-internals`**。Node 会报 `--expose-internals is not allowed in NODE_OPTIONS`。
4. **DSH 无法使用 `--host 0.0.0.0` CLI 参数**。CLI 层有意禁用，需通过插件或 profile 修改。
5. **profile 比代码补丁更持久**。`npm i -g @deepseek-ai/dsh` 会覆盖 `node_modules`，但 `~/.dsh/profiles/` 不受影响。

### 0.3 路径变量速查

每次新终端建议先设置：

```bash
export PREFIX=/data/data/com.termux/files/usr
export NPM_GLOBAL_ROOT=$(npm root -g)
export DSH_ROOT="$NPM_GLOBAL_ROOT/@deepseek-ai/dsh"
export DSH_MOD="$DSH_ROOT/node_modules/@deepseek-ai"
echo "PREFIX=$PREFIX"
echo "DSH_ROOT=$DSH_ROOT"
```

典型值：

```
PREFIX=/data/data/com.termux/files/usr
DSH_ROOT=/data/data/com.termux/files/usr/lib/node_modules/@deepseek-ai/dsh
DSH_MOD=/data/data/com.termux/files/usr/lib/node_modules/@deepseek-ai/dsh/node_modules/@deepseek-ai
```

---

## 1. 环境事实与重要原则

### 1.1 排查前的环境确认

```bash
node -p "process.platform + ' | ' + process.arch + ' | ' + process.version"
# android | arm64 | v24.18.0

ls /proc/self/stat          # /proc/self/stat（可用）
which ps                    # $PREFIX/bin/ps
uname -a                    # Linux ... aarch64 Android
echo "TMPDIR=$TMPDIR"       # $PREFIX/tmp（无 /tmp）
ls -l $PREFIX/bin/bash /bin/sh
# bash 在 $PREFIX/bin/bash；/bin/sh → /system/bin（Android system-as-root）
```

结论：

- **Node 报 `android`，但内核是 Linux**。`/proc`、`/proc/<pid>/stat`、进程组、pty 都可用。
- **没有 `/tmp`**，临时目录是 `$PREFIX/tmp`。
- `node-pty` 本机可用（有本地编译的 `build/Release/pty.node`）：

```bash
node -e "const pty=require('node-pty'); const p=pty.spawn('/bin/sh',['-c','echo PTY_OK'],{cols:80,rows:24}); p.onData(d=>process.stdout.write(d)); p.onExit(()=>{})"
# PTY_OK
```

### 1.2 手动编译总原则

- **每处修改前先备份**：`cp 文件 文件.bak` 或 `cp 文件 文件.androidorig`。
- **所有原生命令都用 API 30 目标**：`CFLAGS/CXXFLAGS=-target aarch64-linux-android30`。
- **不要用 `sed 's/\.link(/.rename(/g'` 一招打天下**。有些调用是 `import { link }`、`link,` 简写属性、`linkSync(` 等形式，`sed` 模式必须精准。
- **改完必须验证**：`node --check`、`node -e "require('./')"`、`grep -n` 确认残留。

---

## 2. 阶段 A：从零安装 Node 与工具链

### 2.1 清理旧环境

如果之前用 nvm 安装过 Node，先清理：

```bash
npm uninstall -g @deepseek-ai/dsh 2>/dev/null || true
npm cache clean --force
rm -rf ~/.cache/node-gyp
```

如遇 nvm 报错：

```
nvm is not compatible with the "PREFIX" environment variable: currently set to "/data/data/com.termux/files/usr"
Run `unset PREFIX` to unset it.
```

**不要执行 `unset PREFIX`**（会破坏 Termux 自身），直接放弃 nvm。

### 2.2 安装 Termux 官方 LTS Node

```bash
pkg update -y
pkg upgrade -y
pkg install -y nodejs-lts
```

验证：

```bash
node -v   # v24.18.0
npm -v    # 11.21.1
```

> `nodejs-lts` 当前为 Node 24.18.0，满足 DSH 对 ≥22.18.0 的要求。

### 2.3 安装编译工具链

```bash
pkg install -y binutils make pkg-config clang python cmake patch git proot which ndk-multilib libandroid-spawn
pkg install -y libvips
```

安装全局 `node-gyp`：

```bash
npm install -g node-gyp
node-gyp --version   # v13.1.0
```

### 2.4 安装 pnpm（安装局域网插件时用）

```bash
npm uninstall -g pnpm 2>/dev/null || true
npm install -g pnpm --allow-scripts=pnpm
pnpm --version   # 12.14.0
```

> **重要**：npm 11+ 默认阻止安装脚本执行。`pnpm` 的 preinstall/postinstall 必须通过 `--allow-scripts=pnpm` 放行，否则 `dsh plugin` 会失败。

---

## 3. 阶段 B：原生模块编译（node-pty / koffi / sharp）

### 3.1 先触发 node-gyp 缓存

执行一次失败安装（预期失败），让 node-gyp 下载头文件：

```bash
CFLAGS="-target aarch64-linux-android30" \
CXXFLAGS="-target aarch64-linux-android30" \
npm install -g @deepseek-ai/dsh
```

即使失败也没关系，继续。

### 3.2 修补 `node-pty` 的 gyp 配置

**症状**：`node-pty` 编译报 `android_ndk_path` 未定义。

**修补**：编辑系统级 `common.gypi`，插入空变量定义：

```bash
sed -i "/variables {/a\\    'android_ndk_path%': ''," "$PREFIX/include/node/common.gypi"

# 验证
grep -A 3 "variables {" "$PREFIX/include/node/common.gypi"
```

> 该修补与 Node 大版本相关。升级 Node 后需要重新执行。

**手动编译 `node-pty`**：

```bash
find "$NPM_GLOBAL_ROOT" -type d -name "node-pty" 2>/dev/null
# 典型路径：$DSH_ROOT/node_modules/node-pty

cd "$DSH_ROOT/node_modules/node-pty"
npx node-gyp rebuild
```

成功标志：

```
gyp info ok
```

验证：

```bash
ls -l build/Release/pty.node
```

如果仍报 `android_ndk_path` 未定义，创建用户级 gyp 覆盖：

```bash
mkdir -p ~/.gyp
echo "{'variables': {'android_ndk_path': ''}}" > ~/.gyp/include.gypi
cd "$DSH_ROOT/node_modules/node-pty"
npx node-gyp rebuild
```

### 3.3 安装 dsh 主包并编译 `koffi`

```bash
CFLAGS="-target aarch64-linux-android30" \
CXXFLAGS="-target aarch64-linux-android30" \
npm install -g @deepseek-ai/dsh
```

验证：

```bash
dsh --version   # 0.2.1-rc.2
```

**编译 `koffi`（`statx` 关键）**：

```bash
cd "$DSH_ROOT/node_modules/koffi"
rm -rf build
export CC="clang --target=aarch64-linux-android30"
export CXX="clang++ --target=aarch64-linux-android30"
node ./cnoke.cjs -P . -D src/koffi --prebuild --release

# 验证
find build -name "*.node" 2>/dev/null
node -e "const koffi = require('./'); console.log('koffi loaded OK')"
# koffi loaded OK
```

### 3.4 编译 `sharp`（原生 libvips）

**症状**：`Could not load the "sharp" module using the android-arm64 runtime`。

**前提**：

```bash
pkg install -y libvips node-addon-api node-gyp pkg-config
PKG_CONFIG_PATH=$PREFIX/lib/pkgconfig pkg-config --modversion vips-cpp
# 8.18.7
```

**定位并编译**：

```bash
find "$NPM_GLOBAL_ROOT" -type d -name "sharp" -path "*@deepseek-ai*" 2>/dev/null
# 典型路径：$DSH_ROOT/node_modules/sharp

cd "$DSH_ROOT/node_modules/sharp"
npm install --no-save node-addon-api node-gyp

cd src
PKG_CONFIG_PATH=$PREFIX/lib/pkgconfig node-gyp configure --directory=.

make -C build BUILDTYPE=Release \
  CXX=$PREFIX/bin/clang++ \
  CC=$PREFIX/bin/clang \
  AR=$PREFIX/bin/llvm-ar \
  RANLIB=$PREFIX/bin/llvm-ranlib
```

成功标志：

```
COPY Release/sharp-android-arm64-0.35.5.node
```

验证：

```bash
cd "$DSH_ROOT/node_modules/sharp"
find . -name "*.node" 2>/dev/null
node -e "const sharp = require('./'); console.log('sharp loaded OK (native)')"
# sharp loaded OK (native)
```

> 如果加载报找不到 `.node`，复制产物：
>
> ```bash
> mkdir -p build/Release
> cp src/build/Release/sharp-android-arm64-0.35.5.node build/Release/
> ```
>
> 如果运行时找不到 `libvips` 动态库：
>
> ```bash
> export LD_LIBRARY_PATH=$PREFIX/lib:$LD_LIBRARY_PATH
> # 并写入 ~/.bashrc
> ```

---

## 4. 阶段 C：绕过 Android 不支持的原生绑定（require-builtin / flock）

### 4.1 替换 `node-addon-require-builtin` 为 Android 桩

**症状**：

```
No usable native binding found for node-addon-require-builtin-android-arm64 (auto)
```

**根因**：该包无 Android 原生绑定，且无 `binding.gyp`，`node-gyp rebuild` 报 `binding.gyp not found`。

**修补**：

```bash
cd "$DSH_ROOT/node_modules/node-addon-require-builtin/lib"
cp index.js index.js.bak

cat > index.js << 'EOF'
'use strict';
if (process.platform === 'android') {
  module.exports = {
    requireBuiltin: function (id) {
      return require(id);
    }
  };
} else {
  const original = require('./index.js.bak');
  module.exports = original;
}
EOF

# 验证
cat index.js
```

> **关键点**：`dsh` 需要的是对象上的 `requireBuiltin` 方法。直接 `module.exports = require` 会报 `addon.requireBuiltin is not a function`。

### 4.2 替换 `flock.js` 为纯 JS 软锁

**症状**：

```
本轮运行失败 flock is not supported on android-arm64
```

**根因**：Android 禁止 `flock(2)` 系统调用，`@deepseek-ai/node-addon-system` 原生模块在加载时抛错。

**修补**：

```bash
cd "$DSH_ROOT/node_modules/@deepseek-ai/node-addon-system/lib"
cp flock.js flock.js.bak

cat > flock.js << 'EOF'
/** Pure-JS flock fallback for Android/Termux (no native flock support). */
import { fstatSync } from 'node:fs';

const locked = new Map(); // key: `${dev}:${ino}` -> fd

function makeContentionError() {
  return Object.assign(new Error('EAGAIN: flock failed'), {
    code: 'EAGAIN',
    errno: 11,
    syscall: 'flock',
  });
}

function acquireSync(fd) {
  const stat = fstatSync(fd);
  const key = `${stat.dev}:${stat.ino}`;
  const existing = locked.get(key);

  if (existing !== undefined) {
    if (existing === fd) return;
    try {
      fstatSync(existing);
      throw makeContentionError();
    } catch (err) {
      if (err && err.syscall === 'flock') throw err;
      locked.delete(key);
    }
  }

  locked.set(key, fd);
}

export async function tryLockExclusive(fd) {
  acquireSync(fd);
}

export function tryLock(fd, cb) {
  try {
    acquireSync(fd);
    if (typeof cb === 'function') cb(0);
  } catch (err) {
    if (typeof cb === 'function') cb(err.errno ?? 11);
    else throw err;
  }
}
EOF

# 验证
cat flock.js
```

> **两个关键坑**：
> 1. **必须用 `fstatSync`**，不能用 `await fstat(fd)`。`fs.fstat` 是回调风格，会报 `The "cb" argument must be of type function. Received undefined`。
> 2. **额外导出 `tryLock(fd, cb)`**，对齐原生绑定的回调签名，防止某些代码路径直接调用它。

**局限**：进程内软锁。跨进程多 worker 场景下互斥会失效，个人 Termux 使用通常可接受。

---

## 5. 阶段 D：绕过 Android 禁止的系统调用（link → rename）

**症状**：

```
EACCES: permission denied, link '...tmp' -> '...session.v4.jsonl.zstd'
```

**根因**：Android SELinux 禁止 `link()` 系统调用（跨目录硬链接）。

**修补位置**：`$DSH_ROOT/node_modules/@deepseek-ai/dsh-session-persistence-jsonl/lib/index.js`

### 5.1 定位所有残留

```bash
cd "$DSH_ROOT/node_modules/@deepseek-ai/dsh-session-persistence-jsonl/lib"

grep -n "\blink\b" index.js | grep -v "rename" | grep -v "unlink" | grep -v "//" | grep -v "\*" | grep -v "symbolic link"
```

典型输出：

```
4:import { link, lstat, mkdir, mkdtemp, open, readFile, readdir, realpath, rm, stat, truncate } from "node:fs/promises";
1615:	link,
2039: await internals.fs.link(staged, currentPath);
3226: await link(tmp, finalPath);
```

> 注意第 1615 行是 **Tab 缩进**（`^Ilink,`），`sed` 用空格模式匹配会失败。

### 5.2 四种形态逐一替换

```bash
cd "$DSH_ROOT/node_modules/@deepseek-ai/dsh-session-persistence-jsonl/lib"

# 1. import 里的 link → rename
sed -i '4s/link/rename/' index.js

# 2. internals.fs.link 调用
sed -i 's/await internals\.fs\.link(/await internals.fs.rename(/g' index.js

# 3. 裸 await link(tmp, finalPath)
sed -i 's/await link(tmp, finalPath)/await rename(tmp, finalPath)/g' index.js

# 4. defaultFileSystem 里的简写属性 link,（Tab 缩进）
sed -i '1615s/\blink,/rename,/' index.js
```

### 5.3 验证

```bash
# import 行
sed -n '3,5p' index.js
# 应是 import { rename, lstat, mkdir, ... } from "node:fs/promises";

# Tab 缩进行
sed -n '1615p' index.js | cat -A
# 应是 ^Irename,$

# 残留检查（应无输出）
grep -n "\blink\b" index.js | grep -v "rename" | grep -v "unlink" | grep -v "//" | grep -v "\*" | grep -v "symbolic link"
```

> **语义说明**：原逻辑是 `link(tmp, finalPath)` 后 `rm(tmp)`，两者等价于 `rename(tmp, finalPath)`。替换安全。
>
> **注意**：如果 `sed '1615s/\blink,/rename,/'` 不生效，先用 `sed -n '1615p' index.js | cat -A` 看实际缩进是 `^I`（Tab）还是空格。

---

## 6. 阶段 E：启动方式固化（shebang / 包装脚本 / 符号链接）

### 6.1 症状与根因

**症状**：

```
Error [ERR_MODULE_NOT_FOUND]: Cannot find package '@deepseek-ai/dsh-app-boot' imported from /.../usr/bin/dsh
```

或：

```
/data/data/com.termux/files/usr/bin/node: --expose-internals is not allowed in NODE_OPTIONS
```

**根因**：`/usr/bin/dsh` 是普通文件，Node 从它所在目录向上查找 `node_modules`，找不到全局包。`NODE_OPTIONS` 也被 Node 明确禁止携带 `--expose-internals`。

### 6.2 找到真实入口

```bash
node -p "require('$DSH_ROOT/package.json').bin.dsh"
# lib/bin.js

export DSH_ENTRY="$DSH_ROOT/lib/bin.js"
```

### 6.3 修改入口 shebang

```bash
sed -i '1s|.*|#!/usr/bin/env -S node --expose-internals|' "$DSH_ENTRY"
head -n 1 "$DSH_ENTRY"
# #!/usr/bin/env -S node --expose-internals
```

### 6.4 创建包装脚本

```bash
cat > ~/dsh-wrapper.sh << 'EOF'
#!/data/data/com.termux/files/usr/bin/bash
# 清除可能干扰 --expose-internals 的 NODE_OPTIONS
unset NODE_OPTIONS
# 用命令行参数携带标志启动 dsh
exec /data/data/com.termux/files/usr/bin/node --expose-internals \
  /data/data/com.termux/files/usr/lib/node_modules/@deepseek-ai/dsh/lib/bin.js "$@"
EOF

chmod +x ~/dsh-wrapper.sh
```

> **为什么包装脚本要 `unset NODE_OPTIONS`**：即使当前 shell 已经清空，`~/.bashrc`、`$PREFIX/etc/bash.bashrc` 里可能残留。包装脚本里主动清除，确保任何情况下 `NODE_OPTIONS` 不会干扰。

### 6.5 把 `/usr/bin/dsh` 替换为符号链接

```bash
cp /data/data/com.termux/files/usr/bin/dsh /data/data/com.termux/files/usr/bin/dsh.bak
rm /data/data/com.termux/files/usr/bin/dsh
ln -s ~/dsh-wrapper.sh /data/data/com.termux/files/usr/bin/dsh

ls -l /data/data/com.termux/files/usr/bin/dsh
# lrwxrwxrwx ... dsh -> /data/data/com.termux/files/home/dsh-wrapper.sh
```

**为什么必须用符号链接**：Node 解析符号链接时，以目标文件所在位置为模块解析基准。指向 `~/dsh-wrapper.sh` 后，Node 在 `~/` 下解析模块，仍会向上到 `$NPM_GLOBAL_ROOT`，能找到 `@deepseek-ai/dsh-app-boot`。

### 6.6 固化启动别名

```bash
echo "alias dsh-web='~/dsh-wrapper.sh web --no-open'" >> ~/.bashrc
source ~/.bashrc
```

以后：

```bash
dsh-web
```

---

## 7. 阶段 F：禁用无法加载的实验插件（cot-translation）

**症状**：

```
@deepseek-ai/dsh-experimental-client-ui-cot-translation: failed to import: Error [ERR_MODULE_NOT_FOUND]
```

**根因**：实验性 CoT 翻译插件依赖平台服务，Android 上无法加载。

### 7.1 移走插件目录

```bash
PLUGIN_DIR="$DSH_MOD/dsh-experimental-client-ui-cot-translation"
mv "$PLUGIN_DIR" "$PLUGIN_DIR.disabled"
```

### 7.2 删除 bundle 中的 insert 条目

```bash
BUNDLE_PATCH="$DSH_MOD/dsh-experimental-cot-translation-bundle/cordis.patch.yml"

cat "$BUNDLE_PATCH"
# - insert:
#     - id: translator
#       name: '@deepseek-ai/dsh-experimental-translator'
#     - id: cot-translation
#       name: '@deepseek-ai/dsh-experimental-client-ui-cot-translation'
#       config:
#         provider: bing
#         targetLanguage: auto

sed -i '/- id: cot-translation/,/targetLanguage: auto/d' "$BUNDLE_PATCH"
cat "$BUNDLE_PATCH"
# 只剩 translator 一条
```

> 如果 `translator` 后续也报错，同样移走：
>
> ```bash
> mv "$DSH_MOD/dsh-experimental-translator" "$DSH_MOD/dsh-experimental-translator.disabled"
> ```

**恢复方式**：把 `.disabled` 改回原名，并恢复 bundle patch。

---

## 8. 阶段 G：运行时问题 1 — terminal inspection unsupported

### 8.1 现象

打开/使用 DSH 持久终端时 100% 报：

```
终端错误：subprocess-local: terminal inspection is unsupported on platform android
```

普通 bash 工具（在文件策略切到 `danger-full-access` 后）不受影响。

### 8.2 定位过程

```bash
grep -rn "inspection is unsupported" "$DSH_MOD/dsh-subprocess-local/lib/"

# 命中：
# lib/runner-launch-B2zsQ1Dz.js:673:
#     throw new Error(`subprocess-local: terminal inspection is unsupported on platform ${platform}`);
```

读工厂函数（第 669~674 行）：

```js
function createProcessInspector(platform = process.platform, arch = process.arch, internals = DEFAULT_INTERNALS) {
	if (platform === "linux") return new LinuxProcessInspector(arch, internals);
	if (platform === "darwin") return new MacProcessInspector(internals);
	if (platform === "win32") return createWindowsProcessInspector();
	throw new Error(`subprocess-local: terminal inspection is unsupported on platform ${platform}`);
}
```

调用点（`lib/index.js:1622`，`spawnTerminal()` 内）：

```js
const inspector = this.terminalInspector ?? createProcessInspector();
```

> `spawnTerminal()` 在 `requireNodePty().spawn(...)` **之前**就调用检查器，Android 直接走到 `throw`，终端必失败（与 PTY 无关）。

### 8.3 全部平台判断（3 处需要改）

```bash
grep -n '"linux"' "$DSH_MOD/dsh-subprocess-local/lib/"*.js
```

| 文件:行 | 作用 | 是否修改 |
|---|---|---|
| `runner-launch-B2zsQ1Dz.js:670` | `createProcessInspector()` 工厂 | **必改**（致命） |
| `runner-launch-B2zsQ1Dz.js:811` | `fallbackOwner()` 僵尸进程组判定 | **必改** |
| `index.js:1037` | `inspectActivity()` 的 idle/busy 分支 | **必改** |
| `index.js:1586` | systemd scope 选择 | **不改**（Android 无 systemd，应保持 fallback） |

### 8.4 修复（3 处）

```bash
cd "$DSH_MOD/dsh-subprocess-local/lib"

# 备份
cp -p runner-launch-B2zsQ1Dz.js runner-launch-B2zsQ1Dz.js.androidorig
cp -p index.js index.js.androidorig

# 1. createProcessInspector（第 670 行）
sed -i '670s/platform === "linux"/platform === "linux" || platform === "android"/' runner-launch-B2zsQ1Dz.js

# 2. fallbackOwner（第 811 行）
sed -i '811s/platform === "linux"/platform === "linux" || platform === "android"/' runner-launch-B2zsQ1Dz.js

# 3. inspectActivity（第 1037 行）
sed -i '1037s/(this.platform === "linux")/(this.platform === "linux" || this.platform === "android")/' index.js
```

### 8.5 验证

**A. 工厂函数**：

```bash
cd "$DSH_ROOT"
node --input-type=module -e "
import { D as createProcessInspector } from '@deepseek-ai/dsh-subprocess-local';
const inspector = createProcessInspector();
console.log('created:', inspector.constructor.name);
"
# created: LinuxProcessInspector
```

**B. 真实 PTY 端到端**：

```bash
cd "$DSH_ROOT"
node --input-type=module -e "
import { D as createProcessInspector } from '@deepseek-ai/dsh-subprocess-local';
const inspector = createProcessInspector();
const self = process.pid;
console.log('snapshot complete =', inspector.snapshot().length > 0);
console.log('self alive =', inspector.isAlive(self));
"
```

**C. 反向验证（证明因果链）**：

```bash
cp runner-launch-B2zsQ1Dz.js.androidorig runner-launch-B2zsQ1Dz.js
# 重新执行 A，应抛出原错误
# 然后重做修复
```

**D. 运行中服务器实测**（重启 `dsh web` 后，在 GUI 里）：

```bash
terminal_open  → started terminal session pty-1
terminal_send  → echo LIVE_TERMINAL_OK; uname -s; echo $SHELL
               → LIVE_TERMINAL_OK
               → Linux
               → /data/data/com.termux/files/usr/bin/bash
terminal_close → closed terminal session pty-1
```

### 8.6 补丁文件

保存为 `terminal-android.patch`，可用于提上游或升级后重新应用。

---

## 9. 阶段 H：运行时问题 2 — ripgrep launch failed

### 9.1 现象

调用 `grep` / `glob` 工具时：

```
Error: grep could not start its search command (ripgrep launch failed)
Error: glob could not start its search command (ripgrep launch failed)
```

这不是"没匹配到"，而是工具**起不来**，模型失去代码搜索能力。

### 9.2 定位过程

```bash
# 1. 直接 import 看错误
node --input-type=module -e "await import('@vscode/ripgrep')"
# FAILS -> Could not find @vscode/ripgrep-android-arm64.
#          Ensure optionalDependencies are installed for this platform (android-arm64).

# 2. 看机制
cat "$DSH_ROOT/node_modules/@vscode/ripgrep/lib/index.js"
# const platformPkg = `@vscode/ripgrep-${process.platform}-${arch}`;   // android-arm64
# 官方 optionalDependencies 只有 linux/darwin/win32
```

### 9.3 修复

**第 1 步：安装系统 ripgrep**

```bash
pkg install -y ripgrep
# ripgrep 15.2.0
command -v rg
# $PREFIX/bin/rg
```

**第 2 步：给 `resolveRgPath()` 加 PATH 回退**

编辑 `$DSH_MOD/dsh-tool-fs-search/lib/index.js`，3 处改动：

**1）新增 imports**：

```js
import { delimiter, isAbsolute, join, parse, relative, sep } from "node:path";
import { accessSync, constants, existsSync, statSync } from "node:fs";
```

**2）新增 `resolveSystemRipgrep()`**：

```js
function resolveSystemRipgrep() {
  const paths = (process.env.PATH ?? "").split(delimiter);
  const name = process.platform === "win32" ? "rg.exe" : "rg";
  for (const dir of paths) {
    if (!dir) continue;
    const candidate = join(dir, name);
    try {
      const st = statSync(candidate);
      if (!st.isFile()) continue;
      accessSync(candidate, constants.X_OK);
      return candidate;
    } catch {}
  }
  return undefined;
}
```

**3）`resolveRgPath()` 的 try/catch 回退**：

```js
try {
  const dependency = (await import("@vscode/ripgrep")).rgPath;
  return process.versions.electron === void 0 ? dependency : dependency.replace(/\.asar(?=[\\/])/u, ".asar.unpacked");
} catch (error) {
  /* No packaged binary for this platform (android-arm64 has no @vscode/ripgrep build): use a system `rg` on PATH. */
  const systemRipgrep = resolveSystemRipgrep();
  if (systemRipgrep !== void 0) return systemRipgrep;
  throw error;
}
```

**备份**：`cp -p lib/index.js lib/index.js.androidorig`

### 9.4 验证

```bash
# 语法
node --check "$DSH_MOD/dsh-tool-fs-search/lib/index.js"

# 真实运行
cd "$DSH_ROOT"
node --input-type=module -e "
import { resolveRgPath, runRipgrep } from '@deepseek-ai/dsh-tool-fs-search';
console.log('rg path =', resolveRgPath());
"
# rg path = /data/data/com.termux/files/usr/bin/rg
```

运行中服务器实测（重启 `dsh web` 后）：

```
grep 工具  pattern "function createProcessInspector" → 命中 1 处
glob 工具  pattern "dsh-android-patch/*.mjs" → 3 个文件
```

### 9.5 补丁文件

保存为 `search-android.patch`。

---

## 10. 阶段 I：运行时问题 3 — sandbox workspace-write 无后端

### 10.1 现象

任何 bash 命令都被拒绝：

```
Error: sandbox mode "workspace-write" is requested but no sandbox backend is usable on this host;
refusing to run the command unconfined. Install bubblewrap or run a Landlock-enforcing kernel (Linux),
ensure sandbox-exec is usable (macOS), or ensure the ACL restricted-token runner can start (Windows)
— otherwise switch the consumer to danger-full-access.
```

### 10.2 根因

- `dsh-sandbox-local/lib/index.js:173-177` 的 `PLATFORM_CHAINS = { linux, darwin, win32 }` **没有 android**。
- 默认模式是 `workspace-write`（`dsh-base/cordis.patch.yml:246`），所以**开箱即用时每次 bash 都会失败**。
- 这不是 bug，而是 **fail-closed 的安全设计**：没有可靠沙箱时拒绝"无约束执行"。Android 内核 4.9 且应用沙箱受限，无法提供 bubblewrap/Landlock。
- 文件工具（read/write/edit）**不受影响**：`dsh-fs-sandbox` 是进程内按路径强制，不依赖 OS 沙箱。

### 10.3 处理：持久化 `danger-full-access`

当前会话是用户在 GUI 里手动切的，新会话会退回默认值。写入用户 profile 覆盖层：

```bash
PROFILE="$HOME/.dsh/profiles/web/cordis.patch.yml"

# 备份
cp -p "$PROFILE" "$PROFILE.bak-$(date +%Y%m%d)-before-android-full-access"

# 追加
cat >> "$PROFILE" << 'EOF'
# Android/Termux: there is no bubblewrap/Landlock sandbox backend on this host,
# so the default workspace-write mode fails closed for every bash call. Persist
# the exact semantics of the GUI "danger-full-access" permission preset
# (sandbox: danger-full-access, approval: never). Remove these two entries to
# restore the default workspace-write + ask behavior.
- id: sandbox-policy
  name: "@deepseek-ai/dsh-sandbox-policy"
  config:
    mode: danger-full-access
    workspaceRoot: !!js process.cwd()
- id: approval
  name: "@deepseek-ai/dsh-user-approval"
  config:
    policy: never
EOF
```

### 10.4 不重启验证（离线）

```bash
cd "$DSH_ROOT"
node lib/bin.js --profile web --dump-config > $PREFIX/tmp/dump.yml

grep -n -A5 "id: sandbox-policy" $PREFIX/tmp/dump.yml
# 应包含 mode: danger-full-access

grep -n -A4 "id: approval" $PREFIX/tmp/dump.yml
# 应包含 policy: never
```

### 10.5 回滚

```bash
# 方式一：恢复备份
cp "$PROFILE.bak-$(date +%Y%m%d)-before-android-full-access" "$PROFILE"

# 方式二：删除文件末尾的 sandbox-policy 与 approval 两段，重启 dsh web
```

> **安全提醒**：`danger-full-access` 意味着 bash 不再受 DSH 文件沙箱限制（本机本来也没有可用沙箱）。若以后 Termux/内核支持 Landlock 或安装 bubblewrap，应改回 `workspace-write`。
>
> **不要使用 `sandbox.runnerCommand` 绕过**——那只是"声称已隔离"的断言，实际没有隔离。

---

## 11. 阶段 J：运行时问题 4 — cmd 被当成 shell，退出码 20

### 11.1 现象

- 在 Web GUI 里"新建终端"后，PTY **立刻结束**，界面显示 `进程已退出（20）`。
- 终端内容只有一行：`cmd: No service specified; use -l to list all services`。
- 反复新建都失败。

> 注意：这次与 §8 的 `terminal inspection is unsupported` 不同。这次 PTY **真的启动了**，只是启动的程序是 `cmd`。

### 11.2 根因

1. **退出码来源**：Android `cmd` 无参数运行 → `exit 20`，输出正是这一行。
2. **GUI 终端链路**：浏览器终端 → `dsh-api-terminal-controller` 的 `spawn()` → `argv: [shell.path, ...shell.args]`；shell 由 `resolveShell()` / `discoverShells()` 解析。
3. **根因入口**：`shellCandidates` 默认值是 `["zsh", "bash", "fish", "pwsh", "powershell", "cmd"]` —— 混入了 Windows 专用命令处理器。`discoverShells()` 对每个名字调用 `resolveExecutable(name)`，而服务器 `PATH=$PREFIX/bin` 里恰好存在 `cmd`（Termux 提供的 Android 服务管理器包装器）。
4. **实测发现结果**（打补丁前）：

```
默认候选发现结果: bash=/data/data/com.termux/files/usr/bin/bash | cmd=/data/data/com.termux/files/usr/bin/cmd
```

5. **为什么每次都启动 cmd**：`dsh-api-terminal-controller/lib/client.js` 把用户选择的 shell 存进浏览器 `localStorage`。新建终端时优先用记住的路径，否则回退 `shells[0]`。只要曾经选过 `cmd`，之后每次新建都会复用 `cmd`。

### 11.3 修复（两层）

**第 1 层：代码根因修复**（`$DSH_MOD/dsh-api-terminal-controller/lib/index.js`）

```js
/** Executable names that only denote Windows command processors; `cmd` collides with Android's service-manager binary on POSIX hosts. */
const WINDOWS_ONLY_SHELLS = new Set(["cmd", "powershell"]);

async function discoverShells(subprocess, configured, candidates, signal) {
	const preferred = await resolveShell(subprocess, configured, signal);
	/* Windows command processors are not shells on POSIX hosts. On Android/Termux the shipped
	   candidate `cmd` resolves to $PREFIX/bin/cmd, the Android service-manager wrapper, which
	   exits 20 when launched as a shell; `powershell` is likewise Windows-only here. */
	const discovered = process.platform === "win32"
		? candidates
		: candidates.filter((candidate) => !WINDOWS_ONLY_SHELLS.has(shellKind(candidate)));
	const found = await Promise.all(discovered.map(async (candidate) => {
		// ... 原逻辑
	}));
	// ...
}
```

**备份**：`cp -p lib/index.js lib/index.js.androidorig`

**第 2 层：profile 兜底**（`~/.dsh/profiles/web/cordis.patch.yml`，升级 dsh 也不会丢）

```bash
PROFILE="$HOME/.dsh/profiles/web/cordis.patch.yml"
cp -p "$PROFILE" "$PROFILE.bak-$(date +%Y%m%d)-before-shell-candidates"

cat >> "$PROFILE" << 'EOF'
- id: terminal-controller
  name: "@deepseek-ai/dsh-api-terminal-controller"
  config:
    shellCandidates: ["zsh", "bash", "fish"]
EOF
```

### 11.4 生效方式（重要）

1. **再重启一次 `dsh web`**（当前进程里没有新补丁与配置）。`Ctrl+C` 停掉后重新执行 `dsh web`。
2. 重启后：新建终端默认 `bash`，shell 菜单只剩 bash（以及已安装的 zsh/fish）。
3. **旧的失败终端标签**：如果它是用 `cmd` 参数创建的，重启后该标签会提示 `Selected shell is not available in this execution environment` —— 关闭它、重新新建即可。浏览器里记住的旧偏好会自动回退到 `bash`。

### 11.5 验证

**A. 直接复现 cmd 陷阱**：

```bash
node -e "const pty=require('node-pty'); const p=pty.spawn('cmd',[],{cols:80,rows:24}); p.onExit(e=>console.log(e))"
# { exitCode: 20, signal: null }
```

**B. 打补丁后 shell 发现**：

```bash
node --input-type=module -e "
import { resolveShell } from '@deepseek-ai/dsh-api-terminal-controller';
const shells = await resolveShell();
console.log(shells.map(s => s.path));
"
# 只应含 bash，不含 cmd/powershell
```

**C. profile 组合结果**：

```bash
cd "$DSH_ROOT"
node lib/bin.js --profile web --dump-config > $PREFIX/tmp/dump2.yml
grep -n -A5 "id: terminal-controller" $PREFIX/tmp/dump2.yml
# shellCandidates:
#   - zsh
#   - bash
#   - fish
```

### 11.6 补丁文件

保存为 `terminal-controller-android.patch`。

### 11.7 与 §12.7 的 cmd: 提示关系

§12.7 里那行 `cmd: No service specified` 当时只是终端里的误触发提示。本次确认了它其实是 Termux 的 `cmd` 包装器，而 DSH 的 shell 候选列表会把同名的它当成 shell。两件事根因相同（Android 同名命令语义不同），但本节是真正导致"无法新建终端"的原因。

---

## 12. 过程中遇到的其他问题与解决

### 12.1 没有 `/tmp` → 写脚本/日志到 `$PREFIX/tmp`

- **现象**：`write` 工具报 `ENOENT: no such file or directory, mkdir '/tmp'`。`node -p "require('os').tmpdir()"` 给出 `$PREFIX/tmp`。
- **原因**：Android 没有 `/tmp`，Termux 用 `$TMPDIR=$PREFIX/tmp`。
- **解决**：所有临时文件改用 `$TMPDIR`/`$PREFIX/tmp`。

### 12.2 `write` 工具报 `EACCES ... link` → 改用 heredoc / edit

- **现象**：`write` 在 `$PREFIX/tmp` 和 workspace 下都报：
  `EACCES: permission denied, link '/.../.README.md.<...>.tmpdir/README.md.tmp' -> '/.../README.md'`
- **原因**：DSH 的 `write` 用"临时目录 + 硬链接"的原子写法；Android/SELinux 在 `/data` 下禁止这种 `link(2)`。
- **解决**：创建/覆盖文件改用 bash heredoc（`cat > file <<'EOF'`），小改动用 `edit` 工具（原地读写，不走 link）。

### 12.3 验证"原文件确实抛错"时，备份文件不能直接 import

- **第一次**：把 `runner-launch-*.js.androidorig` 直接 `import()` → Node 因未知扩展名报错。
- **第二次**：复制成 `.mjs` 放 `$PREFIX/tmp` → 因 bundle 里有相对导入 `./output.js` 而失败（不在原目录）。
- **最终**：复制到原 `lib/` 目录、命名 `runner-launch-ORIG-check.js`（保证 `./output.js` 可解析），验证后立即删除。

### 12.4 `subagent` 不是 Team member → `wait_agent` 返回 `no-active-peer`

- **现象**：`wait_agent` 报 `no-active-peer: No other Team member is running or provisioning`。
- **原因**：`subagent` 是独立管理的子代理，不算 Team teammate；结果通过运行时通知/消息返回。
- **解决**：不阻塞轮询，继续做可独立推进的工作，审计结果随后以消息送达。

### 12.5 `--dump-config` 命令被中断 + `cmd: No service specified`

- **现象**：执行 `node lib/bin.js --profile web --dump-config > dump.yml` 时工具调用被中断，终端随后出现一行 `cmd: No service specified; use -l to list all services`。
- **排查**：
  1. 检查产物：`$PREFIX/tmp/dump.yml` **确实已生成**（47 KB），`dump.err` **为空**（0 字节）。
  2. 用 `grep` 校验内容，确认 `sandbox-policy: mode: danger-full-access` 与 `approval: policy: never` 都在组合结果里。
  3. `ps` 检查没有残留的 `--dump-config`/`bin.js` 进程。
- **原因**：`--dump-config` 本身成功，只是工具调用在返回后被中断；`cmd: ...` 是 Android 的 `cmd` 用法提示，与本任务无关。`command -v cmd` → `$PREFIX/bin/cmd`，无参数直接运行就打印这一行（exit 0）。
- **解决**：改用"检查产物 + `grep` 校验 + YAML 解析"三重确认配置正确。

### 12.6 重启后如何确认补丁真的加载了（而不是"看起来改了"）

运行中的 Node 进程早已把旧模块载入内存，改文件不会热更新。本次通过三件事交叉确认：

1. 进程启动时间（`/proc/<pid>/stat` 的 `starttime`，`stat /proc/<pid>`）晚于补丁/配置修改时间。
2. `--dump-config` 显示组合后的配置。
3. **直接调用工具实测**：`grep`、`glob`、`terminal_open/send/read/close` 全部成功。

### 12.7 其他排查中确认"不是问题"的点

- `systemd`/`systemctl` **不是硬依赖**：`index.js:1586` 的 `if (platform === "linux")` 把整条 systemd scope 路径挡住了，Android 走 fallback containment，只打印一条 warn。
- `/bin/ps` 只在 macOS 检查器里用；Android 走 `/proc`。
- `/bin/sh`、`DEFAULT_BASH_SHELL="/bin/bash"` 在 Termux 下能用（`termux-exec` 的 `LD_PRELOAD` 会重写路径）。更稳妥的旋钮是 `shellPath` 配置。
- 所有 `=== "win32"` / `!== "win32"` 分支在 android 上都按 POSIX 走，行为正确。
- `koffi` 自带 `android_arm64` 构建，可正常加载。

---

## 13. 全部补丁与验证总结

### 13.1 补丁清单（5 个包文件 + 2 组 profile 配置）

| 包 | 文件 | 改动数 | 补丁名 | 阶段 |
|---|---|---|---|---|
| `@deepseek-ai/dsh-subprocess-local` | `lib/runner-launch-B2zsQ1Dz.js` | 2 | `terminal-android.patch` | G |
| `@deepseek-ai/dsh-subprocess-local` | `lib/index.js` | 1 | `terminal-android.patch` | G |
| `@deepseek-ai/dsh-tool-fs-search` | `lib/index.js` | 3 | `search-android.patch` | H |
| `@deepseek-ai/dsh-api-terminal-controller` | `lib/index.js` | 2 hunk | `terminal-controller-android.patch` | J |
| `@deepseek-ai/dsh-session-persistence-jsonl` | `lib/index.js` | 4 处 | （§5 合并） | D |
| `~/.dsh/profiles/web/cordis.patch.yml` | `sandbox-policy` + `approval` | — | 备份 `...before-android-full-access` | I |
| `~/.dsh/profiles/web/cordis.patch.yml` | `terminal-controller.shellCandidates` | — | 备份 `...before-shell-candidates` | J |

### 13.2 验证脚本

建议在 `~/jsq/dsh-android-patch/` 下保存：

| 文件 | 用途 | 通过标准 |
|---|---|---|
| `verify.mjs` | 终端 4 项端到端验证 | 4/4 PASS |
| `verify-search.mjs` | grep/glob 3 项端到端验证 | 3/3 PASS |
| `verify-shells.mjs` | shell 发现 3 项验证 | 3/3 PASS |
| `apply.mjs` | 幂等应用/检查/还原（`--check` / `--restore`） | 7 处 OK |

### 13.3 运行中服务器最终实测

```
grep 工具 → 命中 1 处
glob 工具 → 返回 3 个文件

terminal_open  → started terminal session pty-1 (live-check) [type: shell]
terminal_send  → echo LIVE_TERMINAL_OK; uname -s; echo $SHELL
               → LIVE_TERMINAL_OK
               → Linux
               → /data/data/com.termux/files/usr/bin/bash
terminal_close → closed terminal session pty-1
```

---

## 14. 局域网访问

### 14.1 风险先明确

一旦绑定 `0.0.0.0`，**同一 Wi-Fi 下任何设备都能无认证地访问你的 DSH 实例，等同于把本机的命令执行能力暴露给整个局域网**。请只在家庭或公司可信内网中使用。

### 14.2 推荐：`dsh-network-mode` 插件

**前提**：pnpm 已装且安装脚本执行（见 §2.4）。

```bash
dsh plugin --profile web add dsh-network-mode
```

安装后**重启一次** `dsh web`。然后打开 Web UI：

1. 进入 **设置 → 网络模式**。
2. 拨动 **「局域网访问」** 开关。
3. **再次重启** `dsh web`，`0.0.0.0` 的绑定配置才会真正生效。

### 14.3 如果出现 `switch: HTTP 405`

这是插件切换接口被 DSH 网关拒绝（网关通常只放行 GET/HEAD/POST）。改为手动修改 profile：

```bash
nano ~/.dsh/profiles/web/cordis.patch.yml
```

在顶层 YAML 数组中添加：

```yaml
- id: webserver
  config:
    host: '0.0.0.0'
```

保存后重启：

```bash
dsh web
```

### 14.4 备选：`dsh-lan-access`（安装即生效）

```bash
dsh plugin --profile web add dsh-lan-access
```

安装后**完全重启** `dsh web`。该插件通常安装即绑定 `0.0.0.0`，无图形开关，关闭需卸载。

### 14.5 获取手机 IP 并访问

```bash
ifconfig wlan0 | grep "inet "
# inet 192.168.x.x
```

其他设备访问：

```
http://<手机IP>:3080/?token=<终端输出的token>
```

### 14.6 注意事项

- **统一访问地址**：本机和手机尽量都使用 `http://<手机IP>:3080`，不要混用 `127.0.0.1` 和局域网 IP，否则会话列表可能不一致。
- **换 Wi-Fi 后重启**：DSH 的浏览器信任名单是启动快照，换网后新 IP 可能 403，重启 `dsh web` 即可。
- **切回本机模式**：在设置里关闭开关，或删除 `host: '0.0.0.0'` 配置并重启。

---

## 15. 文件、备份与回滚

### 15.1 补丁目录建议结构（`~/jsq/dsh-android-patch/`）

| 文件 | 用途 |
|---|---|
| `README.md` | 本说明文档 |
| `terminal-android.patch` | 阶段 G 终端 3 处改动 |
| `search-android.patch` | 阶段 H grep/glob 回退改动 |
| `terminal-controller-android.patch` | 阶段 J shell 发现过滤改动 |
| `apply.mjs` | 幂等应用/检查/还原（`--check` / `--restore`） |
| `verify.mjs` | 终端 4 项验证 |
| `verify-search.mjs` | grep/glob 3 项验证 |
| `verify-shells.mjs` | shell 发现 3 项验证 |

### 15.2 所有被修改文件的备份位置

| 路径 | 备份 |
|---|---|
| `$DSH_MOD/dsh-subprocess-local/lib/runner-launch-B2zsQ1Dz.js` | 同目录 `runner-launch-B2zsQ1Dz.js.androidorig` |
| `$DSH_MOD/dsh-subprocess-local/lib/index.js` | 同目录 `index.js.androidorig` |
| `$DSH_MOD/dsh-tool-fs-search/lib/index.js` | 同目录 `index.js.androidorig` |
| `$DSH_MOD/dsh-api-terminal-controller/lib/index.js` | 同目录 `index.js.androidorig` |
| `$DSH_MOD/dsh-session-persistence-jsonl/lib/index.js` | 无备份（直接 sed 修改），可通过 `npm i -g @deepseek-ai/dsh` 恢复原版 |
| `$DSH_MOD/node-addon-require-builtin/lib/index.js` | 同目录 `index.js.bak` |
| `$DSH_MOD/@deepseek-ai/node-addon-system/lib/flock.js` | 同目录 `flock.js.bak` |
| `~/.dsh/profiles/web/cordis.patch.yml` | `cordis.patch.yml.bak-*` |
| `$DSH_MOD/dsh-experimental-client-ui-cot-translation` | 改名为 `.disabled` |
| `$DSH_MOD/dsh-experimental-cot-translation-bundle/cordis.patch.yml` | 无备份（已删 cot-translation 段），可通过重装恢复 |

### 15.3 一键回滚

```bash
# 1. 还原代码补丁（如有 apply.mjs）
cd ~/jsq/dsh-android-patch
node apply.mjs --restore

# 2. 还原 profile 备份
cp ~/.dsh/profiles/web/cordis.patch.yml.bak-20261022-before-android-full-access \
   ~/.dsh/profiles/web/cordis.patch.yml

# 3. 恢复被移走的插件目录
mv "$DSH_MOD/dsh-experimental-client-ui-cot-translation.disabled" \
   "$DSH_MOD/dsh-experimental-client-ui-cot-translation"

# 4. 恢复手工修改的文件（从 .bak / .androidorig 复制回来）
# 5. 重启 dsh web
```

### 15.4 完整重装（最彻底）

```bash
npm uninstall -g @deepseek-ai/dsh
npm cache clean --force
CFLAGS="-target aarch64-linux-android30" \
CXXFLAGS="-target aarch64-linux-android30" \
npm install -g @deepseek-ai/dsh
# 然后重新应用所有补丁
```

---

## 16. 升级后如何重新应用

`npm i -g @deepseek-ai/dsh`（或升级）会覆盖 `node_modules` 里的补丁，profile 不受影响。

### 16.1 升级前备份

```bash
cp -r ~/.dsh ~/.dsh.bak
cp ~/dsh-wrapper.sh ~/dsh-wrapper.sh.bak
```

### 16.2 升级后重做清单

| 阶段 | 项目 | 是否需重做 |
|---|---|---|
| A | Node LTS | 不需要（除非升级 Node） |
| B | `$PREFIX/include/node/common.gypi` 修补 | **升级 Node 后需重做** |
| B | `koffi` API 30 重编译 | 需重做 |
| B | `sharp` 原生编译 | 需重做 |
| C | `node-addon-require-builtin/lib/index.js` 桩 | 需重做 |
| C | `@deepseek-ai/node-addon-system/lib/flock.js` 软锁 | 需重做 |
| D | `dsh-session-persistence-jsonl/lib/index.js` link→rename | 需重做 |
| E | `/usr/bin/dsh` 符号链接 + `~/dsh-wrapper.sh` | 不需要（除非被覆盖） |
| F | `cot-translation` 禁用 | 需重做 |
| G | `dsh-subprocess-local` 3 处 | 需重做 |
| H | `dsh-tool-fs-search` 3 处 | 需重做 |
| I | profile `sandbox-policy` + `approval` | 不需要（profile 保留） |
| J | `dsh-api-terminal-controller` + profile `shellCandidates` | 代码需重做，profile 保留 |

### 16.3 用 `apply.mjs` 批量重应用（推荐）

```bash
cd ~/jsq/dsh-android-patch
node apply.mjs            # 应用（幂等；版本/代码不匹配会安全中止）
node apply.mjs --check    # 只检查
node verify.mjs && node verify-search.mjs && node verify-shells.mjs
# 重启 dsh web
```

> `apply.mjs` 自动匹配 `runner-launch-*.js` 的 hash 文件名（每次发布可能变化）。新版 dsh 若改动了这些代码，会报 `cannot find expected code for "..."` 并停止，不会乱改。此时以三个 `.patch` 为参考重新适配。

---

## 17. Android 上的其他限制（非本次 bug）

### 17.1 进程树隔离降级（可接受）

Android 没有 systemd / 私有 bootstrap，`selectContainmentMode()` 走 fallback containment，日志会输出：

```
subprocess-local is using weaker process-tree containment because platform android has no native managed range;
descendants that escape the process group or direct-parent tree are not guaranteed to terminate or delay waitForExit()
```

含义：逃出进程组/直接父树的孙进程不保证被回收。已修补的僵尸组判定可避免 `terminate()` 空等。这是能力缺失而非 bug。

### 17.2 文件沙箱（已在 §10 处理）

无 bubblewrap/Landlock → `workspace-write` 必然 fail-closed。已持久化 `danger-full-access`。文件工具本身不受影响。

### 17.3 `node-pty` 全新安装风险

本机 `node-pty` 有本地编译的 `build/Release/pty.node`，所以终端可用。但**全新安装 dsh 时**，若没有预编译的 `prebuilds/android-arm64/pty.node`，会报 `Failed to load native module: pty.node`。届时需要：

```bash
pkg install -y build-essential python
# 重新安装/重建 node-pty（或将本机 build/Release/pty.node 复制到新安装对应目录）
```

### 17.4 其他功能级降级（与本次修复无关）

| 功能 | 现象 | 现状 |
|---|---|---|
| 默认 workspace 目录 | `system Documents directory is unavailable on android` | 可配置 `documentsDirectory` 绕过 |
| LibreOffice 转换 | `Unsupported LibreOfficeKit host: android-arm64` | 无 Android 引擎包，功能不可用 |
| 本地语音转文字 | `Local speech is unavailable for android-arm64` | 白名单不含 android；该 bundle 未启用 |
| 工作区依赖（SDK app） | 平台清单永不匹配 android | 仅 SDK app 挂载，不影响 Web profile |

### 17.5 给上游的建议

1. `packages/subprocess/subprocess-local`：把 `android` 归入 Linux 分支（本手册阶段 G 即最小改法），或显式提供 Android inspector。
2. `dsh-tool-fs-search`：当 `@vscode/ripgrep` 无对应平台二进制时回退到系统 `rg`（本手册阶段 H）。
3. 文档：明确 Android/Termux 需要 `danger-full-access`（无 OS 沙箱后端）。
4. 安装期：为 Android 提供 node-pty 预编译产物，或在 `postinstall` 中自动本地编译。
5. `dsh-api-terminal-controller`：`shellCandidates` 默认值应区分平台（本手册阶段 J）。

---

## 18. 常见问题速查表

| 报错/症状 | 阶段 | 处理 |
|---|---|---|
| `nvm is not compatible with "PREFIX"` | A | 不用 nvm，`pkg install nodejs-lts` |
| `koffi` 报 `statx` / `__u32` | B | 用 `-target aarch64-linux-android30` 重编译 |
| `node-pty` 报 `android_ndk_path` | B | 修补 `$PREFIX/include/node/common.gypi` |
| `sharp` 加载失败 `Could not load the "sharp" module` | B | 系统 `libvips` + `node-gyp configure` + `make` |
| `No usable native binding found for node-addon-require-builtin-android-arm64` | C | 替换 `lib/index.js` 为 Android 桩（`requireBuiltin(id) → require(id)`） |
| `flock is not supported on android-arm64` | C | 替换 `flock.js` 为纯 JS 软锁（用 `fstatSync`） |
| `The "cb" argument must be of type function. Received undefined` | C | `flock.js` 中不能用 `await fstat(fd)`，用 `fstatSync` |
| `EACCES: permission denied, link '...'` | D | `link` → `rename` 全面替换（注意 4 种形态） |
| `ERR_MODULE_NOT_FOUND @deepseek-ai/dsh-app-boot` | E | `/usr/bin/dsh` 改为符号链接到包装脚本 |
| `--expose-internals is not allowed in NODE_OPTIONS` | E | 不用 `NODE_OPTIONS`，用包装脚本命令行参数 |
| `cot-translation: failed to import` | F | 移走目录 + 删除 bundle patch 条目 |
| `terminal inspection is unsupported on platform android` | G | 3 处平台判断扩展 `android → linux` |
| `grep/glob could not start its search command (ripgrep launch failed)` | H | `pkg install ripgrep` + `resolveRgPath()` 加 PATH 回退 |
| `sandbox mode "workspace-write" ... no sandbox backend` | I | profile 持久化 `danger-full-access` + approval `never` |
| `进程已退出（20）cmd: No service specified` | J | POSIX 过滤 `cmd`/`powershell` + profile 限定 `shellCandidates` |
| `switch: HTTP 405`（局域网） | 14 | 手动改 `cordis.patch.yml` 的 `host: '0.0.0.0'` 或用 `dsh-lan-access` |
| 3080 端口占用 | 通用 | `dsh web --port 8080` |
| 沙箱不可用 | 通用 | 见阶段 I |
| 新建终端提示 `Selected shell is not available` | J | 关闭旧标签，重新新建（浏览器会回退到 bash） |

---

## 附录 A：全部对话记录

> 按时间顺序记录全部用户消息（原话）与助手每轮回复的实质内容。工具级细节见正文各阶段。

### A.1 会话开始前的上下文

DSH 运行时上下文（节选）：

- 工作目录：`/data/data/com.termux/files/home/jsq`
- 文件策略：`workspace-write`（workspace 为上述目录）
- 审批策略：`ask`
- 实现 checkout：`/data/data/com.termux/files/usr/lib/node_modules/@deepseek-ai/dsh/`
- Web GUI：`http://192.168.151.43:3080`
- 时间：`2026-10-22T01:57:11+08:00`（Asia/Shanghai）

### A.2 用户消息 #1

```
终端错误：subprocess-local: terminal inspection is unsupported on platform android
```

### A.3 助手回复 #1（要点）

- 定位到唯一抛错点 `runner-launch-B2zsQ1Dz.js:673` 与调用点 `spawnTerminal()`；
- 确认环境：`process.platform=android`、内核 Linux、`/proc` 可用、node-pty 可 spawn；
- 修复 3 处（让 android 走 linux 分支），备份原文件；
- 写 `verify.mjs` 做 4 项真实端到端验证（4/4 PASS），并用还原测试证明因果链；
- 结论：**需要重启 `dsh web` 才生效**；
- 提示另外两个 Android 限制：文件沙箱无后端、进程树隔离降级；
- 询问是否把 `danger-full-access` 持久化。

### A.4 用户消息 #2

```
可以，但是要记录你用的指令，遇到的问题怎么解决的，全部归纳到说明文档里面去，包括现刚刚的问题怎么解决的？，所有对话对话
```

### A.5 助手回复 #2（要点）

- 接收并采纳独立审计报告——其中指出 **grep/glob 也因 ripgrep 无 android 包而不可用**；
- 执行 `pkg install -y ripgrep`（15.2.0）+ 给 `resolveRgPath()` 加 PATH 回退补丁；
- 写 `verify-search.mjs` → 3/3 PASS；
- 泛化 `apply.mjs` 支持两个包共 6 处编辑（`--check`/`--restore`），往返测试通过；
- 生成 `terminal-android.patch`、`search-android.patch`；
- 再次提示需重启一次 `dsh web`，并再次询问是否持久化沙箱策略。

### A.6 用户消息 #3（含终端里带出的一行 Android `cmd` 提示）

```
cmd: No service specified; use -l to list all services可以，但是要记录你用的指令，遇到的问题怎么解决的，全部归纳到说明文档里面去，包括现刚刚的问题怎么解决的？，所有对话对话
```

### A.7 助手回复 #3（要点；完成了当时的文档）

- 先完成沙箱持久化（阶段 I），用 `--dump-config` 离线验证；
- 排查并解释"刚刚的问题"（§12.5）：`--dump-config` 被中断但产物完好、`cmd:` 提示与本任务无关；
- 发现服务器已重启，**直接实测** `grep`、`glob`、`terminal_open/send/read/close` 全部正常；
- 将全部指令、问题与解决、对话、审计报告、验证输出汇总成本文档。

### A.8 用户消息 #4

```
进程已退出（20）cmd: No service specified; use -l to list all services无法新建终端
```

### A.9 助手回复 #4（要点）

- 复现：`cmd` 无参数 → `exit 20` + 同一句提示；直接 `spawnTerminal({argv:['cmd']})` 得到逐字相同的输出；
- 定位：`dsh-api-terminal-controller` 的 `shellCandidates` 默认含 `cmd`，而 Termux 的 `$PREFIX/bin/cmd` 是 Android 服务管理器包装器，被 `discoverShells()` 当成可用 shell；选择又被浏览器 `localStorage` 记住，所以每次新建终端都复用它；
- 修复：代码层在 POSIX 上过滤 `cmd`/`powershell`；profile 层限定 `shellCandidates: [zsh,bash,fish]`；
- 验证：`verify-shells.mjs` 3/3 PASS；还原补丁后 B 项按预期失败（cmd leaked）；
- 生成 `terminal-controller-android.patch`，`apply.mjs` 扩到 7 处编辑；
- 更新本文件：摘要、指令、文件/回滚、完整记录；
- 提示：**需要再重启一次 `dsh web`**；旧的失败终端标签关闭后重新新建即可。

### A.10 用户消息 #5

```
根据所有对话生成出现的问题解决的办法以后，根据这个来流程安装 用的什么指令？哪里要修改非常的详细给我写进去，生成md格式文件
```

> 助手产出第一版"从零安装完整流程"Markdown（对应本手册阶段 A~E、F 的部分）。

### A.11 用户消息 #6（粘贴 README.md）

用户提供了 `README.md` 全文，并说：

```
将这个和你之前写的那个文档合并在一起，这后面发生的一些修改的一些问题
```

> 助手产出本合并版手册。

---

## 附录 B：独立审计报告

> 由独立子代理只读审计 DSH 其他包在 Android 上的兼容性，未修改任何文件。

**Blockers**

- **Sandbox has no android chain** — `dsh-sandbox-local/lib/index.js:173-177` `PLATFORM_CHAINS = { linux, darwin, win32 }`; `:497` `const chain = this.internals.chain ?? PLATFORM_CHAINS[this.internals.platform ?? process.platform] ?? [];` ⇒ `:492` `throw new SandboxUnavailableError(mode)` — verified by invoking `chainVerdict` with `platform:"android"` → `"unavailable"`. User-visible message (`dsh-sandbox/lib/index.js:272`): `sandbox mode "workspace-write" is requested but no sandbox backend is usable on this host; refusing to run the command unconfined. Install bubblewrap or run a Landlock-enforcing kernel (Linux)…otherwise switch the consumer to danger-full-access.` Default is `workspace-write` (`dsh-base/cordis.patch.yml:246` `mode: !!js process.env.DSH_PERMISSION_MODE ?? 'workspace-write'`) with `bash-sandbox` mounted (`:249`) ⇒ **every bash call fails out of the box**; also PTC/workflow (`dsh-ptc-runtime-node:1147`, `dsh-workflow-ptc:427`), worktree, plugin-manager. File tools are **not** affected: `dsh-fs-sandbox` fences paths in-process (only injects `sandboxPolicy`), so read/write/edit still enforce the mode.

- **Persistent danger-full-access knobs (all verified)**: (a) env `DSH_PERMISSION_MODE=danger-full-access` (`dsh-base:246`, approval row `:262` then `never`); (b) profile patch row `- id: sandbox-policy` + `config: { mode: danger-full-access }` (shipped precedent `dsh-sdk-minimal/cordis.patch.yml:73`); (c) per-session UI permission preset → `sandbox/mode` event, persisted by replay (active session log contains `{"type":"sandbox/mode",…,"data":{"mode":"danger-full-access","source":"delegation"}}`); (d) per-call `sandbox_permissions` escalation (`dsh-tool-bash:518-525`), which works because `dsh-bash-sandbox:68` returns unconfined for that mode. No safe *code* fix without an Android backend; `sandbox.runnerCommand` (`sandbox-local:255`) bypasses the error but is assertion-only — unsafe.

- **grep/glob tools dead** — `dsh-tool-fs-search/lib/index.js:125` `const dependency = (await import("@vscode/ripgrep")).rgPath;` → `@vscode/ripgrep/lib/index.js:7` `@vscode/ripgrep-${process.platform}-${arch}`; no `android-arm64` build exists (optionalDeps are darwin/linux/win32 only). Observed live: `Error: grep could not start its search command (ripgrep launch failed)` (`:188`); glob identical. Fix: `pkg install ripgrep` + PATH fallback patch, or shim the platform package.

**Degradations**

- `dsh-api-workspace-controller/lib/index.js:633` `default: throw new Error(\`system Documents directory is unavailable on ${platform}\`)` (via `initializeDefault:884`) → first-use default workspace errors; workaround: config `documentsDirectory`.
- `libreoffice-kit/lib/index.js:1496` platformTarget → undefined; `:1540` `Unsupported LibreOfficeKit host: android-arm64`; office-to-pdf is mounted by web-app (`dsh-web-app/cordis.patch.yml:287`), so conversion fails (no android engine package).
- `dsh-experimental-speech-to-text-sensevoice/lib/index.js:207` allow-list omits android → "Local speech is unavailable for android-arm64" (bundle not active).
- `dsh-tool-workspace-dependencies/lib/index.js:166` `manifest.platform !== process.platform` never matches (SDK app only).
- Native deps: no package declares `os`/`cpu`, so installs succeed. `node-addon-system` lacks an android package but degrades gracefully (`index.js:46-58` → path that never exists → probe unusable). `koffi 3.1.1` ships `build/koffi/android_arm64` and loads fine. `node-pty` has no android prebuild (`utils.js:19` checks `prebuilds/android-arm64`); this install's `build/Release/pty.node` was compiled on-device (00:16), so terminals work here but a fresh non-building install fails with "Failed to load native module: pty.node".

**Checked and fine**

- systemd is **not** a hard requirement: `dsh-subprocess-local/lib/index.js:1586` `if (platform === "linux")` gates the whole `systemd-run`/`systemctl` scope path; android takes fallback and only warns `platform android has no native managed range` (`:1605-1606`).
- `/bin/ps` is MacProcessInspector-only (android uses `/proc`); `/bin/sh` in `subprocess-local/lib/runner.js:67` is linux-scope bootstrap only; `/tmp` hits are unused bwrap/seatbelt profiles or the LibreOffice wasm FS. `dsh-terminal-bash:9` `DEFAULT_BASH_SHELL="/bin/bash"` works on Termux only because termux-exec's `LD_PRELOAD` rewrites it (verified pty spawn prints `PTY_OK`; env scrub keeps it, `dsh-subprocess/lib/index.js:52`) — fragile, `shellPath` is the safe knob.
- All win32/darwin-guarded branches behave as POSIX on android; your flock pure-JS fallback and both subprocess-local android patches (`:1037`, `runner-launch:671`) are in place and verified.

**Priority**: fix ripgrep resolution first (cheap, restores grep/glob); then document/ship an Android sandbox default (env or profile) — the fail-closed behavior itself is correct, there is just no backend; office/workspace-default issues are optional-feature level.

---

## 附录 C：验证输出原文

### C.1 工厂函数 / `/proc` 检查器（阶段 G）

```
platform = android
created: LinuxProcessInspector
snapshot complete = true rows = 5
self pid = 27973 tree = [{"pid":27973,"started":"9204198"}]
foregroundPgid(self) = undefined
isAlive(self) = true
unknown platform still throws: subprocess-local: terminal inspection is unsupported on platform plan9
```

### C.2 真实 PTY 端到端（阶段 G）

```
runtime constructed: LocalSubprocessRuntime | name = subprocess
shell = /data/data/com.termux/files/usr/bin/bash
[warn] subprocess-local is using weaker process-tree containment because platform android has no native managed range; descendants that escape the process group or direct-parent tree are not guaranteed to terminate or delay waitForExit()
SPAWNED pid = 28075 (the pre-patch code threw here on android)
inspectForeground = {"processGroupId":28075,"inputWaiting":true}
inspectActivity = {"state":"unknown","revision":1}
outcome = {"exitCode":0,"signal":null}
output bytes = 145 | marker seen = true
RESULT: PASS
```

### C.3 `verify.mjs` 终端 4/4

```
platform=android arch=arm64 node=v24.18.0

PASS  A1 inspector factory accepts android — LinuxProcessInspector
PASS  A2 /proc snapshot + tree + liveness — 8 rows visible, pid 31765 alive
PASS  B spawnTerminal + inspect + exec + exit — pid 31772, fg 31772
PASS  C shellActivity idle -> busy -> idle — idle -> busy -> idle

RESULT: ALL CHECKS PASSED (4/4)
containment warning: subprocess-local is using weaker process-tree containment because platform android has no native managed range; …
```

### C.4 `verify-search.mjs` grep/glob 3/3

```
platform=android arch=arm64 node=v24.18.0

PASS  A resolveRgPath falls back to system rg — /data/data/com.termux/files/usr/bin/rg
PASS  B runRipgrep executes and finds a match — 1 match line(s), workdir=/data/data/com.termux/files/home/jsq
PASS  C zero-match semantics preserved (exit 1 -> noMatches) — noMatches=true

RESULT: ALL CHECKS PASSED (3/3)
```

### C.5 反向验证：还原官方文件后复现原错误

```
platform = android
ORIGINAL CODE THROWS -> subprocess-local: terminal inspection is unsupported on platform android
```

同时 `verify.mjs` 在还原状态下：

```
FAIL  A1 inspector factory accepts android — subprocess-local: terminal inspection is unsupported on platform android
FAIL  A2 /proc snapshot + tree + liveness — subprocess-local: terminal inspection is unsupported on platform android
FAIL  B — subprocess-local: terminal inspection is unsupported on platform android
FAIL  C — subprocess-local: terminal inspection is unsupported on platform android
RESULT: 4 CHECK(S) FAILED (0/4)
```

### C.6 `--dump-config` 组合结果（阶段 I）

```yaml
137:- id: sandbox-policy
138-  name: '@deepseek-ai/dsh-sandbox-policy'
139-  config:
140-    mode: danger-full-access
141-    workspaceRoot: !!js process.cwd()
...
152:- id: approval
153-  name: '@deepseek-ai/dsh-user-approval'
154-  config:
155-    policy: never
```

### C.7 运行中服务器实测（阶段 G/H/I）

```
grep 工具  → Found 1 match: …/runner-launch-B2zsQ1Dz.js Line 669: function createProcessInspector…
glob 工具  → dsh-android-patch/verify.mjs, verify-search.mjs, apply.mjs

terminal_open  → started terminal session pty-1 (live-check) [type: shell]
terminal_send  → echo LIVE_TERMINAL_OK; uname -s; echo $SHELL
               → LIVE_TERMINAL_OK
               → Linux
               → /data/data/com.termux/files/usr/bin/bash
terminal_close → closed terminal session pty-1
```

### C.8 shell 发现与 `cmd` 陷阱（阶段 J）

**打补丁前的默认发现结果**（`shellCandidates` = 出厂默认，含 cmd）：

```
默认候选发现结果: bash=/data/data/com.termux/files/usr/bin/bash | cmd=/data/data/com.termux/files/usr/bin/cmd
```

**直接复现"进程已退出（20）"**：

```
pid = 10153
outcome = {"exitCode":20,"signal":null}
output = "cmd: No service specified; use -l to list all services"
```

**打补丁后 `verify-shells.mjs`**：

```
platform=android arch=arm64 node=v24.18.0

PASS  A the cmd trap is real (exit 20 + Android usage line) — exit 20 · "cmd: No service specified; use -l to list all services"
PASS  B discovered shells exclude Windows-only names — /data/data/com.termux/files/usr/bin/bash
PASS  C first discovered shell spawns, echoes and exits 0 — bash (/data/data/com.termux/files/usr/bin/bash) exit 0

RESULT: ALL CHECKS PASSED (3/3)
```

**还原补丁后（回归证据，`cmd` 重新泄漏）**：

```
FAIL  A the cmd trap is real (exit 20 + Android usage line) — subprocess-local: terminal inspection is unsupported on platform android
FAIL  B discovered shells exclude Windows-only names — cmd leaked into ["/data/data/com.termux/files/usr/bin/bash","/data/data/com.termux/files/usr/bin/cmd"]
FAIL  C first discovered shell spawns, echoes and exits 0 — subprocess-local: terminal inspection is unsupported on platform android
RESULT: 3 CHECK(S) FAILED (0/3)
```

（A/C 同时失败是因为 `--restore` 把阶段 G 的终端补丁也还原了，正好一次性复现两个原始问题。）

**profile 组合结果（`dump2.yml:463-468`）**：

```yaml
- id: terminal-controller
  name: '@deepseek-ai/dsh-api-terminal-controller'
  config:
    shellCandidates:
      - zsh
      - bash
      - fish
```

**最终状态**：

```
verify.mjs        RESULT: ALL CHECKS PASSED (4/4)
verify-search.mjs RESULT: ALL CHECKS PASSED (3/3)
verify-shells.mjs RESULT: ALL CHECKS PASSED (3/3)
apply.mjs --check → 7 处 OK
node --check ×4   → ALL 4 FILES SYNTAX OK
```

---

## 附：本次会话的关键结论（一句话版）

1. **终端问题**：`process.platform === "android"` 未被当作 Linux → 3 处平台判断修复 → 实测终端可用；
2. **grep/glob 问题**：`@vscode/ripgrep` 无 android 二进制 → 装系统 `rg` + PATH 回退 → 实测可用；
3. **沙箱问题**：Android 无 OS 沙箱后端（fail-closed 正确）→ profile 持久化 `danger-full-access`（含备份/回滚）→ `--dump-config` 确认；
4. **新建终端退出码 20**：Termux 的 `$PREFIX/bin/cmd`（Android 服务管理器）被当成 shell → POSIX 过滤 `cmd`/`powershell` + profile 限定候选 → `verify-shells.mjs` 3/3；
5. **升级**：`npm i -g` 后重跑 `node apply.mjs` 即可；profile 不受影响；
6. **所有改动可回滚**：`apply.mjs --restore` + 两份 profile 备份。

---

**文档版本**：合并版 v1.0  
**合并来源**：《Termux 从零手动安装 DSH 完整流程》（阶段 A~F）+《DSH on Android/Termux 完整修复记录》（阶段 G~J + 附录）  
**更新时间**：2026-10-22  
**适用 dsh 版本**：0.2.1-rc.2  
**适用 Node 版本**：v24.18.0  
**验证状态**：全部通过（终端 4/4、grep/glob 3/3、shell 发现 3/3、7 处补丁 OK、4 文件语法 OK）