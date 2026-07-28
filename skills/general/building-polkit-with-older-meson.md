# Component: General / Build

## When to use
When reproducing polkit bugs on environments (like Ubuntu 24.04 containers) where the packaged `meson` version is older than `1.4.0` (e.g. `1.3.2`), but the polkit repository's `meson.build` requires `>= 1.4.0`.

## Technique
Downgrade the required Meson version in `meson.build` and replace any newer Meson-specific API calls with older, fully-compatible equivalents.

Specifically:
1. Change the project's `meson_version` requirement from `>= 1.4.0` to `>= 1.3.2`.
2. Replace `.full_path()` calls on configure-file objects with string-interpolated absolute paths. In meson >= 1.4.0, a configure-file target is a `File` object which supports `.full_path()`. In earlier versions, this method is missing, but the path can be constructed using `meson.current_build_dir() / 'config.h'`.

## Recipe
Before running `meson setup build`, apply these modifications to `meson.build`:
```bash
# Allow older meson version
sed -i "s/meson_version: '>= 1.4.0'/meson_version: '>= 1.3.2'/g" meson.build

# Replace configure-file .full_path() with compatible equivalent
sed -i "s/compiler_common_flags += \['-include', config_h.full_path()\]/compiler_common_flags += \['-include', meson.current_build_dir() \/ 'config.h'\]/g" meson.build
```

## Gotchas
Ensure that system dependencies are fully satisfied, such as installing `duktape-dev` and `gettext` (for `msgfmt`), which are needed for compilation to succeed.
