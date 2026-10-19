# AI 编码 Agent 能力评测 · 交付物说明（README）

本目录是「AI 编码 Agent 能力评测」的完整交付物：两个编码 Agent（Claude Code / ChatGPT 桌面版 Codex）的**逆向工程技术报告**、两者在同一修复任务（PySnooper 三个 bug）上的**完整会话原文**，以及评测所用的**题面集**与**官方答案集**。

---

## 一、目录内容一览

| 路径 | 类型 | 说明 |
|---|---|---|
| `ChatGPT_Codex_逆向技术报告_v3.md` | 报告（Markdown） | Codex 桌面版逆向工程技术报告 v3，约 9.9 万字符，§0–§17 共 18 章 |
| `Claude_Code_逆向技术报告_v2.md` | 报告（Markdown） | Claude Code 2.1.266 逆向工程技术报告 v2，§0–§12 共 13 章 |
| `PySnooper-三个bug-Claude会话原文.jsonl` | 会话记录（JSONL） | Claude 修复 PySnooper 三个 bug 的完整会话原文，563 行 |
| `PySnooper-三个bug-Codex会话原文.jsonl` | 会话记录（JSONL） | Codex 修复同一任务的完整会话原文，993 行 |
| `PySnooper/` | 题面集 | PySnooper 3 个 bug 的题面（**已剥离答案**） |
| `PySnooper_answer/` | 答案集 | PySnooper 3 个 bug 的官方修复补丁（`bug_patch.txt`），**严禁暴露给被测 agent** |
| `ansible/` | 题面集 | ansible 18 个 bug 的题面（**已剥离答案**） |
| `ansible_answer/` | 答案集 | ansible 18 个 bug 的官方修复补丁（`bug_patch.txt`），**严禁暴露给被测 agent** |

---

## 二、交付物一：两份逆向工程技术报告

### 1. `ChatGPT_Codex_逆向技术报告_v3.md`

- **分析对象**：本机安装的 ChatGPT 桌面应用（MSIX 包身份 `OpenAI.Codex`，显示名 `ChatGPT`，内核为 Rust 编写的 `codex.exe`）。
- **章节结构**：§0 执行摘要 → §3 MSIX 包结构 → §4 五层异构栈（MSIX 外壳 / owl 运行时 / `app.asar` / Rust 内核 / 特权辅助进程）→ §5 进程与执行模型 → §6 IPC 与 RPC 协议面 → §7 渲染层安全 → §8 内核能力面 → §9 提示词与上下文工程 → §10 模型与供应商配置 → §11 数据落盘与隐私 → §12 供应链与完整性 → §13 攻击面与风险评级 → §14 加固建议 → §15 未验证项与局限 → §16 附录 → §17 结语。
- **证据规范**：每条结论标注来源（REA Evidence ID / 文件路径 / 字节偏移 / 命令输出），采用 L1–L4 四层证据分级（结构化证据 / 制品证据 / 字符串面证据 / 行为观测）；未验证项集中在 §15 显式列出。

### 2. `Claude_Code_逆向技术报告_v2.md`

- **分析对象**：`claude.exe`（Claude Code 2.1.266），218,971,808 字节，PE32+ / x86-64，由 Bun v1.4.1 编译为单文件可执行。
- **章节结构**：§0 结论速览 → §1 分析方法与 REA 工具链实测 → §2 供应链解剖 → §3 PE 结构解剖 → §4 Bun 打包格式解剖 → §5 应用层结构（提示词分块 / 22 个内置工具 / 138 个斜杠命令 / 2,316 个 Feature Flag / 407 个环境变量 / 模型目录）→ §6 协议与网络面 → §7 权限、沙箱与 Hooks → §8 遥测与隐私 → §9 安全评估 → §10 针对本机的具体建议 → §11 附录 → §12 未验证项。

---

## 三、交付物二：两份会话原文（JSONL）

| 文件 | 行数 | 内容 |
|---|---|---|
| `PySnooper-三个bug-Claude会话原文.jsonl` | 563 | Claude 在 PySnooper 三个 bug 上的完整执行过程 |
| `PySnooper-三个bug-Codex会话原文.jsonl` | 993 | Codex 在同一任务上的完整执行过程 |

- **格式**：JSONL，每行一条会话事件（`type` / `operation` / `timestamp` / `sessionId`，部分行含 `message`、`cwd`、`gitBranch` 等字段），可用任意 JSON 行读取工具逐行解析。
- **评测条件**：同一模型（`deepseek-flash`）分别置于两个 harness 下，执行同一份题面（PySnooper 三个 bug）。
- **结果**：两个 Agent 最终修复结论一致，均为 **3/3 全部测试用例通过**。

---

## 四、评测输入与答案隔离

- **题面集**（`PySnooper/`、`ansible/`）：只包含「是什么 bug + 如何获取源码 + 如何复现 + 复现测试文件」，不含任何修复答案；`fixed_commit_id`、官方补丁 `bug_patch.txt` 已全部移出。各目录内的 `README.md` 给出获取源码（**必须不带 git 历史**）与复现步骤、防泄题清单。
- **答案集**（`PySnooper_answer/`、`ansible_answer/`）：含官方修复补丁 `bug_patch.txt` 与 `fixed_commit_id`，仅供评测/核对使用，**严禁暴露给被测 agent**。

---

## 五、敏感信息更替

> **声明**：本目录内全部交付物在对外交付前已完成敏感信息检查，敏感信息一律替换为占位值 **`12138`**。

---

## 六、文件格式与读取方式

- `.md`：UTF-8 纯文本，任意 Markdown 阅读器/编辑器可直接打开。
- `.jsonl`：UTF-8，逐行合法 JSON（每行可独立 `json.loads` 解析）；可直接文本检索，或用脚本按键过滤。
