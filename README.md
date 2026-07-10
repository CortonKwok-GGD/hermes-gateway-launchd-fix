# Hermes Gateway launchd auto-start fix for macOS 26+

**Issue**: macOS 26.5.2 and later: Hermes Gateway fails to auto-start after reboot despite `RunAtLoad=true` and `KeepAlive=true` in the launchd plist.

**Root cause**: Two independent but compounding problems in `hermes_cli/gateway.py`:

1. **`LimitLoadToSessionType` in plist template** — On macOS 26.5.2, this key causes launchd to silently fail loading the plist during login auto-scan. Removing it (it's unnecessary — the absence defaults to "all session types") restores correct loading on all macOS versions.

2. **`launchctl bootstrap` returns exit 5 on macOS 26+** — All 6 call sites that use `launchctl bootstrap` fail with EIO. The fallback (`_launchd_fallback_to_detached`) only triggers during manual `hermes gateway start/install`, not during launchd's auto-load on login — so reboot auto-start silently fails.

**Fix**: Replace `launchctl bootstrap` with `launchctl load` + `launchctl enable` on macOS 26+. The older `load` command (available since macOS 10.4) is unaffected by this regression. Keep `bootstrap` on older macOS for API consistency.

## Changes

**File**: `hermes_cli/gateway.py`

| Change | Lines | Description |
|--------|-------|-------------|
| Remove `LimitLoadToSessionType` | plist template | Causes silent plist load failure on macOS 26.5.2 |
| Add `_is_macos_26_or_later()` | new | Version detection helper |
| Add `_launchd_load()` | new | Unified loader: `load+enable` on 26+, `bootstrap` on older |
| Replace 6 `bootstrap` calls | install/start/restart/refresh | All switching/fallback paths now use `_launchd_load()` |

## Verification

| Test | Result |
|------|--------|
| `launchctl load` | exit 0 |
| `launchctl enable` | exit 0 |
| `launchctl print` active count | 1 |
| Gateway process | Running |
| WeChat connection | Connected |
| Reboot auto-start | ✅ Confirmed working |

## Patch

```
patch/patch -p1 < patch/gateway-launchd-fix.diff
```

## Related Hermes Issue

- #23387 — `launchctl bootstrap` exit 5 on macOS 26+

## License

MIT — same as Hermes Agent.
