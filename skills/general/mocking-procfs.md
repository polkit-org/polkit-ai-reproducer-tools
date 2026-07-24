# Component: General - Mocking Procfs files

## When to use
When reproducing bugs that involve parsing files in `/proc` (such as `/proc/PID/stat`, `/proc/PID/status`, `/proc/PID/cmdline`) where we need to control the contents of these files for error-path testing or mocking.

## Technique
Use a Linux bind-mount to mount a custom text file over the real `/proc/PID/stat` (or similar) of a live background process.

## Recipe
1. Start a persistent background process (e.g. `sleep`) as the desired target user:
   ```bash
   sleep 3600 >/dev/null 2>&1 &
   MOCK_PID=$!
   disown $MOCK_PID
   ```
2. Create your mocked file:
   ```bash
   echo "mock content" > /tmp/mock_file
   ```
3. Bind-mount the mock file over the procfs entry (as root):
   ```bash
   mount --bind /tmp/mock_file /proc/$MOCK_PID/stat
   ```
4. Run your test program on `$MOCK_PID`.
5. Clean up by simply killing the background process:
   ```bash
   kill -9 $MOCK_PID
   ```
   The kernel will automatically remove the `/proc/$MOCK_PID/` directory and safely unmount the bind mount.

## Gotchas
- Creating a bind mount requires root privileges.
- Always disown background processes and redirect their stdout/stderr to `/dev/null` to prevent the executing shell from hanging/blocking.
