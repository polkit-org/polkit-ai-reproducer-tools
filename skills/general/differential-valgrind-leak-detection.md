# Component: General / Memory Leaks

## When to use
When reproducing memory leaks that occur inside loops or recurring functions, especially in GLib- or GIO-based C programs where static or one-time initialization allocations can obscure the results or cause false positives.

## Technique
Instead of attempting to run a program for long periods to observe high memory growth, use **Differential Valgrind Heap Profiling**:
1. Design the test program to take the number of loop iterations as a command-line parameter.
2. Run the program under Valgrind twice, once with a small iteration count (e.g., $N_1 = 100$) and once with a larger iteration count (e.g., $N_2 = 500$).
3. Record the "in use at exit" blocks and bytes from the Valgrind `HEAP SUMMARY` section for both runs.
4. Compute the difference: $\Delta_{\text{blocks}} = \text{blocks}(N_2) - \text{blocks}(N_1)$ and $\Delta_{\text{bytes}} = \text{bytes}(N_2) - \text{bytes}(N_1)$.
5. If both differences scale linearly with $\Delta N = N_2 - N_1$, this is a rigorous mathematical confirmation of an active, recurring leak in the loop, completely isolated from any constant static initialization overhead or library-internal caching.

## Recipe
```bash
# Run with 100 and 500 iterations
valgrind --leak-check=full --show-leak-kinds=all --log-file=/tmp/val_100.log ./reproducer 100
valgrind --leak-check=full --show-leak-kinds=all --log-file=/tmp/val_500.log ./reproducer 500

# Parse block counts (assuming the line "in use at exit: X bytes in Y blocks")
blocks_100=$(grep "in use at exit:" /tmp/val_100.log | awk '{print $9}' | tr -d ',')
blocks_500=$(grep "in use at exit:" /tmp/val_500.log | awk '{print $9}' | tr -d ',')

block_diff=$((blocks_500 - blocks_100))
echo "Block leak per iteration: $((block_diff / 400)) blocks"
```

## Gotchas
- **Comma Formatting**: Valgrind heap summaries format large numbers with commas (e.g., `2,756`). Be sure to strip the commas with `tr -d ','` before attempting arithmetic operations in shell scripts.
- **Thread-Local Storage (TLS) / Thread Cleanup**: Some thread-related structures or TLS slots may be allocated asynchronously on the first thread run, but stabilize on subsequent runs. Using sufficiently large $N_1$ and $N_2$ (like 100 and 500) ensures such transient startup issues are fully absorbed and do not pollute the delta.
