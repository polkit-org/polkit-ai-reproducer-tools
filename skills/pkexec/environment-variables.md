# pkexec: Environment Variable Handling

## When to use
When reproducing bugs related to pkexec's environment sanitization,
PATH handling, or variable passing behavior.

## How pkexec handles the environment
pkexec runs the target command with a **sanitized environment**. It:

1. Clears most environment variables
2. Sets a safe PATH: `/usr/sbin:/usr/bin:/sbin:/bin`
3. Preserves only a whitelist (LANG, TERM, etc.)
4. Sets PKEXEC_UID to the calling user's UID

## Key variables pkexec preserves
- `LANG` — locale (but see LC_MESSAGES bug)
- `TERM` — terminal type
- `PKEXEC_UID` — original user's UID (set by pkexec)

## Key variables pkexec strips
- `HOME` — reset to target user's home
- `PATH` — reset to safe default
- `LD_LIBRARY_PATH` — stripped for security
- `LD_PRELOAD` — stripped for security
- Custom variables — stripped unless whitelisted in the .policy file

## Testing environment sanitization
```bash
# See what environment pkexec passes through
runuser -u testuser -- pkexec env | sort

# Compare with the calling environment
runuser -u testuser -- env | sort
```

## Policy file environment whitelist
Actions can declare allowed environment variables in their .policy XML:

```xml
<action id="com.example.myaction">
  <annotate key="org.freedesktop.policykit.exec.path">/usr/bin/myprogram</annotate>
  <annotate key="org.freedesktop.policykit.exec.allow_gui">true</annotate>
</action>
```

## Gotchas
- If testing PATH-related bugs, remember pkexec hardcodes its own PATH.
  The user's PATH is NOT passed through.
- `PKEXEC_UID` is a string, not an integer — some programs parse it wrong.
- pkexec's SETUID bit must be set (`chmod 4755`) for it to work. In
  containers, check with `ls -la $(which pkexec)`.
