# Hermes Gateway macOS 重启自启修复报告

> 作者：Corton Kwok
> 环境：macOS 26.5.2 (Sequoia), Hermes Agent v0.17.0-cn.5
> Hermes Issue: #23387

---

## 问题

macOS 重启后，Hermes Gateway（launchd 服务）无法自动启动。尽管 plist 配置了 `RunAtLoad=true` 和 `KeepAlive=true`，Gateway 进程在重启后未出现。

## 影响

- Gateway 不自启 → 消息平台（微信等）离线
- Cron scheduler（运行在 Gateway 进程中）不自启 → watchdog 脚本不触发
- 需要手动 `hermes gateway start` 才能恢复

## 根因

### 两个独立但叠加的问题

#### 问题 A：plist 中的 `LimitLoadToSessionType` 导致 macOS 26.5.2 上加载失败

`hermes gateway install` 生成的 plist 模板包含：

```xml
<key>LimitLoadToSessionType</key>
<array>
    <string>Aqua</string>
    <string>Background</string>
</array>
```

在 macOS 26.5.2 上，launchd 解析带这个 key 的 plist 时，自动加载过程静默失败。即使手动 `launchctl bootstrap` 或 `launchctl load -w` 也会返回 exit code 5（Input/output error）。

删除该 key 后，plist 在所有 session type 下均能正常加载。

#### 问题 B：`launchctl bootstrap` 在 macOS 26+ 上返回 exit 5

Hermes 源码在 5 个关键路径上使用 `launchctl bootstrap` 来加载服务定义：

- `launchd_install()` — 首次安装服务
- `launchd_start()` — 启动服务（plist 丢失路径 + 重载路径）
- `launchd_restart()` — 重启服务
- `refresh_launchd_plist_if_needed()` — 更新 plist 后重载
- `launchd_plist_is_current()` → `refresh_launchd_plist_if_needed()` — 内部 deferred reload（shell 脚本）

在 macOS 26+ 上，`launchctl bootstrap` 在所有场景下均返回 exit code 5（EIO）。Hermes 已有兜底逻辑（`_launchd_fallback_to_detached`），但该兜底只在手动 `start`/`install` 时触发——重启时的自动加载过程没有 Hermes 代码在运行，因此无法触发兜底。

替代方案 `launchctl load`（不带 `-w` 标志）+ `launchctl enable` 在所有 macOS 版本上均能正常工作。

## 修复方案

### 改动 1：删除 `LimitLoadToSessionType`

从 `generate_launchd_plist()` 的 plist 模板中移除该 key。删除后 plist 默认允许所有 session type 加载，这在旧版 macOS 上同样是正确行为。

### 改动 2：macOS 版本感知的加载

新增 `_is_macos_26_or_later()` 检测函数和 `_launchd_load()` 统一加载函数：

```python
def _is_macos_26_or_later() -> bool:
    """Return True on macOS 26+ where launchctl bootstrap is broken (exit 5)."""
    if sys.platform != "darwin":
        return False
    try:
        import platform
        major = int(platform.mac_ver()[0].split(".")[0])
        return major >= 26
    except Exception:
        return False

def _launchd_load(plist_path) -> None:
    """Load plist: macOS 26+ uses load+enable, older uses bootstrap."""
    if _is_macos_26_or_later():
        subprocess.run(["launchctl", "load", str(plist_path)], ...)
        subprocess.run(["launchctl", "enable", f"{domain}/{label}"], ...)
    else:
        subprocess.run(["launchctl", "bootstrap", domain, str(plist_path)], ...)
```

### 改动 3：替换所有 `launchctl bootstrap` 调用

共计 6 处，全部替换为 `_launchd_load()` 或版本感知的 shell 脚本。

## 为什么选择保守方案（版本检测）

- 对旧版 macOS 保持 Apple 推荐的现代 API（`bootstrap`）
- 仅在 macOS 26+ 切换到经过验证的 `load` + `enable`
- 最大程度降低 PR 审核风险
- **简洁方案备选**：所有版本统一使用 `load` + `enable`，删除 `bootstrap` 路径。经测试在旧版 macOS 上同样有效，`load` 是 launchd 最原始的命令（10.4 起），兼容性优于 `bootstrap`。

## 验证

| 测试 | 结果 |
|------|------|
| 语法编译 (`py_compile`) | ✅ |
| `launchctl load` exit code | 0 ✅ |
| `launchctl enable` exit code | 0 ✅ |
| `launchctl print` active count | 1 ✅ |
| Gateway process | PID 35752 运行中 ✅ |
| 微信连接 | Connected ✅ |
| 重启后自启 | ✅（用户确认） |

## 涉及文件

- `hermes_cli/gateway.py` — 全部修改在此文件
- `~/Library/LaunchAgents/ai.hermes.gateway.plist` — 重新生成的 plist（无 LimitLoadToSessionType）

## 相邻问题：Cron ticker 依赖 Gateway

本修复解决了重启后 Gateway 的自启问题。但还需注意一个架构限制：Hermes Cron 的自动 ticker（60 秒循环）位于 Gateway 进程内部。当 Gateway 因为手动 `stop` 而卸载时，Cron ticker 也随之停止，watchdog 脚本无法自动触发。

此问题不在本 PR 范围内，但值得后续讨论：
- 在 Desktop Dashboard 进程中运行独立的 cron ticker（已有代码但 env var 条件未匹配）
- 或通过 launchd agent 独立运行 `hermes cron tick`

## 提交历史（建议）

```
fix(gateway): launchd auto-start broken on macOS 26+ (exit 5)

- Remove LimitLoadToSessionType from plist template (causes silent
  load failure on macOS 26.5.2)
- Replace launchctl bootstrap with load+enable on macOS 26+
- Add _is_macos_26_or_later() and _launchd_load() helpers
- Fixes reboot auto-start regression (#23387)
```
