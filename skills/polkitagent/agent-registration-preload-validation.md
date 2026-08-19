# Component: polkitagent / polkitd

## When to use
When reproducing bugs related to environment sanitization, validation, or privilege boundary checks during authentication agent registration (such as checking for `LD_PRELOAD` or other dangerous environment variables).

## Technique
To verify if `polkitd` securely rejects agent registration when the agent runs with dangerous environment variables (like `LD_PRELOAD`):

1. Write a minimal C preloader library with a constructor (`__attribute__((constructor))`) that writes to a temporary flag file when loaded.
2. Compile the preloader library as a shared object (`-shared -fPIC`).
3. Run the agent (e.g. `pkttyagent` or a custom client) under a non-root test user, passing `LD_PRELOAD=/path/to/preload.so`.
4. Check if registration succeeds or fails. If registration succeeds and the flag file exists, the daemon failed to reject registration of a tainted agent.

## Recipe

**Preloader Source (`preload.c`):**
```c
#define _GNU_SOURCE
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>

__attribute__((constructor))
static void init(void) {
    int fd = open("/tmp/preload_active", O_WRONLY | O_CREAT | O_TRUNC, 0666);
    if (fd >= 0) {
        write(fd, "1\n", 2);
        close(fd);
    }
}
```

**Compilation:**
```bash
gcc -shared -fPIC -o /tmp/preload.so preload.c
chmod 755 /tmp/preload.so
```

**Testing with Expect:**
```tcl
spawn env LD_PRELOAD=/tmp/preload.so pkttyagent
expect {
    "Error registering authentication agent" {
        # Success: Registration rejected
        exit 0
    }
    timeout {
        # Failure: Daemon accepted registration
        exit 1
    }
}
```

## Gotchas
- **Tcl/Expect Bracket Substitution:** Inside `expect` commands or `send_user` calls, do not use `[+]` or `[-]`. Tcl treats brackets `[...]` as command substitutions. Use plain symbols (e.g., `+` or `-`) or escape them as `\[+\]` and `\[-\]`.
- **Permissions:** Ensure the compiled `.so` file has appropriate read and execute permissions (`chmod 755`) so the non-root test user can load it.
- **PTY Allocation:** `pkttyagent` requires a controlling terminal (`/dev/tty`). Spawning it inside `expect` automatically allocates a pseudo-terminal (PTY) so it doesn't fail with terminal initialization errors.
