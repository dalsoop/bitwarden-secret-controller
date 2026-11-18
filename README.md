# bitwarden-secret-controller

A [Claude Code](https://code.claude.com) plugin marketplace with five agent skills for working with secrets kept in
[Bitwarden](https://bitwarden.com) Password Manager and Secrets Manager, without the values ever entering the
conversation.

| Skill | What it covers |
|---|---|
| `bitwarden-secret-access` | Hand a credential to the command that needs it (`bws run`, no-echo wrappers) without printing it |
| `bitwarden-secret-lookup` | Find where a secret lives from an inventory index or lookup command, and stop at the row |
| `bitwarden-secret-placement` | Password Manager or Secrets Manager, personal or organization, collections, projects, names and metadata fields |
| `bitwarden-secret-classify` | Design a secret inventory: groups, one original per secret, minimal copies, consumers, owner and renewal, a generated index |
| `bitwarden-secret-lifecycle` | Rotate, revoke and retire credentials; handle an exposed value by rotating first and keeping the details sealed |

## Install

In Claude Code:

```
/plugin marketplace add dalsoop/bitwarden-secret-controller
/plugin install bitwarden-secret-controller@bitwarden-secret-controller
```

Restart Claude Code if the skills do not appear right away. The skills load on demand when a task matches their
description; you can also invoke one directly, for example `/bitwarden-secret-controller:bitwarden-secret-lookup`.

## Safety model

- **No values in context.** Anything a tool prints becomes part of the conversation. The skills hand values to
  child processes through the environment (`bws run`, wrapper scripts) and never print, echo, list or read them
  back.
- **The person keeps what only a person should do.** Master passwords, 2FA approvals and console-only steps stay
  with the person; the skills never ask the agent to type or store them.
- **Inventory, not search.** Agents find secrets through a value-free inventory index instead of searching vaults
  or repositories.
- **Rotate first.** An exposed value is rotated before the plaintext is cleaned up, and exposure details are kept
  in a restricted record rather than in chat or docs.

The skills are guidance for the agent, not an enforcement layer. For defense in depth, add a `PreToolUse` hook that
denies commands which print secrets (for example `bws secret get`, `bw get`, bare `env`), and run a secret scanner
such as gitleaks before commits.

## Configure for your setup

The skills work without configuration, but they are most useful when your project or user instructions
(`CLAUDE.md`) name your own pieces, for example:

```markdown
## Secrets
- Inventory index: `docs/secret-index.md` in <your inventory repository>; lookup command: `bin/secret-where <term>`.
- Placement rules: `docs/decisions/0001-secret-placement.md` (these override the plugin's defaults).
- Secrets Manager wrapper: `bws-env run --project-id <uuid> -- <command>`.
```

Everything in the skills that names a group, a file layout or a wrapper is an example you can replace.

## Requirements

- Claude Code with plugin support.
- For Secrets Manager: the [`bws` CLI](https://bitwarden.com/help/secrets-manager-cli/) and a machine account
  access token per consumer.
- For Password Manager items: the [`bw` CLI](https://bitwarden.com/help/cli/), unlocked by the person.

## Disclaimer

This project is not affiliated with, endorsed by or sponsored by Bitwarden, Inc. "Bitwarden" is a trademark of its
owner and is used here only to describe compatibility. Review the skills before use; you are responsible for how
your agents handle your credentials.

## License

[MIT](LICENSE)
