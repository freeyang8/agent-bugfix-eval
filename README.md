# Ansible 18-Bug 修复评测 · 会话记录与逆向报告

本目录收录「AI 编码 Agent 能力评测」中 **Ansible 18 个 bug 修复任务** 的完整会话记录，以及两个被测 Agent 的逆向工程技术报告。全部文件已脱敏。

- 评测日期：2026-10-18

- 被测 harness：Claude Code / ChatGPT 桌面版（Codex）

- 任务：在沙箱内完成 Ansible 的 18 个 bug 修复

***

## 一、目录内容

| 文件                                                                       | 类型           | 规模               | 说明                                                                   |
| ------------------------------------------------------------------------ | ------------ | ---------------- | -------------------------------------------------------------------- |
| `e3880c50-e16b-4ff4-bdee-13784514db68.jsonl`                             | 会话记录（JSONL）  | 1,819 行 / 3.5 MB | Claude Code 侧完整会话原文，模型标识 `claude-sonnet`                             |
| `rollout-2026-10-18T13-21-31-01a14d75-3f34-7403-b147-764efa7ffe85.jsonl` | 会话记录（JSONL）  | 2,927 行 / 4.9 MB | Codex 桌面版侧会话记录，多 Agent 协作（`/root` + 子 Agent）模式，模型标识 `deepseek-flash` |
| `ChatGPT_Codex_逆向技术报告_v3.md`                                             | 报告（Markdown） | 2,607 行          | Codex 桌面版逆向工程技术报告 v3                                                 |
| `Claude_Code_逆向技术报告_v2.md`                                               | 报告（Markdown） | 1,063 行          | Claude Code 2.1.266 逆向工程技术报告 v2                                      |

***

## 二、会话记录

两份 JSONL 均为逐行合法 JSON，每行一条会话事件：

- **Claude Code 侧**：字段以 `type` / `uuid` / `timestamp` / `message` / `cwd` / `sessionId` 为主，`queue-operation`、`user`、`assistant`、工具调用等事件按时间顺序排列。

- **Codex 侧**：字段以 `timestamp` / `ordinal` / `type` / `payload` 为主，`session_meta`、`event_msg`、`response_item` 等事件按 `ordinal` 递增排列。

两份记录都包含完整的用户提示、Agent 推理/工具调用、命令输出与最终修复过程，可用于复盘 Agent 在长任务中的规划、试错与收敛行为。

***

## 三、逆向技术报告

### 1. `ChatGPT_Codex_逆向技术报告_v3.md`

- 分析对象：本机安装的 ChatGPT 桌面应用（MSIX 包身份 `OpenAI.Codex`，显示名 `ChatGPT`，内核为 Rust 编写的 `codex.exe`）。

- 章节结构：§0 执行摘要 → §3 MSIX 包结构 → §4 五层异构栈 → §5 进程与执行模型 → §6 IPC 与 RPC 协议面 → §7 渲染层安全 → §8 内核能力面 → §9 提示词与上下文工程 → §10 模型与供应商配置 → §11 数据落盘与隐私 → §12 供应链与完整性 → §13 攻击面与风险评级 → §14 加固建议 → §15 未验证项与局限 → §16 附录 → §17 结语。

- 证据规范：结论标注来源（REA Evidence ID / 文件路径 / 字节偏移 / 命令输出），采用 L1–L4 证据分级；未验证项集中列出。

### 2. `Claude_Code_逆向技术报告_v2.md`

- 分析对象：`claude.exe`（Claude Code 2.1.266），PE32+ / x86-64，由 Bun 编译为单文件可执行。

- 章节结构：§0 结论速览 → §1 分析方法与 REA 工具链实测 → §2 供应链解剖 → §3 PE 结构解剖 → §4 Bun 打包格式解剖 → §5 应用层结构 → §6 协议与网络面 → §7 权限、沙箱与 Hooks → §8 遥测与隐私 → §9 安全评估 → §10 针对本机的具体建议 → §11 附录 → §12 未验证项。

***

## 四、脱敏说明

- 个人身份、账户名、主机名、私有目录路径、硬件信息与凭证类内容统一替换为 `12138`。

- 保持原始行数与 JSON 结构不变；两份 JSONL 逐行 `json.loads` 校验通过（失败行数 0）。

- 公开产品名、公开 URL、通用系统路径（如 `C:\Program Files`）不属于个人敏感信息，予以保留。

***

## 五、读取方式

```bash
# 逐行解析 JSONL
python - <<'PY'
import json
with open("e3880c50-e16b-4ff4-bdee-13784514db68.jsonl", encoding="utf-8") as f:
    for line in f:
        event = json.loads(line)
        print(event.get("type"), event.get("timestamp"))
PY
```

Markdown 报告可直接用任意编辑器或 Markdown 阅读器打开。

***

***

## 七、分支说明

本目录提交在独立分支 `ansible-18bugs-2026-10-25` 上，`main` 分支保留此前的交付物（PySnooper 题面/答案、旧版报告与会话记录），两者互不覆盖，均可单独下载。
