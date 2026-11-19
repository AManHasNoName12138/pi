# Pi Linux / Windows 源码构建与官方版本隔离手册

> 更新：2026-11-19。适用范围：本仓库的 Pi coding-agent CLI / TUI，Linux 与原生 Windows；不包含 p2 服务部署、远程 daemon 或浏览器应用。
>
> 源码基线：[`6b339afc92e1dc6574f3234e1850dff36c77294b`](https://github.com/AManHasNoName12138/pi/commit/6b339afc92e1dc6574f3234e1850dff36c77294b)，`@earendil-works/pi-coding-agent` 版本 `1.5.2`。后续更新源码时，应重新核对构建脚本、配置优先级和资源发现逻辑。

本文参考 [OpenCode Linux / Windows 源码构建、独立安装与并存手册](https://github.com/AManHasNoName12138/opencode/blob/study-notes/study-notes/OpenCode_Linux_Windows_Source_Build_and_Isolation_Guide.md)，沿用“独立工具链、独立工作副本、独立启动入口、独立数据、并存验收”的方法，并按 Pi 的实际实现调整。Pi 的 Node 构建不需要套用 OpenCode 的 release channel、数据库命名和后台服务端口设置。

完成后，官方版仍使用原来的 `pi`；源码版通过本文的 `pi_dev` / `Invoke-PiDev` 启动，认证、配置、模型缓存、会话和项目文件全部放在独立的 `PI_DEV_ROOT` 中。这里的函数名只是独立入口，程序内部仍叫 `pi`，版本号也仍可能与官方版相同。

**保证的边界：**按本文入口运行、使用独立工作副本且不主动引用官方路径时，两版的应用数据与构建产物分开。环境变量不是操作系统访问控制；Pi 的文件工具、shell、扩展及 MCP 仍拥有当前用户的权限。若要求“即使模型执行了错误命令，也不能读写官方数据”，应先使用第 2 节的独立系统账户或虚拟机，再执行后续步骤。两种要求不能只靠改一个目录变量同时满足。

## 1. 先理解 Pi 的构建与数据路径

### 1.1 从 monorepo 根目录构建

本快照要求 Node.js **≥ 22.19.0**。本文为开发版单独下载 Node 24 LTS 的当前补丁版，使用随包提供的 npm；不升级系统 Node，不执行 `npm install -g` 或 `npm link`。

根 `package.json` 的构建顺序为：

```text
telemetry → schema → chord → tui → codemode → mcp → ai
          → durable → env → agent → coding-agent
```

安装使用 `npm ci --ignore-scripts`，保留锁文件，避免安装时自动运行依赖生命周期脚本。构建则显式执行 `npm run build`。不要只进入 `packages/coding-agent` 安装或构建，否则 workspace 依赖可能缺少 `dist/`。

源码构建的正式 Node 入口是：

```text
<源码目录>/packages/coding-agent/dist/bundle/cli.js
```

它与包里的 `bin.pi` 一致。保留整个工作副本、`node_modules`、各包 `dist/` 和资源文件；不能只复制这个 JS 文件。构建会生成拆分模块、worker、主题和其他资源，部分依赖仍为外部依赖。

`npm run build` 会联网获取模型目录，并可能更新 `packages/ai/src/models.generated.ts`；供应商数据写入被 Git 忽略的 `packages/ai/src/providers/data/`。**首次克隆不能直接假设 `build:offline` 可用**：它要求模型数据已经生成。首次完整构建成功后，普通代码迭代可用 `npm run build:offline`；它不是 npm 的离线安装模式，也不保证所有外部工具绝不联网。

### 1.2 只改可执行文件名称并不能隔离数据

| 对象 | 官方默认位置 / 来源 | 本文源码版位置 / 处理 |
| --- | --- | --- |
| Pi 用户目录 | `~/.pi/agent`，可被 `PI_CODING_AGENT_DIR` 改写 | `PI_DEV_ROOT/agent` |
| 登录凭据 | `auth.json` | 新的 `agent/auth.json`，独立登录 |
| 设置、模型与模型目录缓存 | `settings.json`、`models.json`、`models-store.json` | 新的 `agent/` 下同名文件 |
| 会话 | 通常为 `agent/sessions/<编码后的项目路径>/` | 显式指定 `agent/sessions`，由 Pi 按会话中的项目路径筛选 |
| 扩展、技能、提示模板、主题 | 用户目录及项目 `.pi/` | 独立用户目录和独立项目副本 |
| MCP 配置 | 用户目录和受信任项目的 `mcp.json` | 独立文件；不复制官方服务器/凭据配置 |
| 项目设置 | 当前目录 `.pi/settings.json` | 只在 `workspaces/` 内运行 |
| 指令上下文 | 用户目录，以及当前目录一直到文件系统根的 `AGENTS*` / `CLAUDE*` | 检查独立根目录之外的祖先，遇到指令文件即拒绝启动 |
| npm 缓存、全局前缀及配置 | npm 自己的默认目录 / 继承环境 | `cache/npm`、`npm/prefix`、独立 npmrc |
| 其他工具常用用户目录 | `HOME`、Windows 用户目录、XDG、临时目录 | 只在子进程中重定向到独立目录 |

会话目录优先级是：

```text
--session-dir > PI_CODING_AGENT_SESSION_DIR > settings.sessionDir > 默认目录
```

其中项目 `sessionDir` 会在项目信任判断之前参与会话选择。因此启动器同时固定 `PI_CODING_AGENT_DIR` 和 `PI_CODING_AGENT_SESSION_DIR`，并拒绝直接覆盖会话目录或从路径打开/分叉会话。常规恢复使用源码版自己的 `--resume` / `--continue`。

`--no-extensions` 不会关闭祖先目录指令文件的加载，伪造 `HOME` 也不会阻止从真实工作目录向上遍历。`AGENTS.override.md` 只覆盖同目录的其他候选文件，不能屏蔽所有祖先。本文对此单独检查，而不是依赖项目信任设置。

仓库自带的 `pi-test.sh` / `pi-test.ps1` / `pi-test.bat` 是源码调试入口，**默认仍会使用正常用户目录**；`--no-env` 仅清除部分 API 环境变量，不是数据隔离开关。

## 2. 选择隔离强度与目录

### 2.1 需要文件权限保证时

- **Linux：**使用另一个无 sudo 权限的开发账户，官方数据和安装目录只授权官方账户；检查祖先目录的执行权限、目录 ACL 和文件权限。可以用 `sudo -u <开发账户> test -r <官方认证文件>` 与 `sudo -u <开发账户> test -w <官方数据目录>` 验证访问应被拒绝。不要把官方账户的 home、SSH agent、Docker socket 或凭据目录挂载给开发环境。
- **Windows：**使用独立的非管理员本地账户或独立虚拟机，检查官方用户目录的 NTFS ACL，确保开发账户没有读取或写入权限。以该账户实际尝试访问官方 `auth.json` 和数据目录，确认访问被拒绝；不要仅依据用户名不同就认为权限已隔离。
- **虚拟机 / 容器：**只共享独立源码和测试工作区，不挂载官方数据、官方安装目录或宿主高权限 socket。容器的安全性还取决于挂载与运行权限。

若两边本来就被授予同一个共享目录的权限，系统账户不同也不能保护该目录。需要同时保护官方认证的“读取”和官方数据的“写入”。

### 2.2 同一用户下的应用数据隔离

日常开发可以使用同一账户和下列独立目录。不要放在官方 `.pi` 内，也不要使用官方项目的 symlink、junction、硬链接副本或 Git worktree；从远端重新 clone，避免共享项目配置、Git 管理目录和构建文件。

```text
PI_DEV_ROOT/
├── pi-dev.mjs                 # 第 3 节启动器，放在根目录而非仓库内
├── tools/node/                # 独立 Node + npm
├── downloads/                 # 安装包与 SHA-256 清单
├── home/                      # 子进程 HOME / USERPROFILE
├── agent/                     # Pi 的所有默认用户状态
│   ├── settings.json
│   └── sessions/
├── config/ data/ state/ cache/ # XDG 和工具缓存
├── appdata/ tmp/ npm/          # Windows / 临时 / npm 数据
├── reports/                   # 官方数据校验清单
└── workspaces/
    ├── pi/                    # 独立 Pi 源码
    └── demo/                  # 独立测试项目
```

以下 Linux 示例使用 `$HOME/pi-source-dev`；Windows 使用用户目录下的 `pi-source-dev`。这些都必须是**新目录**。如果上级目录有 `AGENTS.md` 等指令文件，启动器会报错；应选择干净位置或独立账户，不要删除官方项目的文件来绕过检查。

## 3. 两个平台共用的隔离启动器

在准备好第 4 / 5 节的根目录后，将下面代码以 UTF-8 保存为根目录下的 **`pi-dev.mjs`**。它只使用 Node 内置模块，不需要提前安装依赖。

它给子进程重新建立环境，保留 PATH、终端、系统目录、代理和 CA 证书设置；不继承 API key、云凭据、SSH agent、`PI_*` 覆盖、Node 预加载参数或 shell 启动脚本变量。PATH 仅在子进程内加入私有 Node。保留的 PATH 仍可能找到官方 `pi`，因此在开发环境里也应使用本文入口；不要让工具通过裸 `pi` 命令调用官方版。

默认生成 `defaultProjectTrust: "never"` 和关闭安装遥测的设置；已有开发设置不会被覆盖。需要项目扩展时，只在检查过的开发副本中调整信任。`PI_SKIP_VERSION_CHECK=1` 关闭版本检查；它不是所有网络访问的总开关。

```javascript
// pi-dev.mjs — save directly in PI_DEV_ROOT; requires Node >= 22.19.
import { existsSync, mkdirSync, realpathSync, writeFileSync } from "node:fs";
import { dirname, delimiter, isAbsolute, join, relative, resolve, sep } from "node:path";
import { fileURLToPath } from "node:url";
import { spawnSync } from "node:child_process";
import { constants } from "node:os";

const root = realpathSync(dirname(fileURLToPath(import.meta.url)));
const windows = process.platform === "win32";
const node = join(root, "tools", "node", ...(windows ? ["node.exe"] : ["bin", "node"]));
const npm = join(root, "tools", "node", ...(windows
  ? ["node_modules", "npm", "bin", "npm-cli.js"]
  : ["lib", "node_modules", "npm", "bin", "npm-cli.js"]));
const inside = (base, path) => {
  const rel = relative(base, path);
  return rel === "" || (!isAbsolute(rel) && rel !== ".." && !rel.startsWith(`..${sep}`));
};
const fail = (message) => { throw new Error(message); };
const contextNames = ["AGENTS.override.md", "AGENTS.md", "AGENTS.MD", "CLAUDE.md", "CLAUDE.MD"];
for (let dir = dirname(root); ; dir = dirname(dir)) {
  for (const name of contextNames) {
    if (existsSync(join(dir, name))) fail(`External ancestor context: ${join(dir, name)}`);
  }
  if (dirname(dir) === dir) break;
}
function privateDir(path) {
  let ancestor = path;
  while (!existsSync(ancestor)) ancestor = dirname(ancestor);
  if (!inside(root, realpathSync(ancestor))) fail(`Directory escapes root: ${path}`);
  mkdirSync(path, { recursive: true, mode: 0o700 });
  if (!inside(root, realpathSync(path))) fail(`Directory escapes root: ${path}`);
  return path;
}
function privateFile(path, content) {
  privateDir(dirname(path));
  if (existsSync(path)) {
    if (!inside(root, realpathSync(path))) fail(`File escapes root: ${path}`);
  } else {
    writeFileSync(path, content, { flag: "wx", mode: 0o600 });
  }
  return path;
}
const dirs = Object.fromEntries([
  "home", "agent", "agent/sessions", "config", "data", "state", "cache",
  "cache/npm", "cache/node", "tmp", "appdata/roaming", "appdata/local",
  "npm/prefix", "workspaces", "reports",
].map((name) => [name, privateDir(join(root, name))]));
privateFile(join(dirs.agent, "settings.json"), JSON.stringify({
  defaultProjectTrust: "never", enableInstallTelemetry: false,
}, null, 2) + "\n");
const inherited = new Map(Object.entries(process.env).map(([key, value]) => [key.toUpperCase(), value]));
const env = {};
for (const key of [
  "PATH", "SystemRoot", "WINDIR", "COMSPEC", "PATHEXT", "ProgramFiles", "ProgramFiles(x86)",
  "TERM", "COLORTERM", "TERM_PROGRAM", "LANG", "LANGUAGE", "TZ",
  "HTTP_PROXY", "HTTPS_PROXY", "ALL_PROXY", "NO_PROXY",
  "SSL_CERT_FILE", "SSL_CERT_DIR", "NODE_EXTRA_CA_CERTS",
]) {
  const value = inherited.get(key.toUpperCase());
  if (value !== undefined) env[key] = value;
}
for (const [key, value] of Object.entries(process.env)) {
  if (/^LC_[A-Z_]+$/.test(key) && value !== undefined) env[key] = value;
}
if (!windows) {
  for (const key of ["HTTP_PROXY", "HTTPS_PROXY", "ALL_PROXY", "NO_PROXY"]) {
    if (env[key] !== undefined) env[key.toLowerCase()] = env[key];
  }
}
Object.assign(env, {
  PATH: `${dirname(node)}${delimiter}${env.PATH || ""}`,
  HOME: dirs.home, USERPROFILE: dirs.home,
  APPDATA: dirs["appdata/roaming"], LOCALAPPDATA: dirs["appdata/local"],
  XDG_CONFIG_HOME: dirs.config, XDG_DATA_HOME: dirs.data,
  XDG_STATE_HOME: dirs.state, XDG_CACHE_HOME: dirs.cache,
  TMPDIR: dirs.tmp, TMP: dirs.tmp, TEMP: dirs.tmp,
  PI_CODING_AGENT_DIR: dirs.agent,
  PI_CODING_AGENT_SESSION_DIR: dirs["agent/sessions"],
  PI_SKIP_VERSION_CHECK: "1",
  NODE_COMPILE_CACHE: dirs["cache/node"],
  NPM_CONFIG_CACHE: dirs["cache/npm"], NPM_CONFIG_PREFIX: dirs["npm/prefix"],
  NPM_CONFIG_USERCONFIG: privateFile(join(root, "npm", "user.npmrc"), ""),
  NPM_CONFIG_GLOBALCONFIG: privateFile(join(root, "npm", "global.npmrc"), ""),
  NPM_CONFIG_IGNORE_SCRIPTS: "true",
  GIT_CONFIG_NOSYSTEM: "1",
  GIT_CONFIG_GLOBAL: privateFile(join(root, "config", "gitconfig"), ""),
});
if (windows) {
  env.HOMEDRIVE = dirs.home.slice(0, 2);
  env.HOMEPATH = dirs.home.slice(2);
}
const [mode = "run", ...args] = process.argv.slice(2);
const cwd = realpathSync(process.cwd());
if (mode === "paths") {
  console.log(JSON.stringify({ root, node, npm, cwd,
    home: env.HOME, agent: env.PI_CODING_AGENT_DIR,
    sessions: env.PI_CODING_AGENT_SESSION_DIR, cache: env.XDG_CACHE_HOME,
  }, null, 2));
  process.exit(0);
}
if (!inside(dirs.workspaces, cwd)) fail(`Run from an independent workspace under ${dirs.workspaces}`);
let executable = node;
let argv;
if (mode === "npm") {
  argv = [npm, ...args];
} else if (mode === "exec") {
  [executable, ...argv] = args;
  if (!executable || !isAbsolute(executable)) fail("exec requires an absolute executable path");
} else if (mode === "run") {
  const blocked = ["--session-dir", "--session", "--fork", "--export"];
  if (args.some((arg) => blocked.some((flag) => arg === flag || arg.startsWith(`${flag}=`)))) {
    fail("Direct session paths/exports are disabled here; use --resume or --continue");
  }
  if (args[0] === "update") fail("Update this source checkout with Git and rebuild");
  const cli = resolve(root, "workspaces", "pi", "packages", "coding-agent", "dist", "bundle", "cli.js");
  if (!existsSync(cli)) fail("Build from the repository root first");
  argv = [cli, ...args];
} else {
  fail("Usage: pi-dev.mjs paths | npm <args...> | exec <absolute-program> <args...> | run <pi-args...>");
}
const result = spawnSync(executable, argv, { cwd, env, stdio: "inherit", shell: false });
if (result.error) throw result.error;
process.exit(result.status ?? (128 + (constants.signals[result.signal] ?? 1)));
```

该启动器拒绝目录通过 symlink / junction 逃出开发根目录，检查的是创建/启动时的真实路径；它不递归审计所有项目文件，也不能阻止运行中的程序修改配置、创建链接或直接访问绝对路径。`exec` 是显式运行外部工具的维护入口，不是沙箱。不要把官方 `auth.json`、会话、`.pi`、扩展或含绝对路径的配置复制进开发目录，也不要在源码会话里调用官方 `pi`、官方更新器或卸载器。

## 4. Linux：安装工具链、构建、运行

### 4.1 创建独立根目录并记录官方入口

需要 Git、Bash、curl、tar、xz、sha256sum；主流程适合常规 glibc Linux。Alpine/musl 需要另行选择适配的 Node 工具链，不能直接照用 glibc 二进制包。

在正常用户终端中执行，遇到错误即停止，不要继续后面的步骤：

```bash
PI_DEV_ROOT="$HOME/pi-source-dev"
test ! -e "$PI_DEV_ROOT" || { echo "Choose a new PI_DEV_ROOT" >&2; exit 1; }
umask 077
mkdir -p "$PI_DEV_ROOT"/{tools,downloads,workspaces,reports}
type -a pi > "$PI_DEV_ROOT/reports/official-command.txt" 2>&1 || true
PI_OFFICIAL_AGENT_DIR="${PI_CODING_AGENT_DIR:-$HOME/.pi/agent}"
printf '%s\n' "$PI_OFFICIAL_AGENT_DIR" > "$PI_DEV_ROOT/reports/official-agent-path.txt"
PI_GIT="$(command -v git)"
test -n "$PI_GIT"
```

如果官方版使用 shell alias、脚本或其他配置设置了数据目录，上面的环境变量不一定覆盖它：检查官方入口与设置，并记录真实的 agent 目录、额外 session 目录和安装目录。第 6 节的基线必须在首次构建/运行前记录。

### 4.2 下载私有 Node 24 LTS 并校验

```bash
case "$(uname -m)" in
  x86_64) PI_NODE_ARCH=x64 ;;
  aarch64|arm64) PI_NODE_ARCH=arm64 ;;
  *) echo "Select a supported Node architecture manually" >&2; exit 1 ;;
esac
curl -fL https://nodejs.org/dist/latest-v24.x/SHASUMS256.txt \
  -o "$PI_DEV_ROOT/downloads/SHASUMS256.txt" || exit 1
PI_NODE_FILE="$(awk -v suffix="-linux-$PI_NODE_ARCH.tar.xz" '$2 ~ (suffix "$") {print $2}' "$PI_DEV_ROOT/downloads/SHASUMS256.txt")"
test -n "$PI_NODE_FILE" || exit 1
PI_NODE_VERSION="${PI_NODE_FILE#node-}"
PI_NODE_VERSION="${PI_NODE_VERSION%-linux-*}"
curl -fL "https://nodejs.org/dist/$PI_NODE_VERSION/$PI_NODE_FILE" \
  -o "$PI_DEV_ROOT/downloads/$PI_NODE_FILE" || exit 1
(
  cd "$PI_DEV_ROOT/downloads" || exit 1
  awk -v file="$PI_NODE_FILE" '$2 == file' SHASUMS256.txt | sha256sum -c -
) || exit 1
mkdir "$PI_DEV_ROOT/tools/node"
tar -xJf "$PI_DEV_ROOT/downloads/$PI_NODE_FILE" \
  -C "$PI_DEV_ROOT/tools/node" --strip-components=1 || exit 1
PI_DEV_NODE="$PI_DEV_ROOT/tools/node/bin/node"
test -z "${NODE_OPTIONS:-}" || { echo "Use a terminal without NODE_OPTIONS" >&2; exit 1; }
"$PI_DEV_NODE" --version
```

下载清单中的版本用于归档 URL，避免 `latest` 在两次请求间变动。保留清单与安装包以记录工具链版本；哈希核对用于检查下载内容，发布者真实性依赖 HTTPS 或自行核验 Node 发布签名。不要使用官方 Pi 的安装脚本来安装源码版。

### 4.3 保存启动器并完成构建

先保存第 3 节的 `pi-dev.mjs`，再在当前 Bash 定义函数。函数不修改父 shell 的 `HOME`、PATH 或任何 Pi 环境变量；重开终端后需要重新设置 `PI_DEV_ROOT` / `PI_DEV_NODE` 并定义函数。

```bash
pi_dev() {
  if [ -n "${NODE_OPTIONS:-}" ]; then
    echo "Use a terminal without NODE_OPTIONS" >&2
    return 1
  fi
  "$PI_DEV_NODE" "$PI_DEV_ROOT/pi-dev.mjs" "$@"
}
pi_dev paths
```

此时先按第 6.1 节保存官方数据基线，再继续：

```bash
cd "$PI_DEV_ROOT/workspaces" || exit 1
pi_dev exec "$PI_GIT" clone --branch study-notes \
  https://github.com/AManHasNoName12138/pi.git "$PI_DEV_ROOT/workspaces/pi" || exit 1
cd "$PI_DEV_ROOT/workspaces/pi" || exit 1
pi_dev exec "$PI_GIT" switch --detach 6b339afc92e1dc6574f3234e1850dff36c77294b || exit 1
pi_dev exec "$PI_GIT" switch -c local/pi-dev || exit 1
pi_dev npm ci --ignore-scripts || exit 1
pi_dev npm run build || exit 1
pi_dev exec "$PI_GIT" rev-parse HEAD
pi_dev exec "$PI_GIT" status --short
pi_dev run --offline --version
pi_dev run --offline --help
```

整个源码树留在上述路径，构建结果直接由独立入口使用。这里不进行全局安装，也不更改 `/usr/bin/pi`、`~/.local/bin/pi` 或原来的 npm prefix。

### 4.4 首次启动

```bash
mkdir -p "$PI_DEV_ROOT/workspaces/demo"
cd "$PI_DEV_ROOT/workspaces/demo" || exit 1
pi_dev run --offline --no-extensions --no-skills --no-prompt-templates --no-themes
```

这一步用于无凭据、无扩展的界面检查；不发送模型请求。确认看不到官方会话与登录状态后退出，再执行 `pi_dev run`，在开发版中单独 `/login`。登录结果应写入开发 `agent/auth.json`。优先用独立测试账户或 API 项目；同一个云账户的计费、配额、服务端会话不因本地目录分开而隔离。

环境白名单故意不继承 `OPENAI_API_KEY`、`ANTHROPIC_API_KEY` 等变量。需要 API key 时，在开发版独立认证/模型配置中配置，不要复制官方凭据文件或把 key 放在命令历史中。涉及浏览器 OAuth 时也应选择正确的开发账户；系统浏览器已有的登录状态不会被这个启动器隔离。

## 5. Windows：原生构建、运行

### 5.1 准备环境

使用本地 NTFS 盘上的新目录、PowerShell **7.3 或更高版本**及 Git for Windows。本文使用 Windows Node 与 npm；Git Bash 仅供 Pi 的 bash 工具使用，不是在 WSL 中构建。不要把 WSL 的 `node_modules` 或 Linux 构建结果拿来给 Windows 入口复用。

在 `pwsh -NoProfile` 终端执行。下列命令中的 `$PI_DEV_ROOT` 不是 PowerShell 的只读 `$HOME` 变量。

```powershell
$ErrorActionPreference = 'Stop'
$PSNativeCommandArgumentPassing = 'Standard'
$PI_DEV_ROOT = Join-Path $HOME 'pi-source-dev'
if (Test-Path -LiteralPath $PI_DEV_ROOT) { throw 'Choose a new PI_DEV_ROOT' }
New-Item -ItemType Directory -Path $PI_DEV_ROOT | Out-Null
foreach ($Name in @('tools', 'downloads', 'workspaces', 'reports')) {
    New-Item -ItemType Directory -Path (Join-Path $PI_DEV_ROOT $Name) | Out-Null
}
Get-Command pi -All -ErrorAction SilentlyContinue |
    Select-Object CommandType, Name, Source, Definition |
    Out-File (Join-Path $PI_DEV_ROOT 'reports/official-command.txt') -Encoding utf8
$PI_OFFICIAL_AGENT_DIR = if ($env:PI_CODING_AGENT_DIR) {
    $env:PI_CODING_AGENT_DIR
} else {
    Join-Path $HOME '.pi/agent'
}
$PI_OFFICIAL_AGENT_DIR | Set-Content (Join-Path $PI_DEV_ROOT 'reports/official-agent-path.txt')
$PI_GIT = (Get-Command git.exe -CommandType Application).Source
```

官方 npm 安装可能对应 `pi.ps1` / `pi.cmd`，独立可执行文件安装可能对应 `pi.exe`。不要把命令显示名当作真正文件路径；记录 shim 及其目标安装目录。官方的额外 session 目录也要纳入第 6 节校验。

### 5.2 下载私有 Node 24 LTS

```powershell
$PI_NODE_ARCH = switch ([System.Runtime.InteropServices.RuntimeInformation]::OSArchitecture.ToString()) {
    'X64' { 'x64' }
    'Arm64' { 'arm64' }
    default { throw 'Select a supported Node architecture manually' }
}
$PI_SUMS_PATH = Join-Path $PI_DEV_ROOT 'downloads/SHASUMS256.txt'
Invoke-WebRequest 'https://nodejs.org/dist/latest-v24.x/SHASUMS256.txt' -OutFile $PI_SUMS_PATH
$PI_PATTERN = '^([a-fA-F0-9]{64})\s+(node-(v24\.[0-9]+\.[0-9]+)-win-' + $PI_NODE_ARCH + '\.zip)$'
$PI_MATCHES = @(Get-Content $PI_SUMS_PATH | Select-String -Pattern $PI_PATTERN)
if ($PI_MATCHES.Count -ne 1) { throw 'Expected exactly one Node archive' }
$PI_MATCH = $PI_MATCHES[0].Matches[0]
$PI_NODE_HASH = $PI_MATCH.Groups[1].Value
$PI_NODE_FILE = $PI_MATCH.Groups[2].Value
$PI_NODE_VERSION = $PI_MATCH.Groups[3].Value
$PI_NODE_ZIP = Join-Path $PI_DEV_ROOT "downloads/$PI_NODE_FILE"
Invoke-WebRequest "https://nodejs.org/dist/$PI_NODE_VERSION/$PI_NODE_FILE" -OutFile $PI_NODE_ZIP
if ((Get-FileHash $PI_NODE_ZIP -Algorithm SHA256).Hash -ne $PI_NODE_HASH) {
    throw 'Node archive checksum mismatch'
}
Expand-Archive -LiteralPath $PI_NODE_ZIP -DestinationPath (Join-Path $PI_DEV_ROOT 'tools')
Move-Item -LiteralPath (Join-Path $PI_DEV_ROOT ('tools/' + $PI_NODE_FILE.Replace('.zip', ''))) `
    -Destination (Join-Path $PI_DEV_ROOT 'tools/node')
$PI_DEV_NODE = Join-Path $PI_DEV_ROOT 'tools/node/node.exe'
if ($env:NODE_OPTIONS) { throw 'Use a terminal without NODE_OPTIONS' }
& $PI_DEV_NODE --version
if ($LASTEXITCODE -ne 0) { throw 'Node failed' }
```

使用本地盘路径，不使用 UNC 路径；本文启动器的 `HOMEDRIVE` / `HOMEPATH` 按盘符路径设置。Windows ARM64 的 Node 包存在不代表每个依赖都有 ARM64 原生包；以安装/构建输出为准，必要时采用完整的一套 x64 工具链并确认系统支持仿真。

### 5.3 保存启动器并构建

将第 3 节代码保存到 `$PI_DEV_ROOT\pi-dev.mjs`，然后定义函数。`Invoke-PiDev` 使用参数数组转交参数，出错立即抛出；不通过拼接命令字符串或 `cmd /c` 启动 npm，也不受 `npm.ps1` 执行策略影响。

```powershell
function Invoke-PiDev {
    if ($env:NODE_OPTIONS) { throw 'Use a terminal without NODE_OPTIONS' }
    & $PI_DEV_NODE (Join-Path $PI_DEV_ROOT 'pi-dev.mjs') @args
    if ($LASTEXITCODE -ne 0) { throw "pi-dev exited with $LASTEXITCODE" }
}
Invoke-PiDev paths
```

先完成第 6.1 节基线，再执行：

```powershell
Set-Location (Join-Path $PI_DEV_ROOT 'workspaces')
Invoke-PiDev exec $PI_GIT clone --branch study-notes `
    https://github.com/AManHasNoName12138/pi.git (Join-Path $PI_DEV_ROOT 'workspaces/pi')
Set-Location (Join-Path $PI_DEV_ROOT 'workspaces/pi')
Invoke-PiDev exec $PI_GIT switch --detach 6b339afc92e1dc6574f3234e1850dff36c77294b
Invoke-PiDev exec $PI_GIT switch -c local/pi-dev
Invoke-PiDev npm ci --ignore-scripts
Invoke-PiDev npm run build
Invoke-PiDev exec $PI_GIT rev-parse HEAD
Invoke-PiDev exec $PI_GIT status --short
Invoke-PiDev run --offline --version
Invoke-PiDev run --offline --help
```

### 5.4 首次启动

```powershell
New-Item -ItemType Directory -Force -Path (Join-Path $PI_DEV_ROOT 'workspaces/demo') | Out-Null
Set-Location (Join-Path $PI_DEV_ROOT 'workspaces/demo')
Invoke-PiDev run --offline --no-extensions --no-skills --no-prompt-templates --no-themes
```

退出后用 `Invoke-PiDev run` 正常启动并单独登录。Pi 会优先寻找 Git for Windows 的 Bash；若安装在非常规位置，可在**开发版** `agent/settings.json` 中添加例如 `"shellPath": "D:\\Tools\\Git\\bin\\bash.exe"`，同时保留已有字段。不要修改官方设置，也不要把 Windows 系统里的旧 WSL `bash.exe` 当成 Git Bash。

重开终端后重新设置根目录、私有 Node 路径和函数即可，无需把开发版命名成 `pi.exe` 或修改用户/系统 PATH。需要持久化函数时，用独立的开发启动脚本，避免覆盖已有官方 alias。

## 6. 两个平台都必须完成的隔离验收

### 6.1 构建之前：对官方数据与安装做基线

先关闭所有官方 Pi 会话及会修改其数据的进程，保持关闭直到完成对比。把下面代码保存为 `PI_DEV_ROOT/snapshot.mjs`。它记录路径、文件内容 SHA-256、mtime 和模式；不保存文件内容，不跟随符号链接。读取可能影响 atime，所以不以 atime 判断变更；Windows ACL 需另外用 `Get-Acl` / `icacls` 检查。

每一个官方 agent 目录、定制 session 目录、独立安装目录都应纳入。npm 安装请覆盖 package 实际目录和入口 shim；如果路径是链接，还应把其真实目标单独列入。不要只校验一个 shim 就宣称整个安装未变。

```javascript
// snapshot.mjs — manifests belong outside every directory being inspected.
import { createHash } from "node:crypto";
import { createReadStream } from "node:fs";
import { lstat, readdir, readlink, writeFile } from "node:fs/promises";
import { isAbsolute, join, relative, resolve, sep } from "node:path";

const [outputArg, ...targetArgs] = process.argv.slice(2);
if (!outputArg || targetArgs.length === 0) throw new Error("snapshot.mjs <output.json> <absolute-target>...");
const output = resolve(outputArg);
const records = [];
async function visit(path) {
  let info;
  try { info = await lstat(path); }
  catch (error) {
    if (error.code === "ENOENT") { records.push({ path, type: "missing" }); return; }
    throw error;
  }
  const record = { path, mode: info.mode, mtimeMs: info.mtimeMs, size: info.size };
  if (info.isSymbolicLink()) {
    records.push({ ...record, type: "link", target: await readlink(path) });
  } else if (info.isDirectory()) {
    records.push({ ...record, type: "directory" });
    for (const name of (await readdir(path)).sort()) await visit(join(path, name));
  } else if (info.isFile()) {
    const hash = createHash("sha256");
    for await (const chunk of createReadStream(path)) hash.update(chunk);
    records.push({ ...record, type: "file", sha256: hash.digest("hex") });
  } else {
    throw new Error(`Unsupported file type: ${path}`);
  }
}
for (const arg of targetArgs) {
  if (!isAbsolute(arg)) throw new Error(`Use an absolute path: ${arg}`);
  const target = resolve(arg);
  const rel = relative(target, output);
  if (rel === "" || (!isAbsolute(rel) && rel !== ".." && !rel.startsWith(`..${sep}`))) {
    throw new Error("Manifest must not be written inside an inspected target");
  }
  await visit(target);
}
await writeFile(output, JSON.stringify(records, null, 2) + "\n", { flag: "wx", mode: 0o600 });
console.log(`Recorded ${records.length} entries`);
```

Linux 示例（先把路径变量补全为实际值；所有路径必须绝对化，`~` 字符串不会自动展开）：

```bash
PI_OFFICIAL_INSTALL_DIR='/replace/with/actual/official/install'
test -e "$PI_OFFICIAL_INSTALL_DIR" || exit 1
"$PI_DEV_NODE" "$PI_DEV_ROOT/snapshot.mjs" "$PI_DEV_ROOT/reports/before.json" \
  "$PI_OFFICIAL_AGENT_DIR" "$PI_OFFICIAL_INSTALL_DIR"
```

Windows 示例：

```powershell
$PI_OFFICIAL_INSTALL_DIR = 'C:\replace\with\actual\official\install'
if (-not (Test-Path -LiteralPath $PI_OFFICIAL_INSTALL_DIR)) { throw 'Set the real installation path' }
& $PI_DEV_NODE (Join-Path $PI_DEV_ROOT 'snapshot.mjs') (Join-Path $PI_DEV_ROOT 'reports/before.json') `
    $PI_OFFICIAL_AGENT_DIR $PI_OFFICIAL_INSTALL_DIR
if ($LASTEXITCODE -ne 0) { throw 'Snapshot failed' }
```

可在末尾继续添加额外的绝对路径。两次快照必须传入**相同路径、相同顺序**，并使用同一 Node 版本。记录读取失败会使脚本失败，不能把失败的清单当作隔离证据。若使用独立账户且开发账户无法读取官方目录，应由官方账户执行快照。

### 6.2 构建之后：从实际 Pi 模块核对路径

将下面文件保存为 `PI_DEV_ROOT/workspaces/pi/isolation-probe.mjs`，验证后删除这个临时探针，不提交到自己的代码改动中。

```javascript
import { homedir } from "node:os";
import { join, resolve } from "node:path";
import { getAgentDir, getAuthPath, getSettingsPath, getModelsPath, getSessionsDir } from "./packages/coding-agent/dist/config.js";

const agent = resolve(process.env.PI_CODING_AGENT_DIR);
const actual = {
  home: homedir(), agent: getAgentDir(), auth: getAuthPath(),
  settings: getSettingsPath(), models: getModelsPath(), sessions: getSessionsDir(),
  explicitSessions: process.env.PI_CODING_AGENT_SESSION_DIR,
};
const expected = {
  home: process.env.HOME, agent, auth: join(agent, "auth.json"),
  settings: join(agent, "settings.json"), models: join(agent, "models.json"),
  sessions: join(agent, "sessions"), explicitSessions: join(agent, "sessions"),
};
for (const key of Object.keys(expected)) {
  if (resolve(actual[key]) !== resolve(expected[key])) throw new Error(`Unexpected ${key}: ${actual[key]}`);
}
for (const key of ["OPENAI_API_KEY", "ANTHROPIC_API_KEY", "AWS_PROFILE", "SSH_AUTH_SOCK", "PI_PACKAGE_DIR", "PI_SELF_UPDATE_COMMAND", "NODE_OPTIONS", "BASH_ENV"]) {
  if (process.env[key]) throw new Error(`Unexpected inherited variable: ${key}`);
}
console.log(JSON.stringify(actual, null, 2));
```

Linux：

```bash
cd "$PI_DEV_ROOT/workspaces/pi" || exit 1
pi_dev exec "$PI_DEV_NODE" "$PI_DEV_ROOT/workspaces/pi/isolation-probe.mjs"
pi_dev npm config get cache
pi_dev npm config get prefix
```

Windows：

```powershell
Set-Location (Join-Path $PI_DEV_ROOT 'workspaces/pi')
Invoke-PiDev exec $PI_DEV_NODE (Join-Path $PI_DEV_ROOT 'workspaces/pi/isolation-probe.mjs')
Invoke-PiDev npm config get cache
Invoke-PiDev npm config get prefix
```

输出必须全部位于开发根目录。探针读取的是构建后的 Pi 路径函数；`paths` 则是启动器自身的配置报告，两者应相符。探针不调用模型，也不读取凭据内容。

### 6.3 检查默认数据、参数保护及父进程环境

1. 在空的开发 `demo` 项目启动：不应出现官方认证、官方会话或官方扩展。用独立账户登录并产生一个有辨识度的测试会话，确认新文件只出现在开发 `agent/`。不想产生费用时，可先只完成无模型请求的启动与路径检查，把真实会话验收留到有测试凭据时。
2. `pi_dev run --session-dir <官方目录>` / `Invoke-PiDev run --session-dir <官方目录>` 应被拒绝；直接 `--session`、`--fork`、`--export` 同样被这个示例入口拒绝。`--resume` 只能列出开发数据目录中的会话。不要向开发目录导入官方会话，因为其头部可能记录官方项目 cwd。
3. 在开发 `workspaces` 之外启动 `run` / `npm` / `exec` 应被拒绝。私有目录如果被链接到根目录外，也应被拒绝。祖先有上下文文件时应停止并换位置。
4. 退出后，在原终端检查 `HOME` / `USERPROFILE`、PATH、`PI_CODING_AGENT_DIR` 与 `PI_CODING_AGENT_SESSION_DIR`；它们应保持启动前的值。重新 `type -a pi` 或 `Get-Command pi -All`，官方命令解析应与基线相同。
5. 关闭开发版不会按名字批量杀死所有 `pi`。不要使用 `pkill pi` / `taskkill /IM pi.exe` 来做并存测试；只退出当前界面或终止已确认的单个进程。扩展启动的外部服务需自行管理，它们不在本手册默认 CLI 进程模型内。

这份入口对直接会话路径作了保守限制，目的是减少误操作，并不修改 Pi 本身的能力。为了避免自行解析整套 CLI 参数，限制也会拒绝在 `--` 后单独出现的同名参数字符串。确实需要导出开发会话时，应先审查输入/输出路径，在隔离账户内使用原 CLI；不要为了方便把官方目录加入启动器允许范围。

### 6.4 比较官方数据是否发生变化

仍保持官方版关闭，重新执行快照，输出改为 `after.json`。例如 Linux：

```bash
"$PI_DEV_NODE" "$PI_DEV_ROOT/snapshot.mjs" "$PI_DEV_ROOT/reports/after.json" \
  "$PI_OFFICIAL_AGENT_DIR" "$PI_OFFICIAL_INSTALL_DIR"
cmp "$PI_DEV_ROOT/reports/before.json" "$PI_DEV_ROOT/reports/after.json"
```

Windows：

```powershell
& $PI_DEV_NODE (Join-Path $PI_DEV_ROOT 'snapshot.mjs') (Join-Path $PI_DEV_ROOT 'reports/after.json') `
    $PI_OFFICIAL_AGENT_DIR $PI_OFFICIAL_INSTALL_DIR
if ($LASTEXITCODE -ne 0) { throw 'Snapshot failed' }
$PI_BEFORE_HASH = (Get-FileHash (Join-Path $PI_DEV_ROOT 'reports/before.json') -Algorithm SHA256).Hash
$PI_AFTER_HASH = (Get-FileHash (Join-Path $PI_DEV_ROOT 'reports/after.json') -Algorithm SHA256).Hash
if ($PI_BEFORE_HASH -ne $PI_AFTER_HASH) { throw 'Official files changed; inspect the manifests' }
```

如果基线还包含额外路径，`after` 也必须包含它们。清单相同说明**本次操作、已列入范围的文件**没有内容或所记录元数据变化；它不能证明不存在一次读取、短暂写入后恢复或未覆盖路径的访问。需要访问层面的保证，使用第 2.1 节的权限边界。

校验通过后，再分别打开官方版与开发版：官方仍显示自己的认证和历史，开发版只显示开发数据。此时官方正常运行会写入自己的状态，不能再拿这次变化误判为开发版污染。

## 7. 日常开发、升级、回退和移除

### 7.1 日常修改

在开发源码目录修改后，通过隔离入口运行：

| 操作 | Linux | PowerShell |
| --- | --- | --- |
| 源码检查 | `pi_dev npm run check` | `Invoke-PiDev npm run check` |
| 已有模型数据时构建 | `pi_dev npm run build:offline` | `Invoke-PiDev npm run build:offline` |
| 完整构建 / 刷新模型数据 | `pi_dev npm run build` | `Invoke-PiDev npm run build` |
| 启动开发版 | `pi_dev run` | `Invoke-PiDev run` |
| 恢复开发会话 | `pi_dev run --resume` | `Invoke-PiDev run --resume` |

`npm run check` 包含格式修正，会修改工作副本；先检查自己的工作区。遵守仓库 `AGENTS.md`，不要把全量 `npm test` 当成必要构建步骤；其中某些测试在存在凭据时可能触发真实服务。修改文档本身不需要跑应用检查。

若需要直接运行 TypeScript，可研究仓库的 `pi-test.*` 与 `scripts/source-resolver.mjs`，但必须继续在本文隔离环境内执行。本文主流程使用编译后的 Node 入口，避免把直接源码运行和产物验证混在一起。

### 7.2 更新源码

源码版通过 `git fetch`、审查目标提交、正常切换/合并后重新构建。所有 Git 与 npm 操作继续通过隔离入口执行。独立 Git 配置不继承真实用户的身份或凭据；需要提交时，只在这个副本里设置 Git 身份，需要访问私库时单独配置开发凭据。

先停止开发 Pi，保存自己未提交的改动，记录当前 Git SHA、Node 版本和锁文件哈希；然后更新。依赖/锁文件变化后重新 `npm ci --ignore-scripts`；需要新的模型数据时用完整构建。检查 `git diff`，不要用 `git reset --hard` 或 `git clean` 清掉自己的修改。

示例入口拒绝 `run update`，避免混淆源码更新与官方更新。也不要设置指向官方命令的 `PI_SELF_UPDATE_COMMAND` 或全局 `settings.selfUpdateCommand`。源码 checkout 通常不会被识别为可自更新的官方全局安装，但这不是隔离保证的依据。

### 7.3 回退

回退程序时使用已记录的源码提交重新构建；更稳妥的方法是保留升级前的**整个独立根目录备份**，包括工具链、依赖、忽略的模型数据与开发状态。备份必须是普通副本或独立快照，不能与运行目录共享可写硬链接。

不要让两个进程同时使用同一份开发 `agent/` 做跨版本迁移实验。程序版本回退不等于数据格式回退；需要恢复开发数据时，在开发进程停止后恢复相匹配的开发备份，绝不操作官方数据。

### 7.4 移除开发版

关闭开发进程及其明确启动的外部服务，核对 `paths` 输出与根目录实际位置，备份需要保留的开发会话，然后仅移除选定的 `PI_DEV_ROOT` 和自己添加的开发函数/快捷方式。不要运行官方卸载器，不要删除正常用户的 `~/.pi`、Windows 用户 `.pi` 或全局 npm 包。

本文不提供带变量递归删除的“一键卸载”命令，以免变量为空或路径误填时影响其他安装。移除后再核对官方命令解析与官方数据。

## 8. 常见问题与验证范围

| 现象 | 原因与处理 |
| --- | --- |
| 缺少 workspace 的 `dist` / 找不到内部包 | 在 monorepo 根目录安装和完整构建，不能只构建 coding-agent |
| 首次 `build:offline` 报缺少模型数据 | 先完成联网 `build`；不要禁用 `check:model-data` 或伪造空数据 |
| 模型目录下载失败 | 检查网络、代理和 CA；白名单保留代理/证书配置，但不继承 provider key |
| npm 找不到 registry 或私库认证 | 开发版不读取官方 npmrc；在开发 `npm/user.npmrc` 单独配置，不复制带凭据的官方文件 |
| 原生依赖加载失败 | 检查 Node/CPU/OS 是否匹配、可选依赖是否被省略；按 lockfile 重新安装，不要直接开启所有安装脚本 |
| `min-release-age` 配置警告 | 根 `.npmrc` 含包管理器相关选项，不同 npm 版本支持不同；核对工具版本，不能把警告或失败包装为通过 |
| 看得到官方登录 / 会话 | 立即退出，检查是否绕过启动器、继承配置、链接数据目录或导入了官方文件；重新做快照与路径探针 |
| `External ancestor context` | 独立根之外有指令文件；换干净目录或独立账户，不删除原文件 |
| Windows 找不到 Bash | 安装 Git for Windows，或只在开发设置里指定它的 `bash.exe` |
| Windows `EPERM` / 文件被占用 | 关闭开发进程，检查杀毒软件锁定与权限；不要通过管理员全局安装绕过 |
| 两版版本号完全相同 | 正常；识别依据是入口、构建提交与数据路径，不是版本字符串 |
| 登录 OAuth 时选择了官方账号 | 浏览器登录态不受子进程 HOME 隔离；使用独立浏览器配置/系统账户和测试账户 |

**本文编写时的验证范围：**已核对所列源码基线的构建、路径、会话优先级、信任与资源发现实现。在 Linux / Node 24.22.1 上完成 35 项文档校验：3 个 Node 示例的语法、8 段 Bash 命令的语法、启动器参数与环境隔离、目录/链接越界拦截、会话路径限制、退出码、快照变更检测以及源码/文档链接。行为校验使用临时夹具 CLI，不代表 Pi 产品构建已经通过。

本次未执行完整依赖安装、完整产品构建、真实供应商登录、PowerShell 运行或原生 Windows 交互验证。第 6.2 节的真实 Pi 路径探针也需要在读者完成构建后运行。第 6 节是各自在目标机器上必须完成的验收步骤。

## 9. 固定源码索引

以下链接均固定到本文基线，避免分支更新后定位漂移：

| 依据 | 文件 |
| --- | --- |
| workspace、Node 下限、构建顺序 | [根 package.json](https://github.com/AManHasNoName12138/pi/blob/6b339afc92e1dc6574f3234e1850dff36c77294b/package.json) |
| CLI 版本、正式入口、资源复制 | [coding-agent/package.json](https://github.com/AManHasNoName12138/pi/blob/6b339afc92e1dc6574f3234e1850dff36c77294b/packages/coding-agent/package.json) |
| Node bundle、拆分资源与外部依赖 | [build-coding-agent-bundle.mjs](https://github.com/AManHasNoName12138/pi/blob/6b339afc92e1dc6574f3234e1850dff36c77294b/scripts/build-coding-agent-bundle.mjs) |
| 模型数据生成与离线构建前提 | [ai/package.json](https://github.com/AManHasNoName12138/pi/blob/6b339afc92e1dc6574f3234e1850dff36c77294b/packages/ai/package.json)、[catalog/README.md](https://github.com/AManHasNoName12138/pi/blob/6b339afc92e1dc6574f3234e1850dff36c77294b/packages/ai/catalog/README.md) |
| 用户数据、资源和更新命令覆盖 | [config.ts](https://github.com/AManHasNoName12138/pi/blob/6b339afc92e1dc6574f3234e1850dff36c77294b/packages/coding-agent/src/config.ts) |
| 会话目录优先级与会话 cwd 切换 | [main.ts](https://github.com/AManHasNoName12138/pi/blob/6b339afc92e1dc6574f3234e1850dff36c77294b/packages/coding-agent/src/main.ts)、[session-manager.ts](https://github.com/AManHasNoName12138/pi/blob/6b339afc92e1dc6574f3234e1850dff36c77294b/packages/coding-agent/src/core/session-manager.ts) |
| 祖先指令、项目资源与信任 | [resource-loader.ts](https://github.com/AManHasNoName12138/pi/blob/6b339afc92e1dc6574f3234e1850dff36c77294b/packages/coding-agent/src/core/resource-loader.ts)、[configuration.md](https://github.com/AManHasNoName12138/pi/blob/6b339afc92e1dc6574f3234e1850dff36c77294b/packages/coding-agent/docs/configuration.md) |
| 用户和项目设置、会话目录设置 | [settings-manager.ts](https://github.com/AManHasNoName12138/pi/blob/6b339afc92e1dc6574f3234e1850dff36c77294b/packages/coding-agent/src/core/settings-manager.ts)、[settings.md](https://github.com/AManHasNoName12138/pi/blob/6b339afc92e1dc6574f3234e1850dff36c77294b/packages/coding-agent/docs/settings.md) |
| 模型目录缓存 | [models-store.ts](https://github.com/AManHasNoName12138/pi/blob/6b339afc92e1dc6574f3234e1850dff36c77294b/packages/coding-agent/src/core/models-store.ts) |
| MCP 配置来源 | [extensions/mcp/config.ts](https://github.com/AManHasNoName12138/pi/blob/6b339afc92e1dc6574f3234e1850dff36c77294b/packages/coding-agent/src/extensions/mcp/config.ts) |
| Windows shell 选择 | [utils/shell.ts](https://github.com/AManHasNoName12138/pi/blob/6b339afc92e1dc6574f3234e1850dff36c77294b/packages/coding-agent/src/utils/shell.ts) |
| 直接源码运行入口 | [pi-test.sh](https://github.com/AManHasNoName12138/pi/blob/6b339afc92e1dc6574f3234e1850dff36c77294b/pi-test.sh)、[pi-test.ps1](https://github.com/AManHasNoName12138/pi/blob/6b339afc92e1dc6574f3234e1850dff36c77294b/pi-test.ps1) |

项目结构与学习路线另见 [开发者入门指南](../docs/UA_ONBOARDING.md) 和 [知识图谱说明](../.ua/README.md)。
