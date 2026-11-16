# 提交信息本地校验指南（pre-commit）

## 1. 概述

仓颉社区各代码仓通过仓库根目录的 `.pre-commit-config.yaml` 统一配置提交信息校验，规则与流水线门禁保持一致。配置完成后，`git commit` 时会自动执行：

| 检查项 | 校验内容 | 依赖 |
|---|---|---|
| `cangjie-commitlint` | 提交信息符合 Conventional Commits 规范 | Node.js + commitlint 17.8.1 |
| `cangjie-dco` | 提交信息包含 `Signed-off-by` 签名行 | pre-commit 内置 pygrep |

本地校验可在推送前发现提交信息问题，避免因门禁失败反复返工。

## 2. 依赖软件版本

按下表准备环境，版本不足时钩子会在安装或提交阶段报错。

| 依赖 | 最低版本 | 推荐版本 | 用途 / 版本依据 |
|---|---|---|---|
| Git | 2.30 | 最新稳定版 | 提供 `commit-msg` 钩子机制与 `core.hooksPath` 行为 |
| Python | 3.9 | 3.10+ | pre-commit 自身运行环境 |
| pre-commit | 2.18 | 4.x（4.6.2 已验证） | 低于 2.18 不支持 `--hook-type` 与 `default_install_hook_types` |
| Node.js | 14 | 20 / 22 LTS（24 亦可） | commitlint 17.8.1 的 `engines` 声明为 `>=v14`；18 及以下已 EOL |
| npm | 随 Node.js 安装 | 与 Node.js 版本匹配 | 安装 commitlint 依赖 |

版本确认：

```bash
git --version
python3 --version      # Windows: python --version
pre-commit --version
node --version
npm --version
```

注意事项：

- **Node.js 必须与 npm 同时在 `PATH` 中**。两者缺失时 pre-commit 会回退到 nodeenv 从 nodejs.org 下载二进制，在内网环境通常长时间卡住；
- 使用 nvm / nvm-windows 时，需确认当前终端已执行 `nvm use`（`nvm use --lts`），否则子进程看不到 node；
- **Windows 上不要使用 `python3` 命令**，它常指向应用商店占位符；请使用 `python` 或 `py -3`。

## 3. 安装 pre-commit

### 3.1 Linux / macOS

安装到用户目录（`--user`），不污染系统 Python，也不需要 root 权限。注意第 2 步：

```bash
# 1) 安装到用户目录
python3 -m pip install --user -i https://mirrors.huaweicloud.com/repository/pypi/simple pre-commit

# 2) 关键：脚本被装到 ~/.local/bin，当前终端的 PATH 里还没有它，
#    直接敲 pre-commit 会报 "pre-commit: command not found"。
export PATH="$HOME/.local/bin:$PATH"      # 当前终端立即生效

# 3) 验证（未刷新 PATH 时可用 ~/.local/bin/pre-commit --version）
pre-commit --version

# 4) 升级
python3 -m pip install --user -i https://mirrors.huaweicloud.com/repository/pypi/simple -U pre-commit
```

补充说明：

- 长期生效建议在 `~/.bashrc`（zsh 为 `~/.zshrc`）中保留一行：`export PATH="$HOME/.local/bin:$PATH"`，重开终端后自动生效；
- Ubuntu 24.04 及以上的系统 Python 受 PEP 668 保护，`--user` 安装会报 `externally-managed-environment`，此时改用系统包 `sudo apt install pre-commit`（**需确认版本 ≥ 2.18**），或创建虚拟环境后安装，详见 7.1。

pip 镜像（华为云），命令行配置一次即可，之后 pip 均生效：

```bash
python3 -m pip config set global.index-url https://mirrors.huaweicloud.com/repository/pypi/simple
python3 -m pip config set global.trusted-host mirrors.huaweicloud.com
python3 -m pip config list        # 确认已写入
```

上述 `pip config set` 等价于写入 `~/.config/pip/pip.conf`，也可直接编辑该文件：

```ini
[global]
index-url = https://mirrors.huaweicloud.com/repository/pypi/simple
trusted-host = mirrors.huaweicloud.com
```

### 3.2 Windows

Windows 上钩子由 Git for Windows 自带的 `sh` 执行，**无需 WSL**。

```powershell
# 0) 确认 Python 可用（python.org 安装包需勾选 "Add python.exe to PATH"）
python --version

# 1) 安装到用户目录
python -m pip install --user pre-commit

# 2) 确认脚本目录已进入用户级 PATH（只需一次），之后必须重开终端
python -m site --user-base        # 其下的 Scripts 目录即 pre-commit.exe 所在位置
#    未在 PATH 中时手动追加用户级 PATH：
#    不要用 setx PATH "%PATH%;..."，它可能截断过长 PATH
#    [Environment]::SetEnvironmentVariable("Path", "$env:Path;$env:APPDATA\Python\Python3XX\Scripts", "User")

# 3) 重开终端后验证（未重开前可用 %APPDATA%\Python\Python3XX\Scripts\pre-commit.exe --version）
pre-commit --version
```

`pip --user` 的可执行文件默认位置：`%APPDATA%\Python\Python3XX\Scripts`。

pip 镜像（国内网络）：

```powershell
python -m pip config set global.index-url https://mirrors.huaweicloud.com/repository/pypi/simple
python -m pip config set global.trusted-host mirrors.huaweicloud.com
```

npm 镜像（国内网络）：

```powershell
npm config set registry https://repo.huaweicloud.com/repository/npm/
```

需要代理时：

```powershell
$env:HTTPS_PROXY="http://<proxy-host>:<port>"
$env:HTTP_PROXY="http://<proxy-host>:<port>"
npm config set proxy http://<proxy-host>:<port>
npm config set https-proxy http://<proxy-host>:<port>
```

Windows 特有约定：`commitlint` 运行在 pre-commit 为 `language: node` 钩子创建的独立环境中，该环境由 nodeenv 生成。nodeenv 不支持在 Windows 上复用系统 Node（`-n system` 会直接报 `Installing system node.js on win32 is not supported!`），因此 Windows 首次执行 `--install-hooks` 时会从 nodejs.org 下载一份 Node。内网或下载缓慢时，在 `%USERPROFILE%\.nodeenvrc` 指定镜像：

```ini
[nodeenv]
mirror = https://npmmirror.com/mirrors/node/
```

下载结果缓存在 `%USERPROFILE%\.cache\pre-commit`，后续提交不会重复下载。

## 4. 在仓库中启用钩子

进入代码仓根目录（任意子目录亦可，只要 `git rev-parse --show-toplevel` 能定位到根）：

```bash
cd <代码仓>

# 安装 commit-msg 钩子，并预装 commitlint 运行环境
pre-commit install --hook-type commit-msg --install-hooks
```

- `--hook-type commit-msg`：只安装提交信息钩子，与 `.pre-commit-config.yaml` 中的 `default_install_hook_types: [commit-msg]` 一致；
- `--install-hooks`：立即创建运行环境（首次会下载 commitlint，可能耗时数分钟），避免首次提交时卡住；
- 仓库必须已包含 `.pre-commit-config.yaml`，否则提示 `No .pre-commit-config.yaml file was found`。

多仓工作区（`.repo` 管理）可在工作区根目录批量执行：

```bash
.repo/repo/repo forall -c 'pre-commit install --hook-type commit-msg --install-hooks'
```

注意：若 `.pre-commit-config.yaml` 存在未暂存改动，`git commit` 会被 pre-commit 拒绝执行并提示 `Your pre-commit configuration is unstaged.`，先 `git add .pre-commit-config.yaml` 即可。

## 5. 提交信息规范

提交信息格式：

```
<type>(<scope>): <subject>

<body>

Signed-off-by: Name <email@example.com>
```

- `type` 允许值：`feat`、`fix`、`docs`、`style`、`refactor`、`test`、`chore`、`revert`、`perf`、`build`、`ci`
- `scope` 可选，用于标注影响范围，如 `feat(compiler): ...`
- 标题不超过 100 字符，`subject` 结尾不使用句号
- 正文每行不超过 100 字符
- 推荐使用 `-s` 参数自动生成 DCO 签名行

合规示例：

```text
feat(lexer): support unicode identifiers

Signed-off-by: Zhang San <zhangsan@example.com>
```

```text
fix: avoid panic when parsing empty source file

Signed-off-by: Li Si <lisi@example.com>
```

不合规示例：

```text
update code                       # 缺少 type
feat: add feature.                # 标题以句号结尾
feat: add feature                 # 缺少 Signed-off-by
```

日常提交：

```bash
git commit -sm "feat: support multi-line string literal"
```

## 6. 校验与自测

```bash
# 确认 commit-msg 钩子已安装
ls .git/hooks/commit-msg          # Windows: dir .git\hooks\commit-msg

# 用示例提交信息手动触发校验（不产生提交）
echo "feat: test message" > /tmp/msg.txt
pre-commit run cangjie-commitlint --all-files --hook-stage commit-msg \
    --commit-msg-filename /tmp/msg.txt

# 查看已安装钩子与版本
pre-commit --version
```

`--all-files` 用于关闭 pre-commit 的 stash 分支，避免 `.pre-commit-config.yaml` 有未暂存改动时被拒绝执行；commit-msg 阶段实际校验的仍是 `--commit-msg-filename` 指定的文件。

`pre-commit run` 返回 0 表示通过，非 0 表示有问题（输出中会注明规则与位置）。也可直接构造一次真实提交验证：`git commit -sm "feat: test"`，如被拦截会打印 commitlint 的具体规则。

## 7. 常见问题

### 7.1 安装 pre-commit 阶段

| 现象 | 平台 | 原因 | 处理 |
|---|---|---|---|
| `externally-managed-environment` | Linux | 系统 Python 受 PEP 668 保护，禁止 `pip install --user` | 改用系统包 `sudo apt install pre-commit`（确认版本 ≥ 2.18），或创建虚拟环境：`python3 -m venv ~/.venvs/precommit && ~/.venvs/precommit/bin/pip install pre-commit`，后续用该 venv 内的 `pre-commit` |
| `pre-commit: command not found` / `不是内部或外部命令` | 全部 | 用户脚本目录不在 `PATH` | 加入 `~/.local/bin`（Windows 为 `%APPDATA%\Python\Python3XX\Scripts`）后重开终端，或直接用绝对路径调用 |
| `python3` 打开应用商店 | Windows | 命中了应用商店占位符 | 改用 `python` 或 `py -3` |
| `No module named pip` | 全部 | 当前 Python 未带 pip | `python -m ensurepip --upgrade` 后重试 |
| 安装过程网络超时 | 全部 | 未配置镜像或代理 | 按 3.1 / 3.2 配置 pip、npm 镜像或代理 |
| `pre-commit` 版本过旧导致参数不支持 | 全部 | 低于 2.18 不支持 `--hook-type` / `default_install_hook_types` | 升级到 4.x：`python3 -m pip install --user -U pre-commit` |

### 7.2 启用钩子与提交阶段

| 现象 | 平台 | 原因 | 处理 |
|---|---|---|---|
| `No .pre-commit-config.yaml file was found` | 全部 | 当前目录不是仓库根目录，或仓库未接入校验 | 在仓库根目录执行；确认仓库已提供该文件 |
| `Your pre-commit configuration is unstaged.` | 全部 | `.pre-commit-config.yaml` 有未暂存改动，pre-commit 拒绝执行 | `git add .pre-commit-config.yaml` 后重新提交 |
| 提交时未触发任何校验 | 全部 | 只安装了 `pre-commit` 类型钩子，未安装 `commit-msg` | 重新执行 `pre-commit install --hook-type commit-msg` |
| 安装钩子被拒绝，提示 `core.hooksPath` | 全部 | 本机设置了全局钩子目录 | `git config --unset core.hooksPath` 后重试，或联系管理员确认 |
| `未找到 node/npm` | 全部 | Node 未安装，或 nvm 未在当前 shell 生效 | 安装 LTS；nvm 用户先执行 `nvm use --lts` 再重开终端 |
| `Node.js 版本过低` | 全部 | commitlint 17.8.1 要求 Node >= 14 | 升级到 20 / 22 / 24 LTS |
| 日志显示下载 Node 二进制并卡住 | Linux/macOS | `PATH` 中没有 node/npm，pre-commit 回退到 nodeenv 从 nodejs.org 下载 | 安装 Node.js 并确保其在 `PATH` 中，删除缓存后重试：`pre-commit clean` |
| 日志显示下载 Node 二进制并卡住 | Windows | nodeenv 在 Windows 不支持复用系统 Node，必定下载一份 | 配置 `~/.nodeenvrc` 的 `mirror`（见 3.2）或代理后 `pre-commit clean` 重试 |
| 首次提交耗时较长 | 全部 | 运行环境尚未创建 | 启用钩子时带上 `--install-hooks` |
| `npm ERR! network` / 依赖拉取超时 | 全部 | 网络或代理限制 | 配置 npm 镜像或代理（见 3.1/3.2），再 `pre-commit clean` 重试 |
| Windows 报路径过长（260 字符） | Windows | 缓存目录 + node_modules 路径超限 | 管理员执行 `git config --system core.longpaths true`，并在系统策略中开启 LongPathsEnabled 后重启 |
| 钩子文件被系统安全软件删除 | Windows | 杀毒/EDR 拦截 `.git/hooks` 下脚本 | 将代码目录加入白名单，再重新安装钩子 |
| 提交信息校验通过但流水线仍失败 | 全部 | 门禁检查合并请求中的全部提交，历史提交存在不合规项 | 按 7.3 修正历史提交 |

### 7.3 修正不合规提交

修改最近一次提交（未推送）：

```bash
git commit --amend          # 修改提交信息
git commit --amend -s       # 仅补充 Signed-off-by
```

修改多条历史提交：

```bash
git rebase -i origin/main
```

将要修改的提交由 `pick` 改为 `reword`，保存后逐个编辑提交信息；rebase 过程同样会触发校验。完成后更新远端分支：

```bash
git push --force-with-lease
```

强制推送前应确认该分支无其他协作者提交。

### 7.4 清理缓存

钩子环境异常（依赖损坏、版本残留）时：

```bash
pre-commit clean        # 清理 ~/.cache/pre-commit（Windows: %USERPROFILE%\.cache\pre-commit）
pre-commit install --hook-type commit-msg --install-hooks
```

## 8. 临时跳过校验

以下方式仅建议在特殊情况（如紧急回滚）使用：

```bash
# 跳过指定检查项
SKIP=cangjie-commitlint git commit -sm "feat: xxx"

# 跳过全部本地钩子
git commit --no-verify -sm "feat: xxx"
```

本地跳过不会改变门禁结果，推送后仍会被流水线拦截。

## 9. 卸载

```bash
cd <代码仓>
pre-commit uninstall
```

卸载仅影响当前仓库，不修改全局 Git 配置；如需同时移除 pre-commit 本体：

```bash
python3 -m pip uninstall pre-commit
```
