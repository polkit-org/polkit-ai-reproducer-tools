# polkitd: JavaScript Authorization Rules

## When to use
When reproducing bugs related to polkit authorization decisions, custom
rules, or the Duktape JavaScript engine.

## How rules work
polkitd loads JavaScript rules from two directories (in order):
1. `/usr/share/polkit-1/rules.d/` — vendor/package rules
2. `/etc/polkit-1/rules.d/` — local admin rules

Files are loaded in lexicographic order. Use numeric prefixes to control
priority: `10-testing.rules` runs before `50-default.rules`.

## Writing a test rule
```bash
cat > /etc/polkit-1/rules.d/10-test.rules << 'EOF'
polkit.addRule(function(action, subject) {
    // Allow testuser to run any command via pkexec without auth
    if (action.id == "org.freedesktop.policykit.exec" &&
        subject.user == "testuser") {
        return polkit.Result.YES;
    }
});
EOF
```

## Available objects in rules

### `action` object
- `action.id` — action identifier (e.g., "org.freedesktop.policykit.exec")
- `action.lookup("key")` — get action annotation

### `subject` object
- `subject.user` — username of the requesting user
- `subject.groups` — array of group names
- `subject.seat` — seat name (from logind)
- `subject.session` — session ID
- `subject.active` — true if session is active
- `subject.local` — true if session is local
- `subject.pid` — process ID
- `subject.isInGroup("groupname")` — check group membership

### Return values
- `polkit.Result.YES` — allow without auth
- `polkit.Result.AUTH_SELF` — require user's own password
- `polkit.Result.AUTH_ADMIN` — require admin password
- `polkit.Result.NO` — deny

## Reloading rules
```bash
# polkitd watches the rules directories with inotify
# Just writing a file triggers a reload. To force:
systemctl restart polkit
```

## Gotchas
- Rules files MUST have `.rules` extension.
- Syntax errors in a rules file cause polkitd to skip it silently.
  Check `journalctl -u polkit` for error messages.
- The Duktape JS engine is ES5 only — no arrow functions, no `let`/`const`,
  no template literals. Use `var` and `function(){}`.
- `subject.active` and `subject.local` require systemd-logind. In containers
  without logind, these may always be false.
