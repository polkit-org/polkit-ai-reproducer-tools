# Component: polkitd - JavaScript Rules Logging and Non-Root Verification

## When to use
When reproducing bugs involving:
1. JavaScript authorization rules and Duktape engine behaviors.
2. Capturing `polkit.log()` output or rule-related debug logs in polkitd.
3. Executing and verifying rule evaluations for non-root users from within a non-root reproducer script.

## Technique
By default, the polkitd systemd unit starts with `--no-debug` and `--log-level=notice`. This results in:
- `stderr` and `stdout` being redirected to `/dev/null`, silencing any logs printed using `fprintf(stderr)` directly (like `js_polkit_log`).
- Info/Debug level messages being completely omitted from the journal.

Furthermore, reproducing as a non-root user `testuser` requires permissions to query `journalctl` to inspect output and verify the log content, as well as correctly targetted processes actually owned by `testuser`.

To resolve these, we can:
1. Modify `/usr/lib/systemd/system/polkit.service` to remove `--no-debug` and set `--log-level=debug`.
2. Add the `testuser` to the `systemd-journal` group so they can query the journal without requiring root privileges.
3. Spawn processes cleanly inside `testuser`'s shell so they are actually owned by `testuser` and subject to rule validation.

## Recipe

### 1. Environment Preparation (As Root)
```bash
# Configure polkit service for logging
if [ -f /usr/lib/systemd/system/polkit.service ]; then
    sed -i 's/polkitd --no-debug/polkitd/g' /usr/lib/systemd/system/polkit.service
    sed -i 's/--log-level=notice/--log-level=debug/g' /usr/lib/systemd/system/polkit.service
fi

# Reload systemd and restart polkit
systemctl daemon-reload
systemctl restart polkit

# Grant testuser permission to read systemd logs
usermod -aG systemd-journal testuser
```

### 2. Verification within non-root reproducer
```bash
# Spawn a process owned by testuser
sleep 100 &
sleep_pid=$!

test_start_time=$(date +"%Y-%m-%d %H:%M:%S")
sleep 1

# Trigger authorization check
pkcheck --process $sleep_pid --action-id org.freedesktop.policykit.exec --allow-user-interaction

# Retrieve logs generated during the run
logs=$(journalctl -u polkit --since "$test_start_time" --no-pager)
```

## Gotchas
- **Bypassing for Root:** Running `pkcheck` on a process owned by `root` (UID 0) often bypasses rule evaluation entirely. Always verify checks against processes actually owned by non-root users.
- **`runuser` Process Ownership:** Running `runuser -u testuser -- sleep 10 &` from a root shell runs `runuser` (owned by root) as the parent process. Use the background shell capability or run inside the non-root environment so the background pid is truly non-root.
- **Silent Rule Errors:** Syntax errors in JS rules files prevent them from loading, and polkitd will skip them. Check journalctl logs for any `Loaded and executed script` confirmation.
