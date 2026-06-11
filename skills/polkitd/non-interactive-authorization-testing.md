# Component: polkitd Non-Interactive Authorization Testing

## When to use
When reproducing backend or daemon-level bugs in `polkitd` (such as rules evaluation crashes, database/Duktape issues, configuration reloads, or D-Bus message handling) where you only need to trigger an authorization check rather than test the full interactive password authentication flow.

## Technique
Instead of using `pkexec`, which attempts to register a textual authentication agent on `/dev/tty` and fails with terminal errors in non-interactive/container environments, use `pkcheck`.
By passing `pkcheck` the PID of the calling user's shell and a standard action, you can cleanly trigger polkitd's rule evaluation engine from any non-privileged user script without a TTY or an interactive password prompt.

## Recipe
To trigger a non-interactive authorization check as `testuser` and capture the result:
```bash
# In the reproducer script run as testuser:
OUTPUT=$(pkcheck -p $$ -a org.freedesktop.policykit.exec 2>&1)
EXIT_CODE=$?

# If the daemon crashed:
# - EXIT_CODE will be 127
# - OUTPUT will contain "Remote peer disconnected" or "NoReply"
```

## Gotchas
- Do not pass `$$` directly from a root shell when executing via `runuser -u testuser -- pkcheck -p $$ ...`, as polkitd restricts querying processes owned by other users. Instead, wrap the command in a user shell so `$$` refers to the user's process:
  `runuser -u testuser -- bash -c 'pkcheck -p $$ -a org.freedesktop.policykit.exec'`
- If the bug is not present (healthy state), `pkcheck` will exit with code `2` ("Authorization requires authentication and -u wasn't passed") and print `polkit\56result=auth_admin`.
