# Component: General / UID Simulation

## When to use
When reproducing bugs involving extreme, negative, or large UIDs/GIDs (e.g., UIDs greater than $2^{31}$) where switching to the user via standard tools like `su` or `sudo` fails due to sandboxing, AppArmor, or PAM authentication constraints.

## Technique
Instead of attempting to authenticate or spawn interactive sessions for users with large/irregular UIDs (which often trigger AppArmor denials in `unix_chkpwd` or PAM errors), compile a lightweight setuid-root C wrapper.
The C wrapper runs as root (using its setuid privilege), calls `setreuid()` to transition both real and effective UIDs to the target large/extreme UID, and then `exec`s the target program.
Since this bypasses the standard authentication/account management stack, it functions perfectly under sandboxed/containerized environments.

## Recipe
1. Write a lightweight helper program (e.g. `setuid_helper.c`):
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    uid_t target_uid = 3333333333U; // Example large UID (> 2^31)
    if (setreuid(target_uid, target_uid) != 0) {
        perror("setreuid failed");
        return 1;
    }
    execl("/usr/bin/pkexec", "pkexec", "whoami", NULL);
    perror("execl failed");
    return 1;
}
```

2. Compile and configure it as setuid-root:
```bash
gcc -O2 setuid_helper.c -o setuid_helper
chown root:root setuid_helper
chmod 4755 setuid_helper
```

3. Run the helper as an unprivileged user (e.g. `testuser`):
```bash
runuser -u testuser -- ./setuid_helper
```

## Gotchas
- **`nosuid` Mount Option:** Ensure that the helper binary is not compiled/located on a partition mounted with the `nosuid` option (such as `/tmp` on many systems). If compiled in a `nosuid` directory, `setreuid` will fail with `Operation not permitted` when run as a non-root user. Compile and run it under `/workspace` or `/usr/bin` instead.
- **Process Subject Resolution:** Be aware that some polkit/systemd checks may query other system parameters (like systemd-logind session or cgroups). While this wrapper changes the real and effective UID, it retains the parent process' cgroups/session unless additional setup is done.
