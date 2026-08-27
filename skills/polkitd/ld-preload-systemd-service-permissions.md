# Component: polkitd LD_PRELOAD in Systemd Environment

## When to use
When using `LD_PRELOAD` to mock logind or other external libraries for `polkitd` running inside a systemd container.

## Technique
Since `polkit.service` has `ProtectSystem=strict` and runs under a restricted environment with `User=polkitd` and `PrivateTmp=yes`, mock libraries must be:
1. Located in globally accessible directories like `/usr/lib/` or `/usr/lib64/` (not restricted by `PrivateTmp` or `/workspace` access policies).
2. Given proper world-readable and world-executable permissions (`chmod 755`).
3. Registered via systemd drop-in configurations (`/etc/systemd/system/polkit.service.d/override.conf`) rather than standard shell command line preloads.

## Recipe
Create a systemd override directory and file (as root):
```bash
mkdir -p /etc/systemd/system/polkit.service.d
cat << 'EOF' > /etc/systemd/system/polkit.service.d/override.conf
[Service]
Environment=LD_PRELOAD=/usr/lib/mock_systemd.so
EOF

systemctl daemon-reload
systemctl restart polkit
```

Ensure permissions on the mock library are permissive:
```bash
cp mock_systemd.so /usr/lib/mock_systemd.so
chmod 755 /usr/lib/mock_systemd.so
```

## Gotchas
- If the mock library is compiled to `/workspace/` or `/tmp/`, the service may fail to load it because of systemd's sandbox protections (`PrivateTmp=yes` or namespace isolations).
- Permissions on standard workspace paths might prevent the `polkitd` user from reading the library, causing polkitd to fail to start entirely (check `journalctl -u polkit` for loading/permission errors).
