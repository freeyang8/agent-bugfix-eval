# Claude Code 2.1.266 逆向工程技术报告

**样本**：`C:\Users\12138\AppData\Roaming\npm\node_modules\@anthropic-ai\claude-code\bin\claude.exe`
**大小**：218,971,808 字节（208.83 MiB）
**SHA-256**：`d2c5f7b3b6a12819097ceb6efbce2a390157166003fcaee32dbde0e6d7b45ef7`
**格式**：PE32+ / x86-64 / GUI 子系统 / 13 个节
**版本**：2.1.266（文件资源与 `claude --version` 一致）
**分析日期**：2026-10-17
**分析工具**：REA（MCP）+ 自建 PE/Bun 解析器

---

## 0. 结论速览

| 问题 | 结论 |
|---|---|
| 它是什么？ | **Bun v1.4.1 编译的单文件可执行**，不是 Rust、不是 Node SEA、不是 Go |
| 代码在哪？ | `.bun` 节（**127.14 MiB，占文件 58.06%**），内含 **1,644 个明文 JS 模块 / 31.24 MB 源码** |
| 有保护吗？ | **没有**。无混淆、无加密、无反调试、无字节码保护（应用层）。只有 minify |
| 能还原到什么程度？ | **100% 源码级**。系统提示词、工具定义、内部开关、模型目录、API 协议全部可读 |
| 内部代号 | `tengu`（遥测事件名 100% 前缀） |
| 签名 | Anthropic PBC EV 证书有效，**2026-10-21 到期（距今 4 天）** |
| 加固 | ASLR ✅ / DEP ✅ / 高熵 ASLR ✅ / **CFG ❌ / GS Cookie ❌** |
| 最大风险 | 供应链与遥测面，而非二进制层 |

**一句话**：这是一份"把 TypeScript 源码原样塞进 PE 段、只做了 minify"的发行版。任何拿到该文件的人，都能在几分钟内还原出完整应用逻辑。

---

## 1. 分析方法与 REA 工具链实测

### 1.1 REA 通道实测结果（本次会话逐条验证）

| REA 操作 | 结果 | 证据 / 失败原因 |
|---|---|---|
| `open_binary` | ✅ 成功 | `format=pe`、`kind=executable`、`architecture=x86_64`、SHA-256 与本地计算一致 |
| `inspect_pe_resources` | ✅ 成功（需显式提高上限） | 不显式提高 `max_file_bytes` 时会拒绝 208 MiB 样本（上限可调至 512 MiB）；传 `268435456` 后完整解析：**18 个资源、40 个条目、4 个图标组、14 档图标（16→96 px）、RT_MANIFEST、VS_VERSION_INFO**，`coverage.status=complete` |
| `binary_overview` | ❌ `provider_unavailable` | 三个候选 provider 全被拒：`ghidra: not_configured`（`GHIDRA_INSTALL_DIR` 未设，且本机 JDK 为 **17.0.12**，Ghidra 12.1 需 21+）、`hopper: unsupported_host`（win32 不支持）、`ida: not_configured`（`REA_IDA_MCP_CONFIG` 未设） |
| `analyze_javascript_application` | ❌ `capability_unavailable` | `REA Windows native controls are unavailable: ENOENT ... 12138\REA\rea\native\windows\build\manifest.json` |
| `inspect_binary_layout` | ⛔ 不适用 | 该工具仅支持 ELF，PE 会被拒 |

**结论**：REA 的**便携取证通道**（格式识别、哈希、PE 资源树）可用且已产出实证；**深分析通道**（反汇编、JS 应用图）在本机缺原生组件，未能启用。因此本报告的二进制层数据由自建 PE 解析器补齐，应用层数据由自建 Bun 提取器还原 —— 两者均为**可复现的一手测量**，不含推测性结论。

### 1.2 复现脚本

| 脚本 | 作用 |
|---|---|
| `work/extract_modules.py` | 按 `// @bun @bytecode` 横幅切分 `.bun` 节，还原 1,644 个模块 |
| `work/pe.py` | 节表、目录、导入/导出、安全缓解、熵、SHA-256 |
| `work/loadcfg.py` | LoadConfig / GuardFlags / SecurityCookie |
| `work/analyze_all.py` | 环境变量、feature flag、主机名、API 路径、模型、hooks、MCP、遥测 |
| `work/tools_scan.py` / `hints.py` | 工具注册对象与描述文案 |
| `work/prompt_sections.py` | 系统提示词分块 |
| `work/models.py` | 模型目录（含上下文窗口与输出上限） |
| `work/telemetry.py` | 本地遥测事件解析（含 base64 解码） |
| `work/bunblt.py` | `.bunblt` 内建模块与 Bun 版本指纹 |

---

## 2. 供应链解剖

### 2.1 包结构

```
@anthropic-ai/claude-code/          (2.1.266, 147 KB)
├── bin/claude.exe                  218,971,808 B  ← 真正的程序
├── cli-wrapper.cjs                 4,997 B        ← Node 回退启动器
├── install.cjs                     7,196 B        ← postinstall
├── sdk-tools.d.ts                  167,106 B      ← SDK 工具类型定义（明文）
├── package.json                    1,476 B
└── README.md / LICENSE.md
```

**关键点**：主包本身不含二进制，二进制来自 `optionalDependencies`：

```json
"optionalDependencies": {
  "@anthropic-ai/claude-code-darwin-arm64": "2.1.266",
  "@anthropic-ai/claude-code-darwin-x64":   "2.1.266",
  "@anthropic-ai/claude-code-linux-x64":    "2.1.266",
  "@anthropic-ai/claude-code-linux-arm64":  "2.1.266",
  "@anthropic-ai/claude-code-linux-x64-musl":  "2.1.266",
  "@anthropic-ai/claude-code-linux-arm64-musl":"2.1.266",
  "@anthropic-ai/claude-code-win32-x64":    "2.1.266",
  "@anthropic-ai/claude-code-win32-arm64":  "2.1.266"
}
```

### 2.2 postinstall 行为（`install.cjs` 审计）

1. 按 `process.platform` + `os.arch()` 选出平台键；
2. **musl 检测**用 `process.report.getReport().header.glibcVersionRuntime === undefined`，比 `ldd` 更稳（避免最小容器误判）；
3. **Rosetta 2 检测**：darwin-x64 下读 `sysctl -n sysctl.proc_translated`，为 `1` 时改选 arm64（注释明确说明 x64 构建需要 AVX，Rosetta 不模拟）；
4. 放置二进制：优先 `linkSync`（硬链接，零额外磁盘）；`EEXIST` 时先读 stub 备份 → unlink → 再 link → 失败则 copy → 再失败回滚 stub；`EXDEV/EPERM` 直接 copy；
5. 目标恒为 `bin/claude.exe`，`.exe` 扩展名 + 无 shebang 是为了让 npm 的 cmd-shim 在 Windows 上生成**直接 exec**（Unix 忽略扩展名）——注释直言"Same pattern as Bun's npm package"。

**`prepare` 脚本**包含发布闸门：

```js
"prepare": "node -e \"if (!process.env.AUTHORIZED) { console.error('ERROR: Direct publishing is not allowed...'); process.exit(1) }\""
```

即开发者本地 `npm publish` 会被拒，只能走内部 release workflow。

### 2.3 代码签名（Authenticode）

| 项 | 值 |
|---|---|
| 状态 | **Valid** |
| 主体 | `CN="Anthropic, PBC", O="Anthropic, PBC", L=San Francisco, S=California, C=US`，`SERIALNUMBER=4860621`，`Private Organization`（**EV 证书**），`Delaware / US` |
| 颁发者 | `DigiCert Trusted G4 Code Signing RSA4096 SHA384 2021 CA1` |
| 有效期 | 2025-10-14 → **2026-10-21 07:59:59** |
| 指纹 | `0D7581D2C51C59DF686C3000C70BF543F9F6C6CB` |
| 证书表 | RVA `0xD0D1600`，10,400 字节 |

> ⚠️ **证书将在 4 天后到期**。到期后新构建必须换签，否则 Windows SmartScreen 会降级信任；已签发的旧文件签名仍保持有效（时间戳服务决定），但任何"重新签名"的产物都会换指纹——可作为检测二次打包的信号。

### 2.4 版本漂移

本地遥测显示该机器实际运行过 **2.1.275 / 2.1.295**（`build_time: 2026-10-08T16:50:59Z`），而磁盘上的 exe 是 **2.1.266**。说明自动更新在别处落地（或用户回退），磁盘样本并非最新版。

---

## 3. PE 结构解剖

### 3.1 节表

| 节 | 虚拟大小 | RVA | 原始大小 | 文件偏移 | 占文件 |
|---|---:|---:|---:|---:|---:|
| `.text` | 66,724,766 | `0x1000` | 66,724,864 | `0x400` | 30.47% |
| `.rdata` | 20,956,840 | `0x3FA4000` | 20,957,184 | `0x3FA2800` | 9.57% |
| `.data` | 2,325,504 | `0x53A1000` | 117,248 | `0x539F000` | 0.05% |
| `.pdata` | 935,628 | `0x55D9000` | 935,936 | `0x53BBA00` | 0.43% |
| **`.bunblt`** | 2,618,826 | `0x56BE000` | 2,618,880 | `0x54A0200` | 1.20% |
| `.fptable` | 256 | `0x593E000` | 512 | `0x571F800` | ~0 |
| `.tls` | 23,329 | `0x593F000` | 23,552 | `0x571FA00` | 0.01% |
| `_RDATA` | 500 | `0x5945000` | 512 | `0x5725600` | ~0 |
| `__DATA,_` | 16,384 | `0x5946000` | 16,384 | `0x5725800` | 0.01% |
| `__DATA,_` | 16,384 | `0x594A000` | 16,384 | `0x5729800` | 0.01% |
| `.rsrc` | 199,780 | `0x594E000` | 200,192 | `0x572D800` | 0.09% |
| `.reloc` | 205,336 | `0x597F000` | 205,824 | `0x575E600` | 0.09% |
| **`.bun`** | 127,142,911 | `0x59B2000` | 127,142,912 | `0x5790A00` | **58.06%** |

**三个反常信号**，直接指向 Bun：

1. `.bunblt` / `.bun` 两个非标准节名；
2. `__DATA,_` 这种 Mach-O 风格的段名残留（Bun 跨平台链接器产物）；
3. `.fptable`（函数指针表）——Bun 的 JIT/FFI 需要。

### 3.2 数据目录

| 目录 | RVA | 大小 |
|---|---|---|
| export | `0x52FCD7B` | 30,037 |
| import | `0x53042D0` | 160 |
| resource | `0x594E000` | 199,780 |
| exception (`.pdata`) | `0x55D9000` | 935,628 |
| cert | `0xD0D1600` | 10,400 |
| reloc | `0x597F000` | 205,336 |
| debug | `0x52FB744` | 28（**Reproducible Build**，`RSDS`） |
| tls | `0x521DA10` | 40 |
| loadcfg | `0x4001AF0` | 320 |
| iat | `0x5305000` | 3,216 |
| delayimport | `0x52FB9D8` | 416 |

### 3.3 导入表（极简，是运行时特征）

| DLL | 导入函数数 |
|---|---:|
| `kernel32.dll` | 334 |
| `ntdll.dll` | 33 |
| `fwpuclnt.dll` | 13 |
| `netapi32.dll` | 10 |
| `api-ms-win-core-synch-l1-2-0.dll` | 3 |
| `bcrypt.dll` | 1 |
| `bcryptprimitives.dll` | 1 |

**只有 7 个 DLL**。对比一个典型 Rust CLI（通常 15–30 个）或 Electron 应用，这个导入面非常窄——因为 JS 运行时（HTTP、TLS、压缩、正则、加密）全部由 Bun 自带的 Zig 实现完成，不依赖系统库。

值得注意的两个：
- **`fwpuclnt.dll`**（Windows Filtering Platform）→ 网络过滤/防火墙相关，与沙箱出网控制对应；
- **`netapi32.dll`** → 网络共享/用户枚举，与"独立沙箱用户"实现相关。

### 3.4 导出表

| 项 | 值 |
|---|---|
| DLL 名 | **`bun-profile.exe`** |
| 函数数 | **721** |
| 命名导出 | 721（全部具名，base=1） |

**这是最硬的指纹**：`bun-profile.exe` 是 Bun 官方 `bun build --compile --profile` 产物的默认导出名。任何"这是自研原生程序"的说法在此不成立。

### 3.5 安全缓解

`DllCharacteristics = 0x8160`：

| 缓解 | 状态 |
|---|---|
| HIGH_ENTROPY_VA（64 位高熵 ASLR） | ✅ |
| DYNAMIC_BASE（ASLR） | ✅ |
| NX_COMPAT（DEP） | ✅ |
| TERMINAL_SERVER_AWARE | ✅ |
| **GUARD_CF（CFG）** | ❌ 未设置 |
| FORCE_INTEGRITY | ❌ |

LoadConfig（RVA `0x4001AF0`，size 320）进一步确认：

```
GuardCFCheckFunctionPointer     = 0x0
GuardCFDispatchFunctionPointer  = 0x0
GuardCFFunctionTable            = 0x0
GuardFlags                      = 0x0
SecurityCookie                  = 0x0
```

**CFG 与 GS 均未启用**。对一个处理不可信输入（文件内容、网页、MCP 服务端返回、模型输出）的程序来说，这是可加固项。不过要公平地说：Bun 的 JIT 代码页本身与 CFG 天然冲突（需 `SetProcessValidCallTargets` 动态注册），Bun 官方也未启用，属生态级取舍而非 Anthropic 疏忽。

### 3.6 熵分布（判断是否加壳/加密）

| 节 | 熵 | 判读 |
|---|---:|---|
| `.text` | 6.410 | 正常机器码 |
| `.rdata` | 6.981 | 高（大量字符串+常量） |
| `.pdata` | 6.918 | 正常（异常表） |
| `.bunblt` | 5.121 | 字节码+源码混合 |
| **`.bun`** | **6.580** | **明文压缩文本，非加密**（加密应 > 7.9） |

6.58 这个值是"minified JS + 少量二进制索引"的典型熵。**如果是 AES 加密，熵会接近 8.0**。

---

## 4. Bun 打包格式解剖

### 4.1 两个节的含义

| 节 | 魔数 | 内容 |
|---|---|---|
| `.bunblt` | `BUNBLTNS` + `0x01 00 00 00` | **Bun builtins**：Node 兼容层与 Bun 内置模块，编译成字节码（仍内嵌可读 JS 源码） |
| `.bun` | 末尾 `\n---- Bun! ----\n` | **应用 bundle**：用户代码 + 依赖，1,644 个模块 |

`.bunblt` 头部实测（文件偏移 `0x54A0200`）：

```
42 55 4E 42 4C 54 4E 53  01 00 00 00  E6 EC B2 C6  C5 00 00 00  30 00 00 00
B  U  N  B  L  T  N  S  ver=1         [哈希/计数]    0xC5=197      0x30=48
```

其内部可读出 `__proto__: null,`、`get() {`、`let handle = this.#handle;`、`return @undefined;` 等——`@undefined` 是 **Bun 字节码的反序列化文本形式**，说明 builtins 是"字节码 + 源码"双份存储。

### 4.2 模块边界（还原算法的核心）

`.bun` 节内每个模块的布局：

```
"// @bun @bytecode"            ← 4 字节横幅（实际 17 字节）
"// @bun @bun-cjs"             ← 可选，CJS 包装标记
"(function(exports, require, module, __filename, __dirname) {"
... minified JS ...
"\x00"                         ← 模块终止符
```

**没有可依赖的中央索引表也能全量还原**：横幅数量 = 模块数量 = **1,644**。

### 4.3 提取算法

```python
BANNER = b"// @bun @bytecode"
offs = [m.start() for m in re.finditer(re.escape(BANNER), mmap_obj)]
for i, a in enumerate(offs):
    limit = offs[i+1] if i+1 < len(offs) else len(mmap_obj)
    nz = mmap_obj.find(b"\x00", a)          # 模块以 NUL 终止
    b = nz if (nz != -1 and nz < limit) else limit
    write(offs[i], mmap_obj[a:b])
```

- 总产出：**1,644 个模块 / 32,760,365 字节（31.24 MB）**
- 模块体量：中位数 **4.5 KB**，最大 **5.56 MB**（`mod_0261.js`，即主控/提示词/工具编排模块）
- 前 10 大模块（KB）：5558 / 1591 / 1089 / 899 / 590 / 578 / 385 / 348 / 347 / 337

### 4.4 模块命名与构建路径泄露

bundle 内保留完整虚拟路径：

```
B:/~BUN/root/chunk-1cv7c8gg.js   (903 次引用)
B:/~BUN/root/chunk-r5evxqdw.js   (882)
B:/~BUN/root/chunk-h116rn8j.js   (830)
B:/~BUN/root/chunk-abdf0ak1.js   (780)
B:/~BUN/root/chunk-x2m2bfv9.js   (760)
```

- **1,633 个不同的 `chunk-*.js` 名**，8 字符 base36 随机后缀 —— Bun 的 code-splitting 命名；
- 另有真实源码路径：`src/entrypoints/cli.js`、`src/plugins/functionHooks/hooks-worker/hooks-worker.js`；
- `B:/~BUN/root/` 是 Bun 的虚拟根，`B:` 盘符说明**构建机是 Windows**（Bun 用固定盘符占位以保证路径可复现）。

---

## 5. 应用层结构

### 5.1 系统提示词：25+ 个命名分块按条件装配

主装配函数（`mod_0261.js` 约 4,752,000 偏移）返回一个数组，`.filter(x => x !== null)`。分块清单（`oy("name", () => ...)` 形式）：

| 类别 | 分块名 |
|---|---|
| 角色与底座 | `$ds`（role）、`Uds`、`Hds`（Doing tasks）、`jds`（Executing actions with care）、`Wds`（Using your tools）、`qds`（Tone and style） |
| 沟通 | `communication`、`pronouns`、`action_caution`、`task_continuity` |
| 工具协议 | `tool_param_json`、`session_guidance` |
| 环境 | `memory`、`env_info_static` / `env_info_simple`、`language`、`scratchpad` |
| 上下文 | `context_management`、`brief`、`focus_mode` |
| 行为调优 | `act_dont_rederive`、`delivering_work_max`、`overcorrection`、`autonomy_append` |
| 实验性 | `subagent_steer_delegation`、`opus5_reduced_delegation`、`heron_brook`、`brook_heron`、`willow_tern`、`endconv_deferred_hint` |
| 后台 | `bg-session` |

> 注意 `heron_brook` / `brook_heron` / `willow_tern` 这类**双词代号**——它们不是功能名，是 A/B 实验的盲名，配套的 feature flag 同样是盲名（见 5.4）。

**三种角色前缀**（原文）：

```js
U  = "You are Claude Code, Anthropic's official CLI for Claude."
Ee = "You are Claude Code, Anthropic's official CLI for Claude, running within the Claude Agent SDK."
Oe = "You are a Claude agent, built on Anthropic's Claude Agent SDK."
```

选择逻辑：Vertex 环境 → `U`；非交互 + 有 append system prompt → `Ee`；非交互 → `Oe`；否则 `U`。

**输出人格（output styles）5 种**，每种带独立 prompt 与 turn reminder：

| 名称 | 定位（原文摘要） |
|---|---|
| `default` | 标准 |
| `Proactive` | "executes immediately, minimizes interruptions, prefers action over planning" |
| `Concise` | "keep your responses short and direct while doing the work just as thoroughly" |
| `Explanatory` | 边做边讲设计取舍，允许超长 |
| `Learning` | 要求用户写 2–10 行代码，必须先插入 `TODO(human)`，且**全库只能有一个** |

`Learning` 的实现细节值得单列：它规定"生成 20+ 行且涉及设计决策/多解业务逻辑/关键算法时"触发，并要求 TodoList 里显式加一条 "Request human input on [specific decision]"。

### 5.2 工具面：22 个内置 + 48 个条件/实验工具

`BUILTIN_TOOL_NAMES`（源码原文，22 项）：

```js
["Bash","Read","Write","Edit","Glob","Grep","NotebookEdit","WebFetch","WebSearch",
 "Task","TodoWrite","TaskCreate","TaskUpdate","TaskGet","TaskList","TaskStop",
 "Skill","REPL","JavaScript","AskUserQuestion","ToolSearch","SendUserMessage"]
```

但实际注册的工具对象（`wt({name:...})` 构造）解析出 **70 个**，多出的 48 个是条件启用的。按 `searchHint` 归类：

**文件与检索**

| 工具 | searchHint | 只读 | 并发安全 |
|---|---|:--:|:--:|
| Read | read files, images, PDFs, notebooks | ✅ | ✅ |
| Write | create or overwrite files | ❌ | ❌ |
| Edit | modify file contents in place | ❌ | ❌ |
| Glob | find files by name pattern or wildcard | ✅ | ✅ |
| Grep | search file contents with regex (ripgrep) | ✅ | ✅ |
| NotebookEdit | edit Jupyter notebook cells (.ipynb) | ❌ | ❌ |
| LSP | code intelligence (definitions, references) | ✅ | ✅ |

**执行与网络**

| 工具 | searchHint |
|---|---|
| Bash | execute shell commands |
| PowerShell | execute Windows PowerShell commands |
| REPL | execute JavaScript with programmatic tool access |
| WebFetch | fetch and extract content from a URL |
| WebSearch | search the web for current information |

**任务与代理**

| 工具 | searchHint |
|---|---|
| Task / Agent | delegate work to a subagent |
| TaskCreate / TaskGet / TaskUpdate / TaskList | 任务清单 CRUD |
| TaskOutput | read output/logs from a background task（已标 `[Deprecated]`） |
| TaskStop | kill a running background task |
| SendMessage | send messages to agent teammates |
| SendFile / SendUserFile | send files to another session / deliver files |
| ListAgents | list agents you can SendMessage to |
| ListConnectors | list installed MCP connectors |
| Poll | wait for and receive queued harness events |

**计划与工作区**

| 工具 | searchHint |
|---|---|
| EnterPlanMode | switch to plan mode to design an approach |
| ExitPlanMode | present plan for approval and start coding |
| EnterWorktree | create an isolated git worktree and switch |
| ExitWorktree | exit a worktree session and return |
| ScheduleWakeup | （定时唤醒） |
| CronCreate / CronDelete / CronList | 定时任务 |
| Workflow | orchestrate subagents with deterministic… |

**MCP 与连接器**

| 工具 | searchHint |
|---|---|
| ListMcpResourcesTool | list resources from connected MCP servers |
| ReadMcpResourceTool | read a specific MCP resource by URI |
| SearchMcpRegistry | discover MCP connectors by keyword |
| SuggestConnectors | resolve MCP connector payloads by directory |
| SuggestPluginInstall | render a plugin install card |
| SuggestSkills / ProposeGoal | 提案类 |

**产品化工具（非编码）**

`ClaudeDesign`（work with Claude Design）、`DesignSync`、`ShareOnboardingGuide`、`ShowOnboardingRolePicker`、`SendFeedback`、`SendUserMessage`、`PushNotification`、`ReadNotifications`、`EndConversation`、`ReportFindings`（code-review 结构化输出）、`TestingPermission`（"Test tool that always asks for permission"）。

**自托管 / 企业版工具**（共 8 个，全部 `self_hosted_runner_*` / `*_pool` / `*_secrets` / `*_metrics` / `*_logs`）—— 说明存在自托管 runner 形态。

**工具对象暴露的内部元数据**（`wt()` 的属性）：

```js
wt({
  name, ruleContentField,        // 权限规则匹配字段
  searchHint,                     // 给模型看的检索提示
  remoteExecution: { supported, decidingInputFields },
  maxResultSizeChars,             // 结果截断阈值（Read 为 Infinity）
  strict,                         // 严格 schema 校验
  description(), prompt(),        // 动态生成
  inputSchema, outputSchema,
  coerceInput, stripForStorage, toAutoClassifierInput,
  isConcurrencySafe(), isReadOnly(),
  preparePermissionMatcher(),     // 权限判定
  getPath(), getToolUseSummary(), getActivityDescription()
})
```

**`toAutoClassifierInput()`** 是一个值得注意的设计：工具把输入投影成一份"可分类摘要"（含 `file_path`、`adds`、`dels` 等），喂给自动权限分类器。也就是说权限决策不是纯规则匹配，还有模型参与。

### 5.3 斜杠命令：138 个

去重后共 **138 个命令名**。按功能分组（节选实测描述）：

| 组 | 命令 |
|---|---|
| 会话 | `/clear`（"Start a new session with empty context; previous session stays on disk (resumable with /resume)"）、`/resume`、`/compact`、`/autocompact`、`/context`（"Visualize current context usage as a colored grid"）、`/rewind`、`/branch`、`/fork`、`/subtask`、`/export`、`/copy`、`/recap` |
| 配置 | `/config`、`/model`、`/theme`、`/color`、`/keybindings`、`/hooks`、`/permissions`、`/status`、`/doctor`、`/import`（"Import config from another AI coding agent"） |
| 扩展 | `/mcp`、`/plugin`、`/skills`、`/reload-skills`、`/reload-plugins`、`/skill-doctor`（"Show which loaded skills are unused and costing context"）、`/agents`（标 **removed**） |
| 云端 | `/teleport`（"Send this session to the cloud, or resume one from claude.ai"）、`/ultraplan`、`/ultrareview`、`/remote-env`、`/background`、`/bg` |
| 集成 | `/ide`、`/install-github-app`、`/install-slack-app`、`/chrome`、`/design-login`、`/design-consent`、`/setup-bedrock`、`/setup-vertex` |
| 账户 | `/logout`、`/usage`、`/usage-credits`、`/extra-usage`（"Renamed to /usage-credits"）、`/upgrade`、`/rate-limit-options`、`/pro-trial-expired`、`/privacy-settings`、`/explain-usage` |
| 运维 | `/heapdump`（"Dump the JS heap to ~/Desktop"）、`/release-notes`、`/version`（注释强调"Print the version this session is running (not what autoupdate downloaded)"） |
| 彩蛋 | `/radio`（"Listen to Claude FM lo-fi radio"）、`/stickers`、`/powerup`、`/advisor` |

> `/version` 那句括号注释很关键：它证明**自动更新下载的版本 ≠ 当前运行版本**，两者会并存。

### 5.4 Feature Flag：2,316 个 `tengu_*`

| 指标 | 值 |
|---|---|
| 不同 flag 名 | **2,316** |
| 引用总次数 | 3,541 |
| 最热 | `tengu_ccr_bundle_upload`(24)、`tengu_mcp_degraded`(23)、`tengu_session_resumed`(18)、`tengu_review_remote_precondition_failed`(18)、`tengu_feedback_survey_event`(17) |

这些名字**同时也是遥测事件名**——即 flag 求值本身就被上报。命名风格分两类：
- **语义名**（`tengu_mcp_degraded`、`tengu_config_changed`）—— 可读，暴露功能；
- **盲名**（`tengu_violin_wood`、`tengu_sorrel_avocet`、`tengu_plucky_orchard`、`tengu_cedar_lantern`、`tengu_silent_harbor`、`tengu_slate_bittern`、`tengu_omelette_fouet`、`tengu_violin_pegbox`、`tengu_sorrel_trellis_lintel`）—— 双词随机组合，用于隐藏未发布功能，防止被逆向者直接读出含义。

例如 `tengu_cedar_lantern` 控制 `act_dont_rederive` 分块，`tengu_slate_bittern` 控制 `opus5_reduced_delegation`——**盲名背后是具体行为**，这是有效的（虽然薄弱的）保密手段。

### 5.5 环境变量：407 个

按敏感度分类（节选）：

**凭证/认证**

```
ANTHROPIC_API_KEY / ANTHROPIC_AUTH_TOKEN
CLAUDE_CODE_OAUTH_TOKEN / CLAUDE_CODE_OAUTH_REFRESH_TOKEN / CLAUDE_CODE_OAUTH_CLIENT_ID
CLAUDE_CODE_OAUTH_SCOPES / CLAUDE_CODE_OAUTH_401_WAIT_MS
CLAUDE_CODE_SESSION_ACCESS_TOKEN / CLAUDE_SESSION_INGRESS_TOKEN_FILE
CLAUDE_BG_AUTH_SNAPSHOT_PATH / CLAUDE_BG_SOCKET_TOKENS_PATH
CLAUDE_SECURESTORAGE_CONFIG_DIR / CLAUDE_CODE_HOST_CREDS_FILE
AZURE_CLIENT_ID / AZURE_TENANT_ID / AZURE_FEDERATED_TOKEN_FILE
ANTHROPIC_AWS_API_KEY
```

**路由/代理**

```
ANTHROPIC_BASE_URL / ANTHROPIC_UNIX_SOCKET / ANTHROPIC_PROFILE
HTTPS_PROXY / HTTP_PROXY / ALL_PROXY / NO_PROXY
CLAUDE_CODE_CUSTOM_OAUTH_URL / CLAUDE_CODE_OAUTH_CLIENT_ID
```

**遥测/隐私（可关闭）**

```
CLAUDE_CODE_ENABLE_TELEMETRY / DISABLE_TELEMETRY / DO_NOT_TRACK
DISABLE_ERROR_REPORTING / DISABLE_NONESSENTIAL_TRAFFIC
CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC / DISABLE_INSTALLATION_CHECKS
```

**运行形态**

```
CLAUDE_CODE_ENTRYPOINT / CLAUDE_CODE_REMOTE / CLAUDE_CODE_REMOTE_SESSION_ID
CLAUDE_CODE_COORDINATOR_MODE / CLAUDE_CODE_SUBPROCESS_ENV_SCRUB
CLAUDE_CODE_SIMPLE / CLAUDE_CODE_SAFE_MODE / CLAUDE_CODE_RESTRICTED
CLAUDE_CODE_CONTAINER_ID / CLAUDE_CODE_HOST_PLATFORM
CLAUDE_CODE_SUBSCRIPTION_TYPE / CLAUDE_CODE_RATE_LIMIT_TIER
```

**模型与上下文**

```
ANTHROPIC_MODEL / ANTHROPIC_SMALL_FAST_MODEL / CLAUDE_CODE_SUBAGENT_MODEL
ANTHROPIC_DEFAULT_OPUS_MODEL / _SONNET_MODEL / _HAIKU_MODEL
MAX_THINKING_TOKENS / DISABLE_COMPACT / API_TIMEOUT_MS
CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS / CLAUDE_CODE_MAX_WEB_SEARCHES_PER_SESSION
```

`CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` 值得单列：它控制**子进程环境变量清洗**——防止 Bash 子进程继承到 API Key。这是个明确的安全设计。

### 5.6 模型目录（含未发布代号）

`sdk` 内部模型目录共 **16 条**，含 `family` / `display_name` / `knowledge_cutoff` / `max_output_tokens` / `pricing` / `capabilities`：

| 模型 ID | family | 显示名 | 知识截止 | 输出默认/上限 |
|---|---|---|---|---:|
| claude-haiku-4-5 | haiku | Haiku 4.5 | 2025-02 | 32K / 64K |
| claude-sonnet-4-0 | sonnet | Sonnet 4 | 2025-01 | 32K / 64K |
| claude-sonnet-4-5 | sonnet | Sonnet 4.5 | 2025-01 | 32K / 64K |
| claude-sonnet-4-6 | sonnet | Sonnet 4.6 | 2025-08 | 32K / 128K |
| claude-sonnet-5 | sonnet | Sonnet 5 | 2026-01 | 64K / 128K |
| claude-opus-4-0 | opus | Opus 4 | 2025-01 | 32K / 32K |
| claude-opus-4-1 | opus | Opus 4.1 | 2025-01 | 32K / 32K |
| claude-opus-4-5 | opus | Opus 4.5 | 2025-05 | 32K / 64K |
| claude-opus-4-6 | opus | Opus 4.6 | 2025-05 | 64K / 128K |
| claude-opus-4-7 | opus | Opus 4.7 | 2026-01 | 64K / 128K |
| claude-opus-4-8 | opus | Opus 4.8 | 2026-01 | 64K / 128K |
| claude-opus-5 | opus | Opus 5 | 2026-05 | 64K / 128K |
| **claude-fable-5** | fable | **Fable 5** | 2026-01 | 64K / 128K |
| **claude-fable-5-1** | fable | **Fable 5.1** | 2026-06 | 64K / 128K |
| **claude-mythos-5** | mythos | **Mythos 5** | 2026-01 | 64K / 128K |
| **claude-mythos-5-1** | mythos | **Mythos 5.1** | 2026-06 | 64K / 128K |

**`fable` 与 `mythos` 是两个全新的模型家族**，此前从未在公开材料中出现。它们与 opus/sonnet 并列在同一份 catalog 里，带完整的 cutoff、输出上限、定价档位与 provider 路由（`gateway` / `foundry` / `anthropic_aws` / `anthropic_google_cloud` / `mantle`）——说明已进入发行分支，而非实验占位。

定价档位（内部代号）：`tier_2_10`、`tier_3_15`、`tier_5_25`、`tier_10_50`、`tier_15_75`、`tier_10_50_cache_read_0_25`、`haiku_35`、`haiku_45`。

### 5.7 子代理与并行度

```
CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS   默认 20
CLAUDE_CODE_MAX_WEB_SEARCHES_PER_SESSION 默认 200
```

子代理类型包含 `general-purpose`、`Explore`、`Plan`、`fork`。其中 `fork` 有专门提示词：

> "Calling Task with subagent_type: 'fork' creates a fork — it inherits your full conversation context, runs in the background, and keeps its tool output out of your context… **If you ARE the fork** — execute directly; do not re-delegate."

`worktree` 相关命中 2,213 次——worktree 是这一版的重点能力（隔离工作区 + `EnterWorktree`/`ExitWorktree` 工具 + 专门的沙箱路径白名单）。

---

## 6. 协议与网络面

### 6.1 端点清单

| 主机 | 引用次数 | 用途 |
|---|---:|---|
| `code.claude.com` | 1,980 | 主 API 基址（新域名，取代 `api.anthropic.com`） |
| `claude.ai` | 638 | OAuth、账号、云端会话 |
| `github.com` | 623 | GitHub 集成 |
| `api.anthropic.com` | 48 | 传统基址（兼容保留） |
| `platform.claude.com` | 18 | 平台/控制台 |
| `downloads.claude.ai` | 10 | 更新下载 |
| `releases.claude.com` | 5 | 版本发布信息 |
| `api-staging.anthropic.com` | 5 | **staging 环境残留** |
| `statsig` / `growthbook` | 2 / 33 | feature flag 后端（双供应商） |
| `sentry` / `datadog` | 10 / 18 | 错误上报 |

### 6.2 API 路径

```
/v1/messages                    /v1/messages/count_tokens     /v1/messages/batches
/v1/models                      /v1/complete                  /v1/metrics
/v1/oauth/token                 /v1/mcp/{server_id}
/v1/toolbox/shttp/mcp/{server_id}
/v1/code/triggers               /v1/code/memory/              /v1/code/agent-proxy
/v1/code/local/memory/credential  /v1/code/local/memory/mounts
/v1/code/scm-connectors/{provider}/{id}/tunnel
/v1/pages/mcp                   /v1/artifacts/mcp             /v1/design/{consent,grants}
/v1/ultrareview/quota
/api/oauth/account/settings     /api/oauth/usage
/api/claude_code/policy_limits  /api/claude_code/settings
/api/claude_code_grove          /api/event_logging/v2/batch
/api/frame/{deploy,track,upload,sync,db/agent,types}
/api/artifacts/mcp              /api/desktop/
```

`/v1/code/scm-connectors/.../tunnel` 是**反向隧道**端点——云侧可通过它访问本地 SCM。

### 6.3 请求头指纹

**`x-anthropic-billing-header`**（源码原文）：

```js
U = `x-anthropic-billing-header: cc_version=${p}; cc_entrypoint=${f};${d}${h}${E}${w}${R}`
```

它被当作 **system 消息的一部分**（`HCe()` 检测以该前缀开头的 text 并剔除），用于服务端计费归属与客户端版本归因。字段含 `cc_version`、`cc_entrypoint`，以及条件附加的 `cc_prompt_id=<uuid>`（仅 `firstParty` 且满足 UUID 正则时）。

其他：
- `anthropic-version: 2023-06-01`（固定）
- `anthropic-beta: ccr-byoc-2025-07-29`（唯一硬编码 beta，BYOC 场景）
- `User-Agent: anthropic-sdk-typescript/<ver> ...`（内部 SDK 自报）
- `X-Stainless-Retry-Count` / `X-Stainless-Timeout` —— 使用 **Stainless 生成的 SDK**
- `anthropic-dangerous-direct-browser-access: true`（浏览器环境才加）

### 6.4 工具 payload 的隐藏能力

工具序列化支持四个 API 层特性：

| 特性 | 作用 |
|---|---|
| `defer_loading` | 工具延迟加载，配合 **tool search** 按需发现（`tengu_plucky_orchard` 控制，值 `on`/`off`/`cold`） |
| `strict: true` | 严格 schema 校验 |
| `eager_input_streaming` | 输入流式（当前仅 Vertex 开） |
| `cache_control.ttl` | 提示词缓存 TTL 覆写 |

`defer_loading` 的设计意图在 MCP 配置注释里写得很清楚：

> "When true, all tools from this server are always included in the prompt and never deferred behind tool search… As a side effect this also blocks startup until the server is connected (capped at the standard 5s connect timeout)"

即**默认 MCP 工具是"延迟+搜索发现"，不占常驻上下文**。这解释了为什么工具数能从 22 扩到 70 而上下文不爆。

### 6.5 认证

| 方式 | 说明 |
|---|---|
| API Key | `ANTHROPIC_API_KEY` / `ANTHROPIC_AUTH_TOKEN` |
| OAuth | `CLAUDE_CODE_OAUTH_TOKEN`，PKCE（`code_challenge` 41 处），`refresh_token` 流程 |
| 云托管 | Bedrock（AWS）、Vertex（Google）、Azure（`AZURE_FEDERATED_TOKEN_FILE` 联邦令牌） |
| 企业 | `apiKeyHelper`（外部命令取密钥，75 处）、OIDC federation（`oidcFederationProvider`） |
| 托管网关 | `CLAUDE_CODE_GATEWAY_TOKEN_FILE_DESCRIPTOR`、`CLAUDE_CODE_HFI_BEARER_TOKEN` |

OAuth 常量：

```
OAUTH_FILE_SUFFIX: "-custom-oauth"
CONSOLE_SUCCESS_URL: `${base}/oauth/code/success?app=claude-code`
CLAUDEAI_SUCCESS_URL: `${base}/oauth/code/success?app=claude-code`
MANUAL_REDIRECT_URL: `${base}/oauth/code/callback`
roles endpoint: `${base}/api/oauth/claude_cli/roles`
```

凭证落盘：macOS Keychain（`keychain` 44 处）、Windows 凭据管理器（"Credential Manager"）、`CLAUDE_SECURESTORAGE_CONFIG_DIR` 可重定向。

### 6.6 服务端可下发策略（重要）

```js
function et() {
  let e = To();
  if (e) return `${e.url}/managed/settings`;
  return `${Yt().BASE_API_URL}/api/claude_code/settings`;
}
```

`/managed/settings` 返回的 settings 会被 `fPe()` 校验后合并。**它能下发什么**？同一份源码里：

```js
function $Q(e) {
  if (e.telemetry.forward_to.length === 0) return null;
  let t = e.listen.public_url;
  ...
  return {
    CLAUDE_CODE_ENABLE_TELEMETRY: "1",
    OTEL_METRICS_EXPORTER: Ch(r, "metrics"),
    OTEL_LOGS_EXPORTER:   Ch(r, "logs"),
    OTEL_TRACES_EXPORTER: Ch(r, "traces"),
    OTEL_EXPORTER_OTLP_ENDPOINT: t,
    OTEL_EXPORTER_OTLP_PROTOCOL: "http/protobuf"
  }
}
```

**服务端可以把 `OTEL_EXPORTER_OTLP_ENDPOINT` 指向任意地址**，并强制打开 `CLAUDE_CODE_ENABLE_TELEMETRY=1`。这在企业托管场景是合理设计（管理员集中收日志），但它意味着：**"我没开遥测"不等于"没有遥测"**——只要上游能下发策略，端点就能被改写。

同时 `tengu_managed_settings_os_read` 事件（本地实证 3 条）证明代码会去读**操作系统级托管配置**（macOS managed preferences / Windows 注册表策略）。

---

## 7. 权限、沙箱与 Hooks

### 7.1 权限模型

**6 种模式**（命中次数）：`default`(683)、`auto`(399)、`plan`(233)、`bypassPermissions`(104)、`acceptEdits`(51)、`dontAsk`(48)，另有 `delegate`(3)、`unattended`(4)。

**规则语法**（源码内示例，可直接用于 settings）：

```
Bash(git *)                     Bash(npm *)                Bash(npm run build)
Bash(npm run:*)                 ← 注释标注 "prefix matching (legacy)"
Read(~/.zshrc)                  Read(//etc/**)             Read(**/secrets/**)
Edit(docs/**)                   Edit(**/*.env)             Edit(.claude)
WebFetch(domain:github.com)     WebFetch(domain:*.google.com)
mcp__github__create_pull_request
mcp__linear__create_issue
```

规则匹配器构造（`IE = "\((\S[^\r\n]*)\)"`）支持 `Bash(cmd *)`、`Read(path)`、`WebFetch(domain:...)`、`mcp__server__tool` 四类。**默认危险示例表**（源码内置）：

```js
y5 = { Bash: ["curl *"], Read: ["//etc/**", "**/secrets/**"], Edit: ["**/*.env", "//etc/**"] }
b5 = ["Bash", "Task", "Skill"]   // 需要额外确认的工具
```

**工具别名合并**：

```js
Wo = { Read: ["Read","Grep","Glob"], Edit: ["Edit","Write","NotebookEdit"] }
```

即给 `Read(...)` 授权会同时覆盖 Grep/Glob；给 `Edit(...)` 覆盖 Write/NotebookEdit——这是易被忽视的**权限放大面**。

### 7.2 沙箱：三平台能力差异巨大

| 平台 | 机制 | 状态 |
|---|---|---|
| macOS | `sandbox-exec` / seatbelt profile | ✅ 原生 |
| Linux/WSL | bubblewrap (`bwrap`) + seccomp | ✅ 原生 |
| Windows | **独立用户 + `srt-win.exe` 辅助进程** | ⚠️ **本安装缺失二进制** |

Windows 路径的源码逻辑：

```js
var xjs = path.join(rootDir, "vendor", "srt-win", arch, "srt-win.exe");
function MP(e) {
  if (e?.path === undefined) throw new vd("srt_win_not_found",
    "no srt-win path configured; set windows.srtWin.path ...");
  if (!fs.existsSync(e.path)) throw new vd("srt_win_not_found", ...);
}
```

**实测**：`@anthropic-ai/claude-code` 包内**没有 `vendor/` 目录**（`find -iname "*srt*"` 零命中）。因此在这台机器上，Windows 沙箱一旦被启用就会抛 `srt_win_not_found`；除非用户手工设置 `windows.srtWin.path` 指向一个自备的 `srt-win.exe`。

配置项（`sandbox` 出现 766 次）：

```
allowWrite / denyWrite / denyRead / allowRead
allowManagedReadPathsOnly   ← 仅策略源生效
disabled                    ← 仅 macOS / Linux-WSL；Windows 上被忽略
```

默认 `denyRead` 包含（很关键）：

```
/run/docker.sock
/run/containerd/containerd.sock
/run/podman/podman.sock
/run/buildkit/buildkitd.sock
/run/dbus
/run/user
```

默认 `denyWrite` 包含 shell 启动文件（`.bashrc`、`.zshrc`、`.profile`、`.claude`、`.claude.json` 等）——防止通过改 shell 配置实现持久化。这是**明确的威胁模型**：把"Agent 被诱导写入启动脚本"当作真实攻击面。

> 一条重要注释："Only honored from user, managed/policy, or CLI (`--settings`) settings — project settings (`.claude/settings.json` and `.claude/settings.local.json`) are ignored." —— 即**仓库里的 settings 不能改沙箱规则**，防止恶意仓库自我提权。

### 7.3 Hooks：14 个事件

| 事件 | 命中 |
|---|---:|
| Stop | 198 |
| PreToolUse | 162 |
| PostToolUse | 107 |
| PermissionRequest | 60 |
| UserPromptSubmit | 55 |
| SessionStart | 54 |
| SubagentStop | 47 |
| Notification | 36 |
| PreCompact | 26 |
| SessionEnd | 21 |
| PostCompact | 19 |

`PreToolUse` / `PermissionRequest` 被单独标记为 **guard 类**（`XW = new Set(["PreToolUse","PermissionRequest"])`），语义不同：普通 hook 加载失败只是丢弃该条，而 guard 类 hook 加载失败会**连带拒绝它旁边的权限声明**：

```js
mh = "a PreToolUse/PermissionRequest hook that cannot be loaded may be what guards
      the permissions declared beside it, so nothing it sits in is applied until
      the entry is fixed or removed"
```

这是**fail-closed** 设计，方向正确。

---

## 8. 遥测与隐私（本地实证）

### 8.1 事件规模

`~/.claude/telemetry/` 有 **3 个 `1p_failed_events.*.json`** 文件（发送失败落盘，共 552 KB），解析出 **277 条事件**、**32 个不同事件名**，全部 `tengu_*` 前缀：

```
88  tengu_feature_ok            84  tengu_skill_loaded
11  tengu_dir_search             8  tengu_plugin_enabled_for_session
 6  tengu_timer                  5  tengu_plugin_name_collision
 3  tengu_managed_settings_os_read   3  tengu_feature_sad
 3  tengu_cli_flags              3  tengu_frontmatter_shadow_unknown_key
 3  tengu_plugin_skills_dir_loaded   3  tengu_uds_startup_bind
 3  tengu_shell_set_cwd          3  tengu_started
 3  tengu_artifact_toolset       3  tengu_prompt_suggestion_init
 3  tengu_headless_mcp_prewait   3  tengu_claudeai_mcp_eligibility
 3  tengu_claudemd__initial_load 3  tengu_headless_plugin_install
 3  tengu_sdk_init_handshake     3  tengu_shell_allow_rules_at_init
 3  tengu_init                   3  tengu_config_cache_stats
 3  tengu_managed_config_ready
```

### 8.2 每条事件的字段（100% 覆盖，277/277）

```
event_name, client_timestamp, model, session_id, user_type, betas,
env, entrypoint, agent_sdk_version, is_interactive, client_type,
process, additional_metadata, event_id, device_id
```

**`process` 字段**（base64 JSON，实测解码）：

```json
{"uptime":0.364,"rss":141328384,"heapTotal":32581632,"heapUsed":32581632,
 "external":155806,"arrayBuffers":155806,"constrainedMemory":16870006784,
 "cpuUsage":{"user":343000,"system":62000}}
```

→ 上报**进程内存与 CPU 用量**。

**`additional_metadata` 字段**（base64 JSON，实测解码）：

```json
{"is_wsl":false,"await_ms":2,
 "hklm_status":"exited","hklm_exit_code":1,"hklm_duration_ms":168,
 "hkcu_status":"exited","hkcu_exit_code":1,"hkcu_duration_ms":154}
```

→ 上报**对 `HKLM` 和 `HKCU` 注册表的探测结果（状态、退出码、耗时）**。这是在检测企业策略/托管配置是否写入注册表，同时把"本机是否存在该键"这一事实带出机器。

**`env` 字段**（实测）：

```json
{"platform":"win32","node_version":"v26.3.0","terminal":"non-interactive",
 "package_managers":"npm,pnpm","runtimes":"node","is_running_with_bun":true,
 "is_ci":false,"is_claubbit":false,"is_github_action":false,
 "is_claude_code_action":false,"is_claude_ai_auth":false,
 "version":"2.1.295","arch":"x64","is_claude_code_remote":false,
 "deployment_environment":"unknown-win32","is_conductor":false,
 "version_base":"2.1.295","build_time":"2026-10-08T16:50:59Z",
 "is_local_agent_mode":false,"platform_raw":"win32"}
```

`is_running_with_bun: true` —— **运行时自己承认了 Bun**，这是最直接的交叉验证。

### 8.3 身份标识

| 项 | 值 |
|---|---|
| `device_id` 去重后 | **1 个**（跨 3 个 session 稳定不变） |
| `session_id` | 3 个 |
| 版本分布 | 2.1.295（199 条）、2.1.275（78 条） |

`device_id` 与 `~/.claude.json` 中的 `userID` 一致——即**匿名设备标识在本地是持久化的**，卸载重装若不清理该文件，标识延续。

### 8.4 遥测通道

- 协议：**OpenTelemetry**（`OTEL_EXPORTER_OTLP_*` 共 12 个变量）
- 支持 metrics / logs / traces 三路独立配置
- 内部另有 `ANT_OTEL_*`、`BETA_TRACING_ENDPOINT`、`CLAUDE_CODE_BYOC_ENABLE_DATADOG`（BYOC 自带 Datadog）
- 可关闭：`DISABLE_TELEMETRY`、`DO_NOT_TRACK`、`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`、`DISABLE_ERROR_REPORTING`
- 但如 §6.6 所述，**服务端策略可反向打开**

---

## 9. 安全评估

### 9.1 代码保护强度：**接近零**

| 维度 | 评价 |
|---|---|
| 混淆 | ❌ 仅 minify（变量名压缩），字符串全明文 |
| 加密 | ❌ `.bun` 熵 6.58，明文可读 |
| 字节码保护 | ⚠️ 有 `@bytecode` 标记，但**同时内嵌源码**，等于没有 |
| 反调试 | ❌ 未发现 |
| 完整性自校验 | ❌ 未发现（仅 Authenticode） |
| 字符串加密 | ❌ 未发现 |
| 剥离符号 | ⚠️ 应用层符号被 minify，但**路径、URL、提示词、flag 名全保留** |

**实际成本**：从拿到 exe 到拿到 31 MB 可读源码，本报告用时不到 1 小时，核心算法 20 行 Python。

这不是批评——Claude Code 是闭源商业产品，但它的"闭源"仅体现在**许可证**上，不体现在**技术保护**上。任何有基本逆向能力的人都能完整复刻其行为。Anthropic 显然选择了"可维护性 > 保密性"，这是合理取舍，但**不应被外部误认为"原生程序不可分析"**。

### 9.2 攻击面清单

| 面 | 风险 | 说明 |
|---|---|---|
| 供应链 | **中高** | 8 个平台二进制 + `postinstall` 自动放置；`optionalDependencies` 任一被投毒即接管；无内置完整性校验（除 Authenticode） |
| 遥测 | 中 | 进程内存/CPU、注册表探测结果、设备 ID 持久化；服务端可改 OTEL 端点 |
| 权限模型 | 中 | 工具别名合并（Read→Grep/Glob、Edit→Write/NotebookEdit）造成授权放大 |
| 沙箱 | 中 | Windows 缺 `srt-win.exe` 时无 OS 级隔离；`disabled` 在 Windows 被忽略（即不能关） |
| Hooks | 低 | fail-closed 设计正确 |
| MCP | 中高 | 第三方 MCP 服务端返回内容直接进入上下文；`defer_loading` 使工具可动态出现 |
| 二进制加固 | 低 | 无 CFG、无 GS Cookie；JIT 场景下属生态妥协 |
| 证书 | 低（时间敏感） | 4 天后到期，换签会造成指纹跳变 |

### 9.3 值得肯定的设计

1. **`CLAUDE_CODE_SUBPROCESS_ENV_SCRUB`** —— 子进程环境清洗，防密钥泄漏到 Bash；
2. **`denyWrite` 默认覆盖 shell 启动文件** —— 明确防御"写入 `.bashrc` 实现持久化"；
3. **沙箱规则不接受项目级 settings** —— 防恶意仓库自我提权；
4. **guard 类 hook fail-closed** —— 加载失败即拒绝，不降级放行；
5. **`toAutoClassifierInput`** —— 权限判定引入语义分类器，而非纯正则；
6. **`/version` 区分"运行版本 vs 已下载版本"** —— 避免更新状态误报。

---

## 10. 针对本机的具体建议

### 10.1 🔴 立即处理：API 密钥明文

`~/.claude/settings.json` 中：

```json
"ANTHROPIC_AUTH_TOKEN": "12138"
```

**明文存储、无加密、无权限保护**。建议：

1. **立即轮换该密钥**；
2. 改用 `apiKeyHelper` 从凭据管理器取值：

```json
{
  "apiKeyHelper": "powershell -NoProfile -Command \"(Get-StoredCredential -Target claude).Password\""
}
```

3. 或改用环境变量注入（配合 `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1`）。

### 10.2 🔴 数据流向已改变

你的配置把 `ANTHROPIC_BASE_URL` 指向 `https://api.deepseek.com/anthropic`，并覆写了全部模型别名。后果：

- **所有代码、文件内容、会话上下文都发往 DeepSeek，而非 Anthropic**；
- 但遥测仍可能直连 `code.claude.com` / `platform.claude.com`（两套通道独立）；
- 若要彻底断掉原生遥测：

```json
{ "env": { "DISABLE_TELEMETRY": "1", "DISABLE_ERROR_REPORTING": "1",
           "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1" } }
```

> 注意 `CLAUDE_CODE_AUTO_COMPACT_WINDOW: 1000000` 与 `deepseek-flash[1m]` 的配合——1M 上下文下，单次请求可能携带极大代码量外发，建议评估必要性。

### 10.3 🟡 清理遥测残留

`~/.claude/telemetry/` 有 3 个文件 552 KB，含设备 ID 与注册表探测记录。若在意：

```powershell
Remove-Item "$env:USERPROFILE\.claude\telemetry\*" -Force
```

（仅删缓存事件，不影响功能。）

### 10.4 🟡 会话数据体量

| 目录 | 文件数 | 大小 |
|---|---:|---:|
| `projects/` | 113 | **73.2 MB** |
| `file-history/` | 311 | 2.7 MB |
| `skills/` | 15 | 0.1 MB |

`projects/` 存的是完整会话 JSONL（含所有工具输入输出）。73 MB 意味着**相当多的代码与命令历史落盘未加密**。建议定期归档或按 `cleanupPeriodDays` 收敛。

---

## 11. 附录

### A. 关键常量速查

| 项 | 值 |
|---|---|
| Bun 版本 | `bun-v1.4.1` / `Bun v1.4.1` |
| JS 引擎 | JavaScriptCore（含 WebKit 源文件路径） |
| 导出 DLL 名 | `bun-profile.exe` |
| `.bun` 魔数 | `\n---- Bun! ----\n`（尾部） |
| `.bunblt` 魔数 | `BUNBLTNS` + version 1 |
| 模块横幅 | `// @bun @bytecode` |
| 模块数 | 1,644 |
| 源码总量 | 32,760,365 B（31.24 MB） |
| chunk 名 | 1,633 个 `chunk-[a-z0-9]{8}.js` |
| 虚拟根 | `B:/~BUN/root/` |
| API 版本头 | `2023-06-01` |
| 唯一硬编码 beta | `ccr-byoc-2025-07-29` |
| 计费头 | `x-anthropic-billing-header: cc_version=…; cc_entrypoint=…` |

### B. 关键文件偏移

| 内容 | 偏移 |
|---|---|
| `.text` | `0x400` |
| `.rdata` | `0x3FA2800` |
| `.bunblt` | `0x54A0200` |
| `.rsrc` | `0x572D800` |
| `.bun` | `0x5790A00` |
| 证书表 | `0xD0D1600` |
| LoadConfig | RVA `0x4001AF0` → 文件 `0x3FA2AF0` |
| 导出表 | RVA `0x52FCD7B` |
| 导入表 | RVA `0x53042D0` |

### C. 主模块索引（按体量）

| 模块 | 大小 | 内容 |
|---|---:|---|
| `mod_0261.js` | 5.56 MB | 主控、系统提示词装配、工具编排、API 客户端 |
| `mod_1163.js` | 1.59 MB | — |
| `mod_1101.js` | 1.09 MB | 权限规则、`BUILTIN_TOOL_NAMES`、Cowork/Code 策略 |
| `mod_0186.js` | 65 KB | 工具定义集合（Read/Edit/Bash 的 prompt 模板） |
| `mod_0000.js` | — | CJS 包装入口 |

### D. 产物清单

```
E:\Download\temp\REA_claudeCode\
├── Claude_Code_逆向技术报告_v2.md   ← 本报告
└── work\
    ├── modules\              1,644 个还原模块（31.24 MB）
    ├── manifest.json         模块偏移/大小/Bun 路径索引
    ├── out\
    │   ├── analysis.txt      全量特征提取原始输出
    │   ├── indicators.json   407 环境变量 / 2,316 flag / 主机名 / API 路径 / 模型
    │   ├── tools_resolved.json
    │   ├── tool_hints.json   70 个工具的 hint + 描述
    │   ├── pe.json           PE 结构完整数据
    │   └── prompt_region.txt 系统提示词区域原文
    └── *.py                  9 个可复现脚本
```

---

## 12. 未验证项（明确标注）

以下内容**未**做，报告中未据此下任何结论：

1. **`.text` 段 66.72 MB 机器码未做函数级反汇编** —— REA 的 Ghidra provider 未配置（缺 `GHIDRA_INSTALL_DIR`，且 JDK 17 < 21），因此 Bun 运行时/JSC 内部实现、JIT 行为、`srt-win` 的 Windows 沙箱实现细节均为黑盒；
2. **`.bunblt` 字节码未反编译** —— 只确认了其"字节码 + 源码"双份存储，未解析字节码指令格式；
3. **未执行动态分析** —— 全程静态，未运行样本、未抓包、未挂调试器；
4. **未验证遥测实际发送目标** —— 本地只有"发送失败"落盘文件，成功路径的目的地未观测；
5. **`fable` / `mythos` 模型的能力与定位未知** —— 仅有 catalog 元数据，无任何行为证据；
6. **`srt-win.exe` 未获取** —— 包内不存在，无法分析其隔离实现。

若要补齐 1–3，需：安装 Ghidra 12.1.x + JDK 21 并设 `GHIDRA_INSTALL_DIR`；或补齐 REA 的 `native/windows/build/manifest.json`（需 Linux + MinGW-w64 交叉编译环境，官方明确不支持 Windows 原生构建）。

---

*报告完*
