# pkexec: Interactive Authentication with Expect

## When to use
When reproducing bugs that involve pkexec's password prompt, pkttyagent
interaction, or the authentication dialogue flow.

## Why expect is needed
pkexec requires a TTY and uses pkttyagent for interactive password prompts.
Non-interactive scripts can't type passwords. Use `expect` to script the
interaction.

## Basic pattern
```bash
#!/usr/bin/expect -f
set timeout 10

# Spawn pkexec with the command to run as root
spawn pkexec whoami

expect {
    "Password:" {
        send "testpass\r"
        expect eof
    }
    timeout {
        puts "TIMEOUT: no password prompt"
        exit 2
    }
    eof {
        puts "EOF: pkexec exited without prompting"
        exit 3
    }
}

# Check exit code
catch wait result
set exit_code [lindex $result 3]
exit $exit_code
```

## Setting up the test user password
```bash
# The container creates testuser with password "testpass"
# If you need to reset it:
echo "testuser:testpass" | chpasswd
```

## Running as testuser
```bash
# Always run pkexec as testuser, not root
runuser -u testuser -- expect /path/to/script.exp
```

## Checking what pkexec returns
```bash
# 0   — command executed successfully
# 126 — not authorized or auth dismissed
# 127 — auth failed (wrong password or PAM error)
```

## Gotchas
- pkexec needs a **real TTY**. Docker exec must use `-t` flag.
- pkttyagent registers as the auth agent automatically when pkexec runs
  from a TTY. If no agent is registered, pkexec fails immediately.
- The password prompt text varies by locale. In English it's "Password:",
  in German "Passwort:", etc.
- pkexec shows "Authentication is needed to run '...' as the super user"
  before the password prompt. Don't confuse this with the prompt itself.
- If using `spawn` in an expect script inside a shell heredoc, use
  `<< 'EOF'` (single-quoted) to prevent shell expansion of `$` and `\`.
