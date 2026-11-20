# Component: Testing unix-session Subjects in polkitd

## When to use
When testing or reproducing bugs involving `unix-session` subjects in `polkitd` (such as authorization checks, JavaScript authority conversions in `push_subject()`, or systemd-logind session lookups).

## Technique
Instead of mocking `libsystemd` with `LD_PRELOAD`, containers with systemd as PID 1 can register real, persistent logind sessions using `systemd-run` and `su -l <user>`. `busctl call` can then be used to invoke `CheckAuthorization` directly using the D-Bus variant signature for `PolkitSubject`.

## Recipe
1. In `prepare_env.sh` (as root), register a persistent logind session for `testuser`:
```bash
usermod -aG systemd-journal testuser

if ! loginctl list-sessions --no-legend | grep -q "testuser"; then
    systemd-run --unit=testuser-session su -l testuser -c "sleep infinity"
    for _ in {1..30}; do
        loginctl list-sessions --no-legend | grep -q "testuser" && break
        sleep 0.1
    done
fi
```

2. In `reproducer.sh` (as `testuser`), query the session ID and invoke `CheckAuthorization`:
```bash
SESSION_ID=$(loginctl list-sessions --no-legend | awk '$3=="testuser" {print $1; exit}')

busctl call org.freedesktop.PolicyKit1 \
    /org/freedesktop/PolicyKit1/Authority \
    org.freedesktop.PolicyKit1.Authority \
    CheckAuthorization \
    '(sa{sv})sa{ss}us' 'unix-session' 1 'session-id' s "$SESSION_ID" \
    'org.freedesktop.login1.power-off' 0 0 ''
```

## Gotchas
- **Session must exist in logind:** polkitd calls `sd_session_get_uid()` on the session ID. If the session does not exist, it aborts early with `Error getting uid for session` (`POLKIT_ERROR_FAILED`) without reaching the JavaScript authority.
- **Session must belong to a non-root user:** Root-owned subjects are short-circuited by `identity_is_root_user()` before the JavaScript authority (`push_subject()`) is called.
- **Caller trust and details:** Non-root callers can only query subjects belonging to their own identity, and must pass an empty `details` dictionary (`{}`). Passing details triggers an early `POLKIT_ERROR_NOT_AUTHORIZED` check.
