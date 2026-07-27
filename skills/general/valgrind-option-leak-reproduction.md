# Component: Valgrind Option Leak Reproduction

## When to use
When reproducing memory leaks in CLI utilities' option parsing (e.g., duplicate/overwritten arguments), especially when binaries are run without authentication or long execution times.

## Technique
1. **Compile with Debug Symbols**: Compile the source with debug symbols (e.g., using Meson/Ninja debug build) to obtain exact file and line number mappings in stack traces.
2. **Stable/Durable PIDs**: Use PID 1 (systemd) as a guaranteed, existing process subject to prevent needing to spawn background processes or lookup dynamic PIDs.
3. **Valgrind Logging**: Pipe Valgrind output to a file and use `grep` to verify the presence of exact line allocations in the call stack.

## Recipe
```bash
# Compile
meson setup build
ninja -C build

# Trace action_id leaks
valgrind --leak-check=full --log-file=/tmp/valgrind_action.log ./build/src/programs/pkcheck -p 1 -a foo -a bar

# Check log for leak at the parsing line
grep -q "pkcheck.c:446" /tmp/valgrind_action.log
```

## Gotchas
- **Permissions with Shared Temp Files**: When running the reproducer as a non-root user (e.g., `testuser`), make sure the temporary log filenames (e.g., `/tmp/valgrind_action_testuser.log`) do not collide with files previously created by `root`, otherwise you will hit `Operation not permitted` errors.
- **False Positives with Exit Codes**: Do not rely purely on Valgrind's `--error-exitcode` because general library initialization leaks (e.g., from GLib/GIO) may also trigger it. Grepping the log for specific lines/filenames in the stack trace is significantly more precise and reliable.
