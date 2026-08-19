# Component: polkitagent

## When to use
When reproducing password typeahead leak bugs (such as polkit issue #545), where unsubmitted terminal inputs are leaked back to the shell after a privileged prompt exits or times out.

## Technique
To machine-verify terminal input leaks:
1. Wrap the password prompting tool (e.g. `pkexec`) with `timeout <seconds>` to force cancellation/exit.
2. Use `expect` to spawn the shell, run the timed-out command, wait for the `Password:` prompt, and send characters *without* a trailing `\r` (no Return key pressed).
3. Wait for the command to timeout and the shell prompt to return.
4. Send a single `\r` (Return key) to submit the terminal's canonical input buffer.
5. Match the shell error output (`<typed-characters>: command not found`) to prove the unsubmitted buffer was leaked back to the shell.

To bypass complex identity selection prompts or multiple-choice questions on old polkit versions (which can also fail with `No session for cookie` inside container sessions lacking systemd-logind):
- Write a temporary polkit JavaScript rule forcing `polkit.Result.AUTH_SELF` for the test user. This makes the prompt go directly to `Password:` for the target user.

## Recipe
### 1. Polkit Rule Setup (e.g., in `prepare_env.sh`):
```bash
cat > /etc/polkit-1/rules.d/10-test-reproducer.rules << 'EOF'
polkit.addRule(function(action, subject) {
    if (action.id == "org.freedesktop.policykit.exec" && subject.user == "testuser") {
        return polkit.Result.AUTH_SELF;
    }
});
EOF
```

### 2. Expect Orchestration:
```expect
spawn bash
expect -re "testuser.* "

# Run pkexec with a 3-second timeout
send "timeout 3 pkexec whoami\r"

expect {
    "Password:" {
        # Send secret without \r
        send "LeakedSecretPassword"
    }
}

# Wait for timeout to kill pkexec and shell prompt to return
expect -re "testuser.* " {
    # Send return to submit the leaked buffer to the shell
    send "\r"
}

expect {
    "LeakedSecretPassword: command not found" {
        puts "BUG REPRODUCED!"
        exit 0
    }
}
```

## Gotchas
- When writing expect scripts in shell wrappers, be careful with brackets `[` and `]`. If enclosed in double quotes, they are treated as TCL command invocations and will fail with `invalid command name` unless escaped as `\\[` and `\\]`, or avoided entirely (e.g. using regular expression patterns like `(testuser|root).* `).
- Do not add `testuser` to the `sudo` group if testing `AUTH_SELF` on older polkit packages, as multiple identities (the user + other admins) will still be prompted unless `AUTH_SELF` is explicitly returned.
