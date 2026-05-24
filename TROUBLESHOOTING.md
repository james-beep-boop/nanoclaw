# NanoClaw v2 Troubleshooting & Known Issues

## CRITICAL: Service Management is Automatic — Do Not Manually Configure

**This is the single most important thing to understand about NanoClaw v2.**

NanoClaw's service is automatically configured and managed by the installation script (`bash nanoclaw.sh` → `setup/service.ts`). This creates a platform-specific service file and handles all OS integration:

- **Linux with systemd:** Creates `~/.config/systemd/user/nanoclaw-v2-[UUID].service` (auto-generated UUID-based name)
- **macOS:** Creates `~/Library/LaunchAgents/com.nanoclaw.plist`
- **Other platforms:** Uses bash wrapper with nohup fallback

**Reference architecture documentation:** https://github.com/nanocoai/nanoclaw/blob/main/setup/service.ts (authoritative source for service setup and design)

### DO NOT:
- ❌ Manually create or edit systemd service files
- ❌ Override environment variables outside the auto-generated service
- ❌ Use `node dist/index.js &` for process management
- ❌ Disable the auto-generated service in favor of a custom one

All of these create conflicts, multiple running instances, and port binding failures.

### DO:
- ✅ Use the auto-generated service file created by setup
- ✅ Manage via systemd: `systemctl --user status/restart/stop nanoclaw-v2-*`
- ✅ View logs: `journalctl --user -u nanoclaw-v2-* -f`
- ✅ Find the exact service name: `systemctl --user list-units | grep nanoclaw`

---

## ACTUAL FIXES (Applied 2026-05-24)

These are the TWO CODE CHANGES that resolved the persistent container timeout issues. Both have been applied and compiled.

### Fix 1: Disable Telegram Adapter (`src/channels/index.ts:11`)

**What was broken:**
- Chat SDK Telegram polling failed repeatedly with network errors
- Errors cascaded into blocking all message processing
- Containers appeared stuck, triggering the absolute ceiling kill logic

**The fix:**
```typescript
import './cli.js';
// Telegram disabled due to persistent polling network errors (see TROUBLESHOOTING.md)
// import './telegram.js';
```

**Why this works:**
- Telegram adapter no longer runs; no polling failures
- Other channels (CLI, etc.) continue working normally
- Prevents error cascade that was blocking legitimate container operations

**Status:** ✅ Applied and compiled

---

### Fix 2: Increase Container Timeout (`src/host-sweep.ts:65`)

**What was broken:**
- Original timeout was 30 minutes (`ABSOLUTE_CEILING_MS = 30 * 60 * 1000`)
- Legitimate container operations could exceed 30 minutes
- Containers killed unnecessarily even when heartbeat was being touched

**The fix:**
```typescript
export const ABSOLUTE_CEILING_MS = 60 * 60 * 1000;  // 60 minutes (was 30)
```

**Why this works:**
- Containers get up to 60 minutes before forced kill
- Heartbeat is touched independently every 5 seconds (separate mechanism)
- Only truly silent containers get killed; active ones with heartbeat are safe
- Gives legitimate operations more time without creating zombie processes

**Status:** ✅ Applied and compiled

---

## Why Other Approaches Failed

### Manual Systemd Service File (WRONG)
**What I tried:** Creating a custom systemd service file (`~/.config/systemd/user/nanoclaw.service`) with manual configuration.

**Why it failed:**
- The auto-generated service file from setup was already active and enabled
- Two conflicting service files managed the same process
- Both competed for port 3100 → multiple instances spawned → port binding failures
- Circuit breaker escalated restart delays after repeated failures
- The more I restarted, the worse it got (cascade effect)

**The lesson:** Service management is already solved by setup. Attempting to override it creates conflicts, not solutions.

---

### Manual Port Configuration (WRONG)
**What I tried:** Configure a different port (3101) via environment file to "work around" the port conflicts.

**Why it failed:**
- Root cause was two services managing the same process, not the port itself
- Port change only hid the symptom (duplicate services) while leaving the real problem
- Proper fix is to remove the duplicate service, not change ports

**The lesson:** Don't work around a systemic problem; fix the system.

---

### Manual Process Management (`node dist/index.js &`) (WRONG)
**What I tried:** Directly starting the process with `&` (background job) instead of using systemd.

**Why it failed:**
- No single point of control → multiple background instances could exist
- No prevention of duplicate binding → port conflicts
- No clean restart logic → circuit breaker would count every start/failure
- After 6+ failed restarts, circuit breaker would lock out new attempts for 15 minutes
- This made it appear the service was broken, when really multiple instances were fighting

**The lesson:** Ad-hoc process management cascades into failure. Use the OS-level service system.

---

## Issue: Persistent Heartbeat Timeouts & Container Kills (RESOLVED 2026-05-24)

**Resolved:** May 21-24, 2026

Both code fixes above (Telegram disabled + 60-minute timeout) were the solution. The system is now stable.

### Verification
- ✅ No Telegram polling errors in logs  
- ✅ Containers allowed up to 60 minutes before kill
- ✅ Service managed by auto-generated systemd service
- ✅ Dashboard accessible via auto-generated service
- ✅ Single process, no port conflicts

### After Code Changes
After editing the source code, rebuild and restart:
```bash
pnpm run build
systemctl --user restart nanoclaw-v2-*
journalctl --user -u nanoclaw-v2-* -f  # Monitor startup
```

### Long-term Improvements
Once this is stable:
1. Add circuit breaker to Chat SDK polling (fail-fast after N consecutive errors)
2. Add alerting for polling failures (don't silently backoff forever)
3. Consider graceful degradation: skip problematic channels, continue on others
4. Validate all channel credentials at startup, not just on first poll failure

---

## Historical Context: Learning from Failed Approaches (Archive)

The resolution path above involved multiple troubleshooting attempts that failed. Documented here for reference:

1. **Manual process management** (`node dist/index.js &`) — Created multiple running instances competing for port 3100, cascading into circuit breaker escalation and 15-minute lockout delays.

2. **Manual systemd service creation** — Attempted to create a custom service file, but the auto-generated service from setup was already installed and running, causing two services to compete for the same port.

3. **Port override via environment variable** — Tried to work around conflicts by changing the port, but the real issue was duplicate service management, not the port itself.

4. **Circuit breaker state file** — Reset `data/circuit-breaker.json` as a temporary measure, but the real problem was the duplicate service files fighting each other.

**Key lesson:** The setup script already handles all service management correctly via `setup/service.ts`. Manual modifications create conflicts instead of solving problems. Always verify what's already installed before adding new configuration.

For details on what went wrong with each approach, see the git history or the full context in the referenced GitHub architecture documentation.
