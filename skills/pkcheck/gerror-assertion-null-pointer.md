# Component: pkcheck / CLI Tools

## When to use
When reproducing bugs in polkit CLI utilities (like `pkcheck`, `pkexec`, `pkttyagent`) that involve missing arguments leading to assertion failures and crashes.

## Technique
Check if the crash is due to a null-pointer dereference of a `GError` object. In GLib, when function arguments fail a critical assertion (like `g_return_val_if_fail`), the function returns `NULL` or a default value *without* setting the `GError` pointer. If the calling code assumes that a `NULL` return value always means a `GError` is set and dereferences it (e.g., `error->message`), it will segfault.

To verify if this bug still exists in current HEAD:
1. Examine the calling code's error handling.
2. Check if a null check `error ? error->message : "..."` has been introduced.
3. If the bug has been fixed upstream (e.g., in polkit commit `7550b683`), mark the reproduction as unsuccessful (`success: false`).

## Recipe
To inspect the GitHub / GitLab git history for a specific fix or check details of a commit:
```bash
gh api repos/polkit-org/polkit/commits/<commit-hash> --jq '{message: .commit.message, files: [.files[].filename]}'
```

## Gotchas
- Modern GLib / glibc handles formatting a NULL pointer as `(null)` without crashing, but dereferencing `error->message` when `error` is NULL will always segfault.
- Always check the current HEAD source code since many older bugs have already been silently or explicitly patched in the repository.
