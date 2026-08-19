# Component: polkitd / JS Rules Engine

## When to use
When reproducing bugs involving missing or corrupted JavaScript globals, or forcing failure paths during rules parsing and engine lookups (such as `duk_get_global_string` failures for `Subject`, `Action`, `polkit`, etc.).

## Technique
By creating a `.rules` file loaded with high priority (lexicographically early, e.g. `00-delete-subject.rules`), we can use the JavaScript `delete` keyword on global objects or constructors inside polkit's Duktape context. Since polkitd watches and loads these rules during initialization, this deletes the property from the global object. Any subsequent internal engine lookups (e.g., `push_subject` fetching `"Subject"`) will fail, allowing you to trigger error paths and test robustness or reproduce crashes.

## Recipe
Write a high-priority rule file to `/etc/polkit-1/rules.d/00-delete-subject.rules`:
```javascript
delete Subject;
```

Restart polkit to load and execute the rule:
```bash
systemctl restart polkit
```

Trigger any authorization request as a regular user (which forces polkitd to build the subject/action JS context):
```bash
pkexec echo hello
```

Verify the service status or crash log:
```bash
journalctl -u polkit --since "1 minute ago"
```

## Gotchas
- Deleting essential constructors like `Subject` will cause `polkitd` to fail all subsequent authorization checks, and may crash the daemon if it has unhandled error paths (such as `GError` contract violations).
- Always clean up the temporary rule and restart polkit afterward to restore normal system operations:
  ```bash
  rm -f /etc/polkit-1/rules.d/00-delete-subject.rules
  systemctl restart polkit
  ```
