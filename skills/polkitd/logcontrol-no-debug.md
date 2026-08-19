# Component: LogControl1 and Logging Behaviors

## When to use
When reproducing bugs related to polkit logging levels, syslog outputs, LogControl1 properties, or testing `--no-debug` option configurations.

## Technique
Normally, `polkitd` redirects standard output and standard error to `/dev/null` if the `--no-debug` option is passed in its startup command (which is default in systemd services).
To see debug or custom log levels, or standard error outputs:
1. Create a systemd service override:
   ```ini
   [Service]
   ExecStart=
   ExecStart=/usr/lib/polkit-1/polkitd --log-level=notice
   ```
2. Reload and restart polkit:
   ```bash
   systemctl daemon-reload
   systemctl restart polkit
   ```
3. Use D-Bus to query/set properties of `/org/freedesktop/LogControl1`:
   ```bash
   busctl get-property org.freedesktop.PolicyKit1 /org/freedesktop/LogControl1 org.freedesktop.LogControl1 LogLevel
   busctl set-property org.freedesktop.PolicyKit1 /org/freedesktop/LogControl1 org.freedesktop.LogControl1 LogLevel s "crit"
   ```

## Recipe
To trigger common polkitd standard error/assertion messages (such as `Error performing authentication`), you can use `expect` to abruptly spawn and close a `pkexec` process during the authentication phase:
```bash
expect -c 'spawn pkexec whoami; expect "Authenticating as: Super User (root)"; close'
```

## Gotchas
- LogControl1's `LogLevel` property only controls log filtering within the `polkit_backend_authority_log()` function.
- Many warning/error logs in the codebase use direct standard error outputs (like `g_printerr` or GLib assertions/warnings) rather than the structured logging helper.
- These direct `g_printerr` and GLib messages are NOT affected by LogControl1 settings and will continue to be emitted when `--no-debug` is removed.
