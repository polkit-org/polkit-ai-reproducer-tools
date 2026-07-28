# Component: polkitd

## When to use
When reproducing bugs related to user group membership lookup limits or failures (e.g., when a user has a large number of supplementary groups, or when group lookups fail due to a buffer size restriction).

## Technique
Instead of attempting to parse polkitd logs from syslog or journalctl (which can be heavily filtered or inaccessible depending on log-level settings), write a self-validating polkit authorization rule in `/etc/polkit-1/rules.d/` that directly checks the state of the `subject` object in JavaScript, and returns `polkit.Result.YES` when the unexpected state (e.g., empty `subject.groups`) is observed.

This enables a machine-verifiable exit code from a simple, non-interactive `pkcheck` call.

## Recipe

1. Write a `00-test.rules` file to `/etc/polkit-1/rules.d/` that intercepts the test user's actions:
```javascript
polkit.addRule(function(action, subject) {
    if (action.id === "org.freedesktop.policykit.exec" && subject.user === "testuser") {
        if (subject.groups.length === 0) {
            // If the bug exists, groups list is empty.
            // Return YES to signal reproduction/success to pkcheck.
            return polkit.Result.YES;
        } else {
            return polkit.Result.NO;
        }
    }
});
```

2. Run `pkcheck` in the reproducer script:
```bash
pkcheck --action-id org.freedesktop.policykit.exec --process $$
if [ $? -eq 0 ]; then
    echo "Bug reproduced successfully!"
    exit 0
else
    echo "Bug not reproduced."
    exit 1
fi
```

## Gotchas
- When executing the rule, make sure the checked process is actually owned by the test user (e.g. by running `runuser -u testuser -- bash -c 'pkcheck ... --process $$'`). If `pkcheck`'s checked PID is owned by root, the rule won't match `testuser`.
- Some environments filter stdout/stderr of daemon processes, making `polkit.log()` hard to capture, so relying on the `pkcheck` exit code via rules-driven authorization is far more robust.
