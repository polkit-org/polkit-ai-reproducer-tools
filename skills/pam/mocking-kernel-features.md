# Component: PAM / Kernel Feature Mocking

## When to use
When reproducing bugs that depend on kernel-specific features or system calls (such as newer socket options like `SO_PEERPIDFD`) which are supported by the host's kernel but need to be disabled/mocked to simulate older systems or specific environments.

## Technique
Use a tiny custom shared library with `LD_PRELOAD` to intercept library calls like `getsockopt` and return specific error codes (such as `ENOPROTOOPT` or `ENODATA`) when called with specific parameters.

For systemd socket-activated services like `polkit-agent-helper-1`, the environment variable `LD_PRELOAD` must be injected into the systemd service context using a drop-in override file, followed by a systemd daemon reload.

## Recipe

### 1. The Interception Shared Library (C)
```c
#define _GNU_SOURCE
#include <stdio.h>
#include <unistd.h>
#include <dlfcn.h>
#include <errno.h>
#include <sys/socket.h>

#ifndef SO_PEERPIDFD
#define SO_PEERPIDFD 77
#endif

typedef int (*orig_getsockopt_type)(int, int, int, void *, socklen_t *);

int getsockopt(int sockfd, int level, int optname, void *optval, socklen_t *optlen) {
    if (level == SOL_SOCKET && optname == SO_PEERPIDFD) {
        errno = ENOPROTOOPT;
        return -1;
    }
    orig_getsockopt_type orig_getsockopt = (orig_getsockopt_type)dlsym(RTLD_NEXT, "getsockopt");
    return orig_getsockopt(sockfd, level, optname, optval, optlen);
}
```
Compile with:
```bash
gcc -shared -fPIC -o /workspace/libfake_getsockopt.so /workspace/libfake_getsockopt.c -ldl
```

### 2. Injecting into Systemd Service
Create the override drop-in:
```bash
mkdir -p /etc/systemd/system/polkit-agent-helper@.service.d/
cat << 'EOF' > /etc/systemd/system/polkit-agent-helper@.service.d/override.conf
[Service]
Environment="LD_PRELOAD=/workspace/libfake_getsockopt.so"
EOF
systemctl daemon-reload
```

## Gotchas
- **TCL/Expect syntax:** If writing expect scripts, avoid using literal square brackets `[...]` in text output strings (such as `[SUCCESS]`) as TCL parses them as command substitutions and fails with `invalid command name`. Use parenthesis `(SUCCESS)` instead.
- **Systemd standard paths:** Systemd overrides must be located in `/etc/systemd/system/` rather than `/usr/lib/systemd/system/`.
- Ensure standard socket services are started (`systemctl start polkit-agent-helper.socket`) so that socket activation is active and triggers the helper.
