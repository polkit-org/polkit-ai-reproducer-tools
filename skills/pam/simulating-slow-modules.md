# PAM: Simulating Slow Authentication Modules

## When to use
When reproducing bugs where authentication hangs or blocks due to slow responses from hardware (fingerprint, smartcard) or network (LDAP, Kerberos).

## Technique
Create a minimal C PAM module that sleeps for a specific duration in `pam_sm_authenticate`.

## Recipe
### 1. Create the module source (`pam_sleep.c`)
\`\`\`c
#include <security/pam_modules.h>
#include <security/pam_ext.h>
#include <unistd.h>

PAM_EXTERN int pam_sm_authenticate(pam_handle_t *pamh, int flags, int argc, const char **argv) {
    pam_info(pamh, "Simulating delay...");
    sleep(10);
    return PAM_AUTH_ERR; // Fail to fall back to next module
}

PAM_EXTERN int pam_sm_setcred(pam_handle_t *pamh, int flags, int argc, const char **argv) {
    return PAM_SUCCESS;
}
\`\`\`

### 2. Compile
\`\`\`bash
gcc -fPIC -shared -o /lib/x86_64-linux-gnu/security/pam_sleep.so pam_sleep.c -lpam
\`\`\`

### 3. Use in PAM stack
Add to `/etc/pam.d/polkit-1` (or relevant service):
\`\`\`
auth [success=done default=ignore] pam_sleep.so
auth include common-auth
\`\`\`

## Gotchas
- The module must be in a directory searched by PAM (e.g., \`/lib/x86_64-linux-gnu/security/\`).
- Use \`[success=done default=ignore]\` to ensure the stack continues to password authentication after the delay.
