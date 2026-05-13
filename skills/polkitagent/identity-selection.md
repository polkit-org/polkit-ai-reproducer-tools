# Component: polkitagent

## When to use
When reproducing bugs involving pkttyagent's identity selection prompt (e.g., "Multiple identities can be used for authentication").

## Technique
To trigger the identity selection prompt, you must have multiple users that polkit considers "admin" for the requested action.

1. Identify the admin group (usually `wheel` or `sudo` in `/usr/share/polkit-1/rules.d/50-default.rules`).
2. Add the current user to that group.
3. Create at least one other user and add them to that same group.
4. Set passwords for both users (polkit filters out users without passwords in some configurations).
5. Run `pkexec` or another polkit-aware command.

## Recipe
```bash
# Add current user to wheel
usermod -aG wheel testuser
# Create another admin
useradd adminuser
usermod -aG wheel adminuser
# Set passwords
echo "testuser:testpass" | chpasswd
echo "adminuser:testpass" | chpasswd
```

```tcl
# Expect script to detect the prompt
expect "Multiple identities can be used for authentication:"
```

## Gotchas
- If only one admin user exists, pkttyagent defaults to that user and asks for the password directly (or shows "Authenticating as: ...").
- Tcl/Expect `puts` statements: `[something]` will be interpreted as a command substitution. Escape them as `\[something\]` or use curly braces `{}` if you don't need variable substitution.
- Some polkit versions/configurations might only list users with a valid login shell or home directory.
