# General: D-Bus Introspection for polkit

## When to use
When debugging polkit authorization failures, verifying polkitd is running,
or tracing D-Bus communication between polkit components.

## Checking polkitd is registered on the bus
```bash
busctl list | grep PolicyKit
# Expected: org.freedesktop.PolicyKit1

busctl tree org.freedesktop.PolicyKit1
# Shows the object tree
```

## Checking authorization for an action
```bash
# Check if testuser is authorized for an action
pkcheck --action-id org.freedesktop.policykit.exec \
        --process $$ --allow-user-interaction 2>&1

# List all registered actions
pkaction
pkaction --verbose --action-id org.freedesktop.policykit.exec
```

## Tracing D-Bus messages
```bash
# Watch all polkit-related D-Bus traffic
dbus-monitor --system "interface='org.freedesktop.PolicyKit1.Authority'" &

# Then trigger the action in another terminal
pkexec whoami
```

## Using busctl for direct D-Bus calls
```bash
# Call CheckAuthorization directly
busctl call org.freedesktop.PolicyKit1 \
  /org/freedesktop/PolicyKit1/Authority \
  org.freedesktop.PolicyKit1.Authority \
  CheckAuthorization \
  "(sa{sv})sa{ss}us" \
  "unix-process" 2 "pid" u $$ "start-time" t 0 \
  "org.freedesktop.policykit.exec" \
  0 \
  "" \
  1 ""
```

## Gotchas
- `dbus-send` uses a different type syntax than `busctl`. Prefer `busctl`
  for complex calls.
- polkitd is D-Bus activated — the first D-Bus call to PolicyKit1 will
  start it if it's not already running.
