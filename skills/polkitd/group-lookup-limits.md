# Component: polkitd

## When to use
When reproducing bugs related to user group membership lookups or JavaScript rules that check groups.

## Technique
To test if polkit correctly handles users with a large number of groups (e.g., in LDAP environments), you can programmatically create many groups and add a test user to them.

```bash
# Create 600 groups
for i in $(seq 1 600); do
    groupadd "testgroup$i"
done

# Add user to all 600 groups (comma-separated list is faster)
GROUP_LIST=$(seq -s, 1 600 | sed 's/\([0-9]\+\)/testgroup\1/g')
usermod -G "testuser,$GROUP_LIST" testuser
```

Then use a polkit rule to verify membership:
```javascript
polkit.addRule(function(action, subject) {
    if (subject.isInGroup("testgroup600")) {
        return polkit.Result.YES;
    }
});
```

## Gotchas
- `pkcheck` output escapes dots as `\56`. For example, `polkit.result=yes` is printed as `polkit\56result=yes`. Use robust grep patterns like `grep "result=yes"`.
- Some versions of polkit have a hardcoded limit of 512 groups in `polkitbackendduktapeauthority.c`.
- `getgrouplist()` returns -1 if the buffer is too small, and polkit must be checked to see if it correctly reallocates.
- Rules are loaded in lexicographical order from `/etc/polkit-1/rules.d/`. Use a prefix like `00-` to ensure your rule runs first.
