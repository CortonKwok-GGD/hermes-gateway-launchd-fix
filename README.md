# Hermes Gateway macOS 26+ — launchd Auto-Start Fix

**Issue**: macOS 26.5.2 and later: Hermes Gateway fails to auto-start after reboot despite `RunAtLoad=true` and `KeepAlive=true` in the launchd plist.

**PR**: [#62223](https://github.com/NousResearch/hermes-agent/pull/62223)

**Root cause**: Two independent but compounding problems in `hermes_cli/gateway.py`:

1. **`LimitLoadToSessionType` in plist template** — On macOS 26.5.2, this key causes launchd to silently fail loading the plist during login auto-scan. But removing it entirely breaks Background-session support (SSH / non-Aqua logins). **Fix**: Version-conditional — omit on macOS 26+, preserve `[Aqua, Background]` on older.

2. **`launchctl bootstrap` returns exit 5 on macOS 26+** — All call sites fail with EIO. The fallback (`_launchd_fallback_to_detached`) only triggers during manual `hermes gateway start/install`, not during launchd's auto-load on reboot. **Fix**: On macOS 26+, use `launchctl load` + `launchctl enable` instead. Keep `bootstrap` on older macOS for API consistency + EIO stale-label recovery.

## Changes

**File**: `hermes_cli/gateway.py` (+130/-40 lines)

| Change | Lines | Description |
|--------|-------|-------------|
| Conditional `LimitLoadToSessionType` | plist template | `_limit_load_section()` — omit on 26+, preserve `[Aqua, Background]` on < 26 |
| Add `_is_macos_26_or_later()` | new | Version detection helper using `platform.mac_ver()` |
| Modify `_launchctl_bootstrap()` | existing | macOS 26+ path: `load plist` + `enable domain/label`; preserve `< 26` bootstrap with stale-label recovery |
| Unify 6 call sites | install/start/restart/refresh | All go through `_launchctl_bootstrap()` or version-aware shell script |

**File**: `tests/hermes_cli/test_gateway_service.py` (+222 lines)

| Test | What it covers |
|------|----------------|
| `test_launchd_plist_omits_limit_load_on_macos_26` | macOS 26+ plist omits `LimitLoadToSessionType` |
| `TestLaunchctlBootstrapMacOs26` (3 tests) | `load/enable` normal / load fail / enable fail |
| `test_launchd_start_reloads_with_load_enable_on_macos_26` | `launchd_start()` kickstart fail→load/enable→kickstart |
| `test_launchd_restart_reloads_using_launchctl_bootstrap_on_macos_26` | `launchd_restart()` unloaded branch on 26+ |
| `TestRetryLaunchctlBootstrapUntilRegisteredMacOs26` (2 tests) | retry loop registration verify + TimeoutExpired retry |

## PR Review History

teknium1 (Hermes Agent author) raised 3 issues via automated code review:

1. **Unconditional `LimitLoadToSessionType` removal** → Fixed with version-conditional `_limit_load_section()`
2. **Incomplete coverage of bootstrap call sites** → Fixed: deferred reload script + restart recovery now version-aware
3. **No test changes** → Fixed: 8 new tests + autouse fixtures on existing tests

## Patches

- `patch/pr-62223.patch` — Original submission (before review fixes)
- `patch/pr-62223-fix.patch` — Final version addressing all review feedback

## Verification

| Test | Result |
|------|--------|
| `launchctl load` | exit 0 |
| `launchctl enable` | exit 0 |
| `launchctl print` active count | 1 |
| Gateway process | Running |
| WeChat connection | Connected |
| Reboot auto-start | ✅ Confirmed working |

## Related

- Hermes Issue #23387 — `launchctl bootstrap` exit 5 on macOS 26+
- PR #62223 — fix(gateway): macOS 26+ launchd auto-start

## License

MIT — same as Hermes Agent.
