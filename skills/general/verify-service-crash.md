# General: Verifying Service Crashes with journalctl

## When to use
When reproducing bugs that cause a system daemon (like `polkitd`) to crash (SIGSEGV, SIGABRT).

## Technique
Instead of just relying on the exit code of the client tool (which might fail for many reasons), use `journalctl` to confirm that the service itself crashed with a specific signal.

To allow a non-root test user to check the logs, add them to the `systemd-journal` group in the preparation phase.

## Recipe
In `prepare_env.sh` (as root):
```bash
usermod -aG systemd-journal testuser
```

In `reproducer.sh` (as testuser):
```bash
START_TIME=$(date '+%Y-%m-%d %H:%M:%S')
# ... trigger the crash ...
sleep 1
if journalctl -u <service_name> --since "$START_TIME" | grep -q "status=11/SEGV"; then
    echo "Service crashed with SIGSEGV"
    exit 0
fi
```

## Gotchas
- Systemd might restart the service automatically.
- `journalctl` might have a slight delay in logging; a short `sleep` is recommended.
- If the container doesn't use `systemd`, this technique won't work.
