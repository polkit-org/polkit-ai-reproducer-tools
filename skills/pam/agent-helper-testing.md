# PAM: Authentication Agent Helper Testing

## When to use
When reproducing bugs in the authentication flow — PAM configuration,
polkit-agent-helper-1, password validation, or agent session issues.

## How polkit authentication works

1. User runs `pkexec command` → pkexec asks polkitd for authorization
2. polkitd says "auth required" → triggers the authentication agent
3. pkttyagent (TTY agent) shows "Password:" prompt
4. User types password → pkttyagent passes it to polkit-agent-helper-1
5. polkit-agent-helper-1 validates via PAM → reports success/failure
6. polkitd grants/denies authorization

## Key binaries
```
/usr/lib/polkit-1/polkit-agent-helper-1   — PAM helper (SETUID root)
/usr/bin/pkttyagent                        — TTY authentication agent
```

## PAM configuration
polkit uses the `polkit-1` PAM service. Config at:
```
/etc/pam.d/polkit-1
```

Typical contents:
```
auth       include      system-auth
account    include      system-account
password   include      system-password
session    include      system-session
```

## Testing polkit-agent-helper-1 directly
```bash
# The helper reads username on argv and password on stdin
# It's SETUID root, so it can be run by any user
echo "testpass" | /usr/lib/polkit-1/polkit-agent-helper-1 testuser
echo $?  # 0 = success, non-zero = failure
```

## Testing pkttyagent
```bash
# Register pkttyagent for the current session
pkttyagent --notify-fd 5 --fallback 5>/dev/null &
AGENT_PID=$!

# Now trigger an auth prompt
pkexec whoami

kill $AGENT_PID
```

## Gotchas
- polkit-agent-helper-1 must be SETUID root. Check:
  `ls -la /usr/lib/polkit-1/polkit-agent-helper-1`
  Should show `-rwsr-xr-x` (note the `s`).
- If PAM auth fails, check `/var/log/secure` (Fedora) or
  `journalctl -t polkit-agent-helper-1` for details.
- pkttyagent needs a TTY (stdin must be a terminal). In Docker,
  use `docker exec -t` to allocate one.
- The helper binary path varies by distro:
  - Fedora: `/usr/lib/polkit-1/polkit-agent-helper-1`
  - Ubuntu: `/usr/lib/policykit-1/polkit-agent-helper-1`
  Find it with: `find / -name polkit-agent-helper-1 2>/dev/null`
