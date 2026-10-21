# PAM: Socket-Activated Helper and faillock Reproduction

## When to use
When reproducing bugs involving polkit 127+ socket-activated `polkit-agent-helper-1`, PAM conversation handling, cancellation behavior, or `pam_faillock` lockout tallying.

## Technique
polkit 127 introduced socket activation for `polkit-agent-helper-1` via `/run/polkit/agent-helper.socket`.
In legacy mode (polkit <= 126 or when the socket unit is inactive), the agent forks `polkit-agent-helper-1` directly and terminates it with `SIGTERM` on cancellation (`kill_helper()`), preventing PAM from returning an error.
In socket-activated mode, `kill_helper()` only closes the client socket. The helper reads EOF (`fgets` returns NULL in `conversation_function()`) and returns `PAM_CONV_ERR`, causing PAM to process the failure and log it in `pam_faillock`.

To test this:
1. Start `polkit-agent-helper.socket` (`systemctl start polkit-agent-helper.socket`).
2. Enable `pam_faillock` in PAM using `authselect select local with-silent-lastlog with-faillock --force`.
3. Add the test user to `wheel` group so polkit admin rules ask for the test user's credentials rather than root's.
4. Use `expect` to send `\x04` (EOF / Ctrl+D) at the `Password:` prompt to trigger `polkit_agent_session_cancel()`.

## Recipe

### 1. Environment Setup (as root)
```bash
# Enable faillock in PAM
authselect select local with-silent-lastlog with-faillock --force

# Add user to wheel so auth_admin targets testuser
usermod -aG wheel testuser

# Start socket activation
systemctl start polkit-agent-helper.socket

# Clear faillock tally
faillock --user testuser --reset
```

### 2. Triggering Cancellation (as testuser)
```bash
expect << 'EOF'
set timeout 10
spawn pkexec true
expect {
    -re "Password:.*" {
        send "\x04"
        expect eof
    }
}
catch wait
EOF

# Check faillock tally
faillock --user $(id -un)
```

## Gotchas
- If `polkit-agent-helper.socket` is not started, polkit falls back to spawning the setuid helper directly (`session->child_pid > 0`), which sends `SIGTERM` on cancel and does not trigger the bug.
- If the test user is not in the `wheel` group, polkit's default admin rule targets `root`, and any auth failures will be tallied against `root` rather than the test user.
