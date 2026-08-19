# General: Non-interactive su Logins with systemd-logind Session Registration

## When to use
When reproducing polkit or D-Bus/systemd bugs in containerized environments where:
1. `pkcheck` or other tools require the caller to belong to a registered systemd-logind session (otherwise they fail with errors like `Cannot determine session the caller is in`).
2. Run-of-the-mill `runuser` commands do not trigger the PAM session stack, thus leaving processes without logind sessions.
3. Traditional `su - testuser` commands fail with PAM password checking or capability/namespace restrictions under Docker.

## Technique
By configuring `/etc/pam.d/su` to use `pam_permit.so` specifically for the `auth` and `account` groups, but preserving the standard `session` substacks (such as `system-auth` or `postlogin` which include `pam_systemd.so`), we can allow passwordless, non-interactive `su` logins that successfully register active logind sessions.

## Recipe

1. Write the following custom `/etc/pam.d/su` configuration:
```ini
#%PAM-1.0
auth       sufficient   pam_permit.so
account    sufficient   pam_permit.so
session    include      system-auth
session    include      postlogin
session    optional     pam_xauth.so
```

2. Inside your reproducer script, use the `XDG_SESSION_ID` environment variable to detect if you are running in a session. If not, transparently execute yourself inside `su - testuser`:
```bash
if [ -z "$XDG_SESSION_ID" ]; then
    echo "Not in a logind session. Re-running ourselves inside 'su - testuser'..."
    su - testuser -c "bash /workspace/output/reproducer.sh"
    exit $?
fi

# Actual test steps go here...
```

## Gotchas
- Do not place `session sufficient pam_permit.so` at the top of the session stack because it will succeed immediately and prevent `pam_systemd.so` from running, meaning systemd-logind will never register the session.
- Always verify the login session has been registered using `loginctl list-sessions` or checking for the presence of the `XDG_SESSION_ID` environment variable.
