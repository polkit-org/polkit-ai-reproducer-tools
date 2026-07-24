# Component: PAM Bypassing via pam_permit.so

## When to use
When reproducing polkit authorization bugs in a containerized, sandboxed, or restricted environment where PAM password check helpers (like `unix_chkpwd`) fail due to lack of SETUID capability, AppArmor restrictions, or `NoNewPrivileges=yes` on the systemd service.

## Technique
By overriding `/etc/pam.d/polkit-1` to use `pam_permit.so` for all PAM management groups (`auth`, `account`, `password`, `session`), we can make PAM authentication succeed instantly and unconditionally. This eliminates the need for valid user password entry and prevents permission denials from underlying password-checking binaries, allowing the test to focus solely on the Polkit daemon and helper protocol interactions.

## Recipe
Write the following content to `/etc/pam.d/polkit-1`:
```ini
#%PAM-1.0
auth       sufficient   pam_permit.so
account    sufficient   pam_permit.so
password   sufficient   pam_permit.so
session    sufficient   pam_permit.so
```

## Gotchas
- Ensure `pam_permit.so` is available on the system (usually located in `/lib64/security/` or `/lib/security/`).
- Bypassing PAM means `pkexec` will not prompt for a password at all, or if it does, any input (including empty) will be accepted. Expect scripts should be designed to handle both immediate success/failure and optional password prompting.
- Always run `systemctl daemon-reload` and restart related services if needed to ensure the PAM configuration is reloaded.
