# polkitd: D-Bus Interface

## When to use
When reproducing bugs in the polkitd daemon, testing authorization
checks programmatically, or debugging D-Bus communication issues.

## polkitd's D-Bus service
- Bus name: `org.freedesktop.PolicyKit1`
- Object path: `/org/freedesktop/PolicyKit1/Authority`
- Interface: `org.freedesktop.PolicyKit1.Authority`

## Key methods

### CheckAuthorization
Check if a subject is authorized for an action:
```bash
busctl call org.freedesktop.PolicyKit1 \
  /org/freedesktop/PolicyKit1/Authority \
  org.freedesktop.PolicyKit1.Authority \
  CheckAuthorization "(sa{sv})sa{ss}us" \
  "unix-process" 2 "pid" u $$ "start-time" t 0 \
  "org.freedesktop.policykit.exec" 0 "" 1 ""
```

### EnumerateActions
List all registered actions:
```bash
busctl call org.freedesktop.PolicyKit1 \
  /org/freedesktop/PolicyKit1/Authority \
  org.freedesktop.PolicyKit1.Authority \
  EnumerateActions "s" "en"
```

## Watching for signals
```bash
# Watch for authorization changes
dbus-monitor --system \
  "type='signal',sender='org.freedesktop.PolicyKit1'"
```

## Gotchas
- polkitd is D-Bus activated. If it's not running, the first D-Bus call
  to its bus name will start it automatically.
- `CheckAuthorization` with `AllowUserInteraction` flag (value 1) will
  trigger the auth agent. Without it, it just checks cached/implicit auth.
- The `unix-process` subject type requires a valid PID and start-time.
  Use `$$` for current PID, `0` for start-time if unknown.
