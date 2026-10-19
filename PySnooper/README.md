# PySnooper —— Bug 修复任务（题面，不含答案）

- 上游仓库: https://github.com/cool-RR/PySnooper
- 语言/Python 版本: 见各 bug 的 bug.info
- 共 3 个 bug

本目录只给"是什么 bug + 如何获取源码 + 如何复现 + 复现测试文件"，
**不含**任何修复答案（官方补丁 bug_patch.txt、fixed_commit_id 已全部移出，见 `PySnooper_answer`）。

## 目录结构
```
bugs/<id>/
  bug.info            定位信息: python_version / buggy_commit_id / test_file（已移除 fixed_commit_id）
  run_test.sh         复现命令（在项目根目录执行）
  setup.sh            安装/编译命令
  requirements.txt    依赖
  <test_file 文件名>   复现测试文件（取自 fixed commit，会覆盖到源码对应路径）
```

## 获取源码（buggy 版本，务必不带 git 历史）

> ⚠️ 关键：**不要**让被测 agent 通过 git 历史反推出修复提交。以下方式都不会把修复历史交到 agent 手里。

方式一（推荐）：下载指定 commit 的源码压缩包（无 `.git`、无历史）
```
curl -L -o PySnooper.zip "https://codeload.github.com/cool-RR/PySnooper/zip/<bug.info 中的 buggy_commit_id>"
unzip PySnooper.zip
```
解压得到 `PySnooper-<commit>/`，即该 buggy commit 的完整源码，压缩包内不含任何 git 历史或修复提交。

方式二：只拉取该 commit 的浅历史
```
git init PySnooper && cd PySnooper
git remote add origin https://github.com/cool-RR/PySnooper.git
git fetch --depth 1 origin <buggy_commit_id>
git checkout FETCH_HEAD
```

方式三：完整 clone 后立即清除历史
```
git clone https://github.com/cool-RR/PySnooper.git
cd PySnooper
git checkout <buggy_commit_id>
rm -rf .git
```
`rm -rf .git` 必须执行——否则克隆下来的历史里就含修复提交。

> BugsInPy 的 `bugsinpy-checkout` 已刻意不再提供：它会把 `fixed_commit_id`、改动文件清单复制进工作树，且指向本地 `projects/` 目录，等价于直接把答案交给 agent。

## 复现步骤
1. 按上面任一方式取得 buggy 版本源码。
2. 执行 `setup.sh`（装依赖 / 编译）。
3. 把本目录 `bugs/<id>/` 下的复现测试文件复制到源码的 `test_file` 路径。
4. 在项目根目录执行 `run_test.sh` 里的命令 —— 输出即为该 bug 的报错信息（traceback / 断言失败）。

## 防泄题清单（给评测操作者）

- **只把本目录（题面）挂载进 agent 沙箱**：不要挂载 `实验环境` 根目录或 `BugsInPy/`——同级的 `PySnooper_answer/` 和 `BugsInPy/projects/` 内含有 `bug_patch.txt`、`fixed_commit_id` 等答案。
- **交给 agent 的源码工作树必须无 git 历史**，且不含 `bugsinpy_bug.info` / `bugsinpy_patchfile.info` / `bugsinpy_run_test.sh` 等 BugsInPy 元数据副本。
- `复现测试文件`（取自 fixed commit）**允许提供**：它描述的是期望行为（测试规格），不是修复代码；请只把它放到源码的测试目录中。
- 评测校验使用数据集侧产物（`*-fail.txt` / `*-pass.txt`），这些不要暴露给 agent。
- 若 agent 具备联网能力，注意 `buggy_commit_id` 本身仍可在 GitHub 上继续浏览；如需更严格隔离，请在离线镜像中只提供该 commit 的源码快照。

## Bug 一览

### bug 1
- python_version: `3.8.1`
- buggy_commit_id: `e21a31162f4c54be693d8ca8260e42393b39abd3`
- test_file: `tests/test_chinese.py`
- 复现测试文件: `test_chinese.py`

复现命令:
```
pytest -q -s tests/test_chinese.py::test_chinese
```

### bug 2
- python_version: `3.8.1`
- buggy_commit_id: `e21a31162f4c54be693d8ca8260e42393b39abd3`
- test_file: `tests/test_pysnooper.py`
- 复现测试文件: `test_pysnooper.py`

复现命令:
```
pytest -q -s tests/test_pysnooper.py::test_custom_repr_single
```

### bug 3
- python_version: `3.8.1`
- buggy_commit_id: `6e3d797be3fa0a746fb5b1b7c7fea78eb926c208`
- test_file: `tests/test_pysnooper.py`
- 复现测试文件: `test_pysnooper.py`

复现命令:
```
pytest -q -s tests/test_pysnooper.py::test_file_output
```