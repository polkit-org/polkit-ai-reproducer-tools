# Component: pkexec - Non-ASCII Argument Handling

## When to use
When reproducing bugs related to how `pkexec` handles command-line arguments, especially those containing non-ASCII (UTF-8) characters or specific string lengths.

## Technique
Use `python3` to reliably generate strings with specific byte/character sequences and lengths. This avoids shell-dependent encoding issues. For capturing warnings that appear *before* authentication, use `timeout` with `2>&1` to capture stderr and grep for the expected error message.

## Recipe
```bash
# Generate a 54-character string with UTF-8 characters at specific positions
ARG=$(python3 -c "print('_' * 18 + 'ö' + '_' * 20 + 'é' + '_' * 14)")

# Capture output and check for GLib critical warnings
OUTPUT=$(timeout 5 pkexec /path/to/executable "$ARG" 2>&1)
if echo "$OUTPUT" | grep -q "g_variant_new_string(): requires valid UTF-8"; then
    echo "Bug reproduced"
fi
```

## Gotchas
- `pkexec` often emits warnings to `stderr` before it prompts for a password.
- If you use `expect`, be aware that `exec` within `expect` might have its own encoding issues when dealing with non-ASCII output from subprocesses.
- Some bugs in `pkexec` are sensitive to the total length of the path and arguments.
- The authentication prompt itself might change (e.g., showing `[Invalid UTF-8]`) when this bug is triggered.
