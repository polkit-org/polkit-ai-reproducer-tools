# Component: Mocking systemd/logind Session Calls in polkitd

## When to use
When reproducing bugs that involve polkit's session monitor, systemd-logind integrations, seat properties, active/local session status, or JavaScript rule evaluations of `subject.session`/`subject.seat` properties. This is particularly useful in containerized test environments where real logind sessions are absent.

## Technique
To simulate various session and seat topologies (such as systemd user sessions, graphical displays, or active/inactive seats) without relying on real PAM logins, compile a small C library that overrides `libsystemd`'s login functions and preload it into `polkitd` via `LD_PRELOAD`.

Importantly, since `polkitd` is run with `--no-debug` by default (which redirects `stdout`/`stderr` to `/dev/null`), run `polkitd` manually *without* the `--no-debug` option to capture custom `polkit.log()` JS messages printed to `stderr`.

## Recipe
1. Create a mock C source file `mock_systemd.c`:
```c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <errno.h>

int sd_pid_get_session(pid_t pid, char **ret_session) {
    return -ENODATA;
}
int sd_uid_get_display(uid_t uid, char **ret_display) {
    if (ret_display) *ret_display = strdup("session-mock");
    return 0;
}
int sd_session_get_seat(const char *session, char **ret_seat) {
    if (ret_seat) *ret_seat = strdup("seat-mock");
    return 0;
}
```

2. Compile into a shared library:
```bash
gcc -shared -fPIC -o /workspace/mock_systemd.so mock_systemd.c
```

3. Run `polkitd` in the background with `LD_PRELOAD` enabled, avoiding `--no-debug`:
```bash
pkill -f polkitd || true
LD_PRELOAD=/workspace/mock_systemd.so /usr/lib/polkit-1/polkitd > /workspace/polkitd.log 2>&1 &
```

4. Write a custom JS rule under `/etc/polkit-1/rules.d/` logging `subject.session` or other properties, trigger an authorization check (e.g. `pkcheck -p $$ -a ...`), and inspect the log.

## Gotchas
- When a binary drops privileges internally (like `polkitd` dropping root to `polkitd` user), the kernel does *not* clear `LD_PRELOAD` unless an `execve` occurs. Therefore, the preloaded library remains active.
- Always use `strdup` when returning strings (such as in `ret_session` or `ret_seat`) because the caller in polkit will attempt to call `free()` on them.
