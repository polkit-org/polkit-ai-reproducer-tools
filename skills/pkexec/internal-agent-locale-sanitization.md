# Component: pkexec

## When to use
When reproducing bugs involving localization, translations, or locale handling inside `pkexec` (especially when running its internal/fallback textual authentication agent).

## Technique
Understand that `pkexec` is a setuid tool and explicitly sanitizes/nukes its environment using `clearenv()` early in its main execution block. 
Because of this:
1. Any internal authentication agent spawned/registered by `pkexec` (e.g. `polkit_agent_text_listener`) will execute in a cleared environment.
2. When the agent registers itself via `polkit_agent_listener_register`, the underlying registration logic (`server_register()` in `polkitagentlistener.c`) attempts to retrieve the locale using `g_getenv ("LANG")`.
3. Since the environment has already been cleared, `g_getenv ("LANG")` returns `NULL` and defaults the registered locale to `"en_US.UTF-8"`.
4. As a consequence, `polkitd` receives an `"en_US.UTF-8"` locale request and translates the authentication message into English rather than the user's intended locale.

To test true localized agent registration, a separate external agent (like `pkttyagent` running in a separate process that has NOT cleared its environment) should be used, or the binary needs to be patched/built with environment variables preserved during agent registration.

## Recipe
To reproduce the environment sanitization locale bug:
1. Compile the target language's translation:
   ```bash
   msgfmt /workspace/polkit-src/po/hu.po -o /usr/share/locale/hu/LC_MESSAGES/polkit-1.mo
   ```
2. Run `pkexec` with the target locale set:
   ```bash
   export LANG=hu_HU.utf8
   export LC_MESSAGES=hu_HU.utf8
   pkexec ls
   ```
3. Even though the locale environment variables are set and the `.mo` file exists, the prompt remains in English.

## Gotchas
- The environment variable backup list (`environment_variables_to_save`) inside `pkexec.c` saves `LANG` and `LC_MESSAGES`, but these are only restored and set *after* successful authorization/authentication has already completed, which is after the agent is registered and the prompt has been shown.
