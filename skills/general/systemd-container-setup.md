# General: systemd Container Environment

## When to use
Every reproducer runs in this environment. Understand it before debugging.

## Environment details
- systemd is PID 1 — it manages all services.
- dbus-daemon starts automatically via systemd.
- polkitd starts automatically as a D-Bus activated service.
- systemd-logind tracks user sessions (needed for polkit auth).

## Verifying services are running
```bash
systemctl is-system-running         # should be "running" or "degraded"
systemctl status polkit             # polkitd status
systemctl status dbus               # dbus-daemon status
systemctl status systemd-logind     # logind status
busctl list | grep PolicyKit        # verify polkitd is on the bus
```

## Common issues
- If polkitd is not running, check `journalctl -u polkit` for startup errors.
- If auth prompts never appear, logind may not have a session for testuser.
  Check with `loginctl list-sessions`.
- "degraded" system state is fine — it usually means a non-critical unit
  failed (like cloud-init or network). polkit works regardless.

## Gotchas
- Do NOT run `dbus-daemon --system` manually — it conflicts with systemd's.
- Do NOT run `polkitd` manually — it's D-Bus activated.
- The container runs `--privileged` with host cgroups mounted. This is
  required for systemd to function.
