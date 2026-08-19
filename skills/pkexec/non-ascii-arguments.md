# Component: pkexec - Non-ASCII Argument Handling

## When to use
When reproducing bugs related to how `pkexec` handles command-line arguments, especially those containing non-ASCII (UTF-8) characters or specific string lengths that trigger truncation or slicing.

## Technique
Use `python3` with raw sys stdout buffer writing to reliably generate strings with specific byte/character sequences and lengths, avoiding shell-dependent and locale-dependent encoding issues (like `UnicodeEncodeError` in POSIX locale).
For capturing warnings that appear before/during authentication check, use `timeout` with `2>&1` to capture stderr and grep for the expected error message.

## Recipe
```bash
# Generate a 54-character string with UTF-8 characters at specific positions
ARG=$(python3 -c "import sys; sys.stdout.buffer.write(b'_' * 18 + 'ö'.encode('utf-8') + b'_' * 20 + 'é'.encode('utf-8') + b'_' * 14)")

# Capture output and check for GLib critical warnings
OUTPUT=$(timeout 5 pkexec /path/to/executable "$ARG" 2>&1)
if echo "$OUTPUT" | grep -q "g_variant_new_string(): requires valid UTF-8"; then
    echo "Bug reproduced"
fi
```

## Gotchas
- `pkexec` often emits warnings to `stderr` before it prompts for a password.
- If the command line is longer than 80 characters, `pkexec` truncates it using byte-level slicing (`g_stpcpy(g_stpcpy(cmdline_short + 38, " ... "), command_line + strlen(command_line) - 37)`).
- This byte-level truncation can split multi-byte UTF-8 sequences, producing invalid UTF-8 strings.
- Always use `sys.stdout.buffer.write()` instead of standard `print()` in python generators to prevent Python from throwing `UnicodeEncodeError` in environments set up with ASCII/POSIX locale.
