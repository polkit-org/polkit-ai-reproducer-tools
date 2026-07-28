# Component: PAM Session Conversations

## When to use
When reproducing bugs involving PAM conversation handling during non-auth phases of the PAM stack (such as `pam_open_session`, `pam_acct_mgmt`, etc.), where an application's conversation handler might crash, block, or handle responses incorrectly.

## Technique
Create a minimal C PAM module that triggers an informational or interactive conversation message (using standard PAM helper functions like `pam_info`) during the target phase. Compile it and configure `/etc/pam.d/<service>` to use it in that phase.

## Recipe
### 1. Create the module source (`pam_test_conv.c`)
```c
#include <stddef.h>
#include <security/pam_modules.h>
#include <security/pam_ext.h>

PAM_EXTERN int pam_sm_authenticate(pam_handle_t *pamh, int flags, int argc, const char **argv) {
    return PAM_SUCCESS;
}

PAM_EXTERN int pam_sm_setcred(pam_handle_t *pamh, int flags, int argc, const char **argv) {
    return PAM_SUCCESS;
}

PAM_EXTERN int pam_sm_acct_mgmt(pam_handle_t *pamh, int flags, int argc, const char **argv) {
    return PAM_SUCCESS;
}

PAM_EXTERN int pam_sm_open_session(pam_handle_t *pamh, int flags, int argc, const char **argv) {
    // Send an informational message via conversation
    pam_info(pamh, "This is a custom conversation message from the PAM session module!");
    return PAM_SUCCESS;
}

PAM_EXTERN int pam_sm_close_session(pam_handle_t *pamh, int flags, int argc, const char **argv) {
    return PAM_SUCCESS;
}
```

### 2. Compile and install
```bash
gcc -fPIC -shared -o /lib/x86_64-linux-gnu/security/pam_test_conv.so pam_test_conv.c -lpam
```

### 3. Use in PAM stack
Specify the module under the corresponding phase (e.g. `session`) in `/etc/pam.d/polkit-1`:
```ini
#%PAM-1.0
auth       sufficient   pam_permit.so
account    sufficient   pam_permit.so
password   sufficient   pam_permit.so
session    required     pam_test_conv.so
session    sufficient   pam_permit.so
```

## Gotchas
- Always include `<stddef.h>` before the PAM headers to avoid compilation issues where `NULL` is not defined.
- To execute TTY-dependent commands (such as `pkexec`) that normally block or check for a controlling terminal, run them wrapped in the `script` utility (e.g. `script -q -c "pkexec whoami" /dev/null`) to emulate a pseudo-TTY.
