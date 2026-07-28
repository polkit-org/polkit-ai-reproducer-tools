# Component: polkitd Expat CDATA Handling

## When to use
When reproducing polkitd bugs related to `.policy` file parsing, especially regarding special characters, XML entities, or string truncation.

## Technique
polkitd parses `.policy` files using the Expat library. Expat does not guarantee that the character data handler (`_cdata` in `polkitbackendactionpool.c`) will be called only once for a given text node. Specifically:
- Expat splits text chunks and triggers the CDATA callback multiple times when it encounters entity references (e.g., `&amp;`, `&lt;`, `&gt;`).
- If the CDATA callback implementation overwrites the target variable instead of appending the new data to a dynamic buffer (using `g_string_append_len` or similar), only the final text chunk after the last entity reference will be preserved.

To reproduce this:
1. Define a custom action with XML entities inside text fields (`<description>`, `<message>`, `<vendor>`).
2. Reload polkitd daemon.
3. Query the verbose action info using `pkaction -v -a <action-id>`.

## Recipe
Create a policy file with entity references:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE policyconfig PUBLIC "-//freedesktop//DTD PolicyKit Policy Configuration 1.0//EN" "http://www.freedesktop.org/standards/PolicyKit/1/policyconfig.dtd">
<policyconfig>
  <vendor>Some text &amp; some more</vendor>
  <action id="org.example.test">
    <description>Test &amp; Verify</description>
    <defaults><allow_any>yes</allow_any><allow_inactive>yes</allow_inactive><allow_active>yes</allow_active></defaults>
  </action>
</policyconfig>
```

Query the parsed metadata:
```bash
pkaction -v -a org.example.test
```

## Gotchas
- When writing the policy file, make sure it is syntactically valid XML. An invalid XML structure will prevent the policy file from loading altogether.
- Always restart the `polkit` systemd service (`systemctl restart polkit`) to ensure the latest compiled binaries and policies are loaded.
