# Component: polkitagent

## When to use
When reproducing bugs or testing features involving `pkttyagent` (e.g. testing options, localization, or connection behavior) in non-interactive shell environments, where a direct terminal is not allocated.

## Technique
By default, `pkttyagent` attempts to open the current controlling terminal (`/dev/tty`) to handle password prompts securely. If executed in a pipeline, background process, or simple `runuser` without a TTY, it will fail immediately with:
`Error creating textual authentication agent: Error opening current controlling terminal for the process ('/dev/tty'): No such device or address`

To bypass this restriction and test `pkttyagent` programmatically, use `expect`. Spawning `pkttyagent` inside `expect` automatically allocates a pseudo-terminal (pty) for the process, allowing it to initialize and communicate with `polkitd`.

## Recipe
```bash
# Execute within a bash script running as a test user
OUTPUT=$(expect -c '
set timeout 5
spawn pkttyagent --system-bus-name :1.999
expect {
    "Only unix-process and unix-session subjects" {
        exit 0
    }
    "Error creating textual authentication agent" {
        exit 10
    }
    eof {
        exit 11
    }
    timeout {
        exit 12
    }
}
')
```

## Gotchas
- When executing as another user (e.g., using `runuser -u testuser --`), `expect` must be run as that target user so the allocated TTY has the correct permissions.
- Make sure to set a reasonable `set timeout` inside `expect` so the script does not block indefinitely if the agent does not exit or fail as expected.
