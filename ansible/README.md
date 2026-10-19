# ansible —— Bug 修复任务（题面，不含答案）

- 上游仓库: https://github.com/ansible/ansible
- 语言/Python 版本: 见各 bug 的 bug.info
- 共 18 个 bug

本目录只给"是什么 bug + 如何获取源码 + 如何复现 + 复现测试文件"，
**不含**任何修复答案（官方补丁 bug_patch.txt、fixed_commit_id 已全部移出，见 `ansible_answer`）。

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
curl -L -o ansible.zip "https://codeload.github.com/ansible/ansible/zip/<bug.info 中的 buggy_commit_id>"
unzip ansible.zip
```
解压得到 `ansible-<commit>/`，即该 buggy commit 的完整源码，压缩包内不含任何 git 历史或修复提交。

方式二：只拉取该 commit 的浅历史
```
git init ansible && cd ansible
git remote add origin https://github.com/ansible/ansible.git
git fetch --depth 1 origin <buggy_commit_id>
git checkout FETCH_HEAD
```

方式三：完整 clone 后立即清除历史
```
git clone https://github.com/ansible/ansible.git
cd ansible
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

- **只把本目录（题面）挂载进 agent 沙箱**：不要挂载 `实验环境` 根目录或 `BugsInPy/`——同级的 `ansible_answer/` 和 `BugsInPy/projects/` 内含有 `bug_patch.txt`、`fixed_commit_id` 等答案。
- **交给 agent 的源码工作树必须无 git 历史**，且不含 `bugsinpy_bug.info` / `bugsinpy_patchfile.info` / `bugsinpy_run_test.sh` 等 BugsInPy 元数据副本。
- `复现测试文件`（取自 fixed commit）**允许提供**：它描述的是期望行为（测试规格），不是修复代码；请只把它放到源码的测试目录中。
- 评测校验使用数据集侧产物（`*-fail.txt` / `*-pass.txt`），这些不要暴露给 agent。
- 若 agent 具备联网能力，注意 `buggy_commit_id` 本身仍可在 GitHub 上继续浏览；如需更严格隔离，请在离线镜像中只提供该 commit 的源码快照。

## Bug 一览

### bug 1
- python_version: `3.6.9`
- buggy_commit_id: `25c5388fdec9e56517a93feb5e8d485680946c25`
- test_file: `test/units/galaxy/test_collection.py`
- 复现测试文件: `test_collection.py`

复现命令:
```
pytest test/units/galaxy/test_collection.py::test_verify_collections_no_version
```

### bug 2
- python_version: `3.6.9`
- buggy_commit_id: `de59b17c7f69d5cfb72479b71776cc8b97e29a6b`
- test_file: `test/units/utils/test_version.py`
- 复现测试文件: `test_version.py`

复现命令:
```
pytest test/units/utils/test_version.py::test_alpha
pytest test/units/utils/test_version.py::test_numeric
```

### bug 3
- python_version: `3.6.9`
- buggy_commit_id: `70219df9056ffb1e2766f572fbe71f7a1800c9f5`
- test_file: `test/units/module_utils/test_distribution_version.py`
- 复现测试文件: `test_distribution_version.py`

复现命令:
```
pytest test/units/module_utils/test_distribution_version.py::test_distribution_version
```

### bug 4
- python_version: `3.6.9`
- buggy_commit_id: `d91658ec0c8434c82c3ef98bfe9eb4e1027a43a3`
- test_file: `test/units/playbook/test_collectionsearch.py`
- 复现测试文件: `test_collectionsearch.py`

复现命令:
```
pytest test/units/playbook/test_collectionsearch.py::test_collection_static_warning
```

### bug 5
- python_version: `3.6.9`
- buggy_commit_id: `2af76f16be8cf2239daaec4c2f31c3dcb4e3469e`
- test_file: `test/units/module_utils/common/validation/test_check_required_arguments.py`
- 复现测试文件: `test_check_required_arguments.py`

复现命令:
```
pytest test/units/module_utils/common/validation/test_check_required_arguments.py::test_check_required_arguments_missing_multiple
```

### bug 6
- python_version: `3.6.9`
- buggy_commit_id: `90898132e456ee1993db99a1531379f1b98ee915`
- test_file: `test/units/galaxy/test_collection_install.py`
- 复现测试文件: `test_collection_install.py`

复现命令:
```
pytest test/units/galaxy/test_collection_install.py::test_build_requirement_from_path_with_manifest
pytest test/units/galaxy/test_collection_install.py::test_build_requirement_from_path_no_version
pytest test/units/galaxy/test_collection_install.py::test_add_collection_requirement_to_unknown_installed_version
```

### bug 7
- python_version: `3.6.9`
- buggy_commit_id: `cd146b836e032df785ecd9eb711c6ef23c2228b8`
- test_file: `test/units/modules/network/eos/test_eos_vlans.py`
- 复现测试文件: `test_eos_vlans.py`

复现命令:
```
pytest test/units/modules/network/eos/test_eos_vlans.py::TestEosVlansModule::test_eos_vlan_replaced
```

### bug 8
- python_version: `3.6.9`
- buggy_commit_id: `81378b3e744cd0d13b33d18a4f8a38aeb8a6e97a`
- test_file: `test/units/plugins/shell/test_powershell.py`
- 复现测试文件: `test_powershell.py`

复现命令:
```
pytest test/units/plugins/shell/test_powershell.py::test_join_path_unc
```

### bug 9
- python_version: `3.6.9`
- buggy_commit_id: `9276dd2007a73172ed99ab5a56cded4298d3cd2b`
- test_file: `test/units/modules/packaging/os/test_redhat_subscription.py`
- 复现测试文件: `test_redhat_subscription.py`

复现命令:
```
pytest test/units/modules/packaging/os/test_redhat_subscription.py::test_redhat_subscribtion
```

### bug 10
- python_version: `3.6.9`
- buggy_commit_id: `e368f788f71c338cd3f049d5d6bdc643a51c0514`
- test_file: `test/units/modules/system/test_pamd.py`
- 复现测试文件: `test_pamd.py`

复现命令:
```
pytest test/units/modules/system/test_pamd.py::PamdServiceTestCase::test_remove_first_rule
pytest test/units/modules/system/test_pamd.py::PamdServiceTestCase::test_remove_last_rule
```

### bug 11
- python_version: `3.6.9`
- buggy_commit_id: `da07b98b7a433493728ddb7ac7efbd20b8988776`
- test_file: `test/units/modules/network/ios/test_ios_banner.py`
- 复现测试文件: `test_ios_banner.py`

复现命令:
```
pytest test/units/modules/network/ios/test_ios_banner.py::TestIosBannerModule::test_ios_banner_nochange
pytest test/units/modules/network/ios/test_ios_banner.py::TestIosBannerIos12Module::test_ios_banner_nochange
```

### bug 12
- python_version: `3.6.9`
- buggy_commit_id: `05e2e1806162393d76542a75c2520c7d61c2d855`
- test_file: `test/units/plugins/lookup/test_env.py`
- 复现测试文件: `test_env.py`

复现命令:
```
pytest test/units/plugins/lookup/test_env.py
```

### bug 13
- python_version: `3.6.9`
- buggy_commit_id: `41472ee3878be215af8b933b2b04b4a72b9165ca`
- test_file: `test/units/cli/test_galaxy.py`
- 复现测试文件: `test_galaxy.py`

复现命令:
```
pytest test/units/cli/test_galaxy.py::test_collection_install_with_url
```

### bug 14
- python_version: `3.6.9`
- buggy_commit_id: `a1ab093ddbd32f1002cbf6d6f184c7d0041d890d`
- test_file: `test/units/galaxy/test_api.py`
- 复现测试文件: `test_api.py`

复现命令:
```
pytest test/units/galaxy/test_api.py::test_get_role_versions_pagination
```

### bug 15
- python_version: `3.6.9`
- buggy_commit_id: `b1e8a6c1cbd2a668b462995487b819ef7dd8ba4b`
- test_file: `test/units/modules/network/eos/test_eos_eapi.py`
- 复现测试文件: `test_eos_eapi.py`

复现命令:
```
pytest test/units/modules/network/eos/test_eos_eapi.py::TestEosEapiModule::test_eos_eapi_vrf
```

### bug 16
- python_version: `3.6.9`
- buggy_commit_id: `2a9964ede8b2b77a62a005f6f5abc964b2819b0e`
- test_file: `test/units/module_utils/facts/hardware/test_linux_get_cpu_info.py`
- 复现测试文件: `test_linux_get_cpu_info.py`

复现命令:
```
pytest test/units/module_utils/facts/hardware/test_linux_get_cpu_info.py::test_get_cpu_info_missing_arch
```

### bug 17
- python_version: `3.6.9`
- buggy_commit_id: `9cb47832d15c61884b30d70f9d4e0f816b064b05`
- test_file: `test/units/module_utils/facts/test_facts.py`
- 复现测试文件: `test_facts.py`

复现命令:
```
pytest test/units/module_utils/facts/test_facts.py::TestFactsLinuxHardwareGetMountFacts::test_get_mount_facts
```

### bug 18
- python_version: `3.6.9`
- buggy_commit_id: `bb1256ca9aa4c22225dbeef0ef23a20fa9388b2f`
- test_file: `test/units/cli/test_galaxy.py`
- 复现测试文件: `test_galaxy.py`

复现命令:
```
pytest test/units/cli/test_galaxy.py::test_collection_default
```