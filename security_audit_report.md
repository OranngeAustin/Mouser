# 深度安全审计与后门检测报告

**审计目标**：Mouser - Logitech Mouse Remapper
**审计范围**：全代码库，包括 Python 源码 (`core/`, `tests/`, `main_qml.py` 等) 及构建和设置脚本 (`build_macos_app.sh`, `packaging/linux/install-linux-permissions.sh`)。
**审计时间**：2026-05-03

## 1. 供应链漏洞审计 (Supply Chain Audit)
**使用工具**：`pip-audit`, `bandit` (在隔离沙盒中运行)
**审计结果**：
- **依赖项扫描 (`pip-audit -r requirements.txt`)**：未在项目的依赖树中发现任何已知的 CVE 漏洞。依赖版本均安全。
- **静态应用安全测试 (`bandit`)**：未发现任何高危（High）漏洞。仅发现部分中等/低危（Medium/Low）问题，主要是单元测试脚本 (`tests/test_build_support.py` 和 `tests/test_startup.py`) 中存在硬编码的临时目录 `/tmp/...` (如 `B108:hardcoded_tmp_directory`)。
  - *潜在危害*：在测试或构建阶段可能会发生临时文件竞争或劫持，但这些代码不会在最终用户的生产环境中执行。
  - *修复建议*：在 `tests/` 下使用 Python 标准库 `tempfile` 生成随机的临时目录而不是硬编码的 `/tmp/...`。

## 2. 异常外连 (Exfiltration Checks)
**审计结果**：**安全**
**分析说明**：
经过对整个项目的源代码（特别是核心逻辑及入口文件 `main_qml.py`, `core/` 下所有文件）的手工检查和关键字自动化搜索 (`requests`, `urllib`, `socket`，`http` 等)：
- **无隐蔽网络请求**：除了导入过 `urllib.parse`（用于处理本地应用的路径编码）以外，未发现任何网络请求库的发包行为。
- **IPC 机制安全**：在 `main_qml.py` 中发现使用了 `QLocalSocket` 进行本地进程间通信 (IPC)，但这纯粹用于单实例应用检查（当程序已在运行，再次启动时拉起已有窗口）。
- **无数据外传机制**：未发现任何能够将捕获的鼠标键盘数据发送到外部网络的隐藏 IP 或域名。

## 3. 后门与隐藏逻辑 (Backdoors & Logic Bombs)
**审计结果**：**安全**
**分析说明**：
- **未发现代码混淆**：代码库整洁，未使用 `eval()`, `exec()`, `__import__()` 等易被用于执行隐藏逻辑的函数。
- **合法的子进程调用**：代码库中使用 `subprocess` 的地方均为正常的本地功能调用，例如：
  - `core/startup.py`：调用 `launchctl` 处理 macOS 下的自启动 (`LaunchAgent`)。
  - `core/version.py`：调用 `git` 获取当前提交信息，在发生错误时也妥善捕获了异常。
  - `core/app_detector.py`：在 Linux 系统中调用 `xdotool` / `kdotool` 来检测前台应用窗口，以匹配情景模式。

## 4. 权限提升风险 (Privilege Escalation)
**审计结果**：**可疑（需注意），但无恶意**
**文件定位**：
- `packaging/linux/install-linux-permissions.sh` (全篇)
**分析说明**：
该安装脚本如果未使用 `root` 执行，会主动调用 `sudo` 或 `pkexec` 请求提权。
- *潜在危害*：作为提权脚本，它具有较高的系统权限。
- *安全确认*：经过代码审查，该脚本的功能非常有限且目的正当。它仅仅将本地包含的 udev 规则文件 (`69-mouser-logitech.rules`) 复制到 `/etc/udev/rules.d/` 目录下，并执行 `modprobe uinput` 与 `udevadm` 进行重载。该 udev 规则使用 `TAG+="uaccess"` 技术，将 `/dev/hidraw` 和 `/dev/uinput` 的读写权限动态赋予当前活跃的桌面会话用户，从而**避免了让 Mouser 主程序本身以 root 权限运行**。
- *修复建议*：虽然行为合法，但从安全最佳实践来看，可以建议在脚本开头明确显示脚本即将执行的所有操作，并提示用户审阅 `69-mouser-logitech.rules` 文件，以增强用户的信任。

## 5. 敏感数据处理 (Sensitive Data Handling)
**审计结果**：**安全**
**分析说明**：
- **日志存储**：项目的日志系统 (`core/log_setup.py`) 会将程序的标准输出重定向到本地的 `Mouser/logs/mouser.log`。
- **无敏感记录**：虽然在 `core/key_simulator.py` 中存在诸如 `print(f"[KeySimulator] execute_action({action_id})")` 的输出，但这仅记录了触发的宏动作 ID（如 `alt_tab`），并**没有记录用户实际敲击的任意字母或文本**。
- 因此，不会导致用户的敏感密码或隐私数据以明文的形式记录在本地日志中。

## 结论
经过深度审计，Mouser 代码库**没有包含任何后门、木马、外连以及恶意提权逻辑**。该项目是一个严格在本地运行的辅助工具，其权限请求和系统操作均是为了完成合法的鼠标改键、配置存储以及前台应用检测功能。依赖项版本均为安全状态。

目前发现的唯一需要改进的地方是测试脚本中的硬编码临时目录（存在本地低度风险），不影响最终用户的安全体验。