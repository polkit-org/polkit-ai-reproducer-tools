# General: Locale and Localization Testing

## When to use
When reproducing bugs related to locale handling, translated messages,
LC_MESSAGES vs LANG behavior, or character encoding issues.

## Key environment variables
```
LANG          — fallback locale for all categories
LC_MESSAGES   — locale for user-facing messages (overrides LANG)
LC_ALL        — overrides everything (rarely set in practice)
```

polkit and pkttyagent should respect `LC_MESSAGES` for prompt text.
If they use `LANG` instead, that's a bug.

## Setting up locales in the container

### Fedora
```bash
# Install specific language packs
dnf install -y glibc-langpack-de glibc-langpack-fr glibc-langpack-en

# Verify
locale -a | grep de_DE
```

### Ubuntu/Debian
```bash
apt-get install -y locales
sed -i 's/# de_DE.UTF-8/de_DE.UTF-8/' /etc/locale.gen
locale-gen
```

## Testing locale behavior with pkexec
```bash
# Set LANG to German, LC_MESSAGES to English
export LANG=de_DE.UTF-8
export LC_MESSAGES=en_US.UTF-8

# If pkexec/pkttyagent shows "Passwort:" instead of "Password:",
# it's incorrectly using LANG instead of LC_MESSAGES
```

## Using expect to check prompt language
```bash
export LANG=de_DE.UTF-8
export LC_MESSAGES=en_US.UTF-8

expect << 'EOF'
set timeout 5
spawn pkexec whoami
expect {
  "Passwort:" { puts "BUG: used LANG instead of LC_MESSAGES"; exit 0 }
  "Password:" { puts "CORRECT: used LC_MESSAGES"; exit 1 }
  timeout     { puts "TIMEOUT: no prompt appeared"; exit 2 }
  eof         { puts "EOF: pkexec exited without prompt"; exit 3 }
}
EOF
```

## Gotchas
- Locale packages must be installed BEFORE testing. Missing locales
  silently fall back to C/POSIX.
- `locale -a` lists available locales. If your target locale isn't
  listed, the test is invalid.
- pkexec sanitizes the environment — some locale variables may be
  stripped. Check with `pkexec env | grep LC`.
