---
name: bitwarden-secret-lookup
description: Answers where a secret lives - which copy is the original, which copies exist, who reads it and how, who owns it and how it is rotated - from the team's secret inventory index or lookup command, without opening the vault or reading any value. Use first whenever a task needs to locate a credential, before grepping repositories, opening docs or touching Bitwarden.
---

# Find a secret

Ask the inventory first. Do not grep repositories, read ciphertexts or search the vault to find a secret.

1. Run the lookup command your setup names (project or user instructions such as CLAUDE.md), for example:

   ```sh
   secret-where <term>
   ```

   `<term>` is any part of an item or secret name, an encrypted file path, a consumer, host or service. Without a
   command, search the index file the instructions name: `grep -i -e <term> <index-file>`.
2. **The row is the answer.** Stop there unless you are rotating or moving the secret; do not cross-check other
   docs.
3. Rotating, revoking or moving: open only the runbook the row links, then follow bitwarden-secret-lifecycle
   (rotation) or bitwarden-secret-classify (a new copy or consumer, or a move to another group).
4. **No row:** try one shorter or different term. Still nothing means the secret is not in the inventory: say so
   and ask before guessing where it lives. Registering it is bitwarden-secret-classify.
5. **No inventory configured:** say so and ask. Do not list vault items to look for it: `bw list items` and
   `bws secret list` return values. A person can search the vault by name in the Bitwarden app.

Using the value once found is bitwarden-secret-access.
