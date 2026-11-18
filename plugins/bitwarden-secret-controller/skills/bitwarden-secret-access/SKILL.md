---
name: bitwarden-secret-access
description: Hands a credential kept in Bitwarden (API key, token, DSN, password, admin or web login) to the command that needs it without the value ever appearing in the conversation, using `bws run` for Secrets Manager and a no-echo wrapper for Password Manager items. Use whenever a command, script, deploy or login needs a credential. To find where a credential lives first, use bitwarden-secret-lookup.
---

# Secret access: inject, never print

Everything a tool call prints becomes part of the conversation and can end up in transcripts, logs and caches.
A secret must reach the process that needs it **without being printed**: not by the agent, not by the command,
not by an error message.

## Pick the path

| Need | Path | Who acts |
|---|---|---|
| Machine credential: API key, token, DSN, deploy key, unattended job | Secrets Manager, injected with `bws run` | agent |
| Human login: web console, NAS, admin UI, `sudo` password | Password Manager item, handed over by a no-echo wrapper or by a broker that asks the person to approve each request | agent, with the person's approval where the setup requires it |
| A master password, a 2FA approval on the person's phone, a token that can only be copied from a web page | the person, on the device they are using | person |

If you do not know which item or project holds the credential, run bitwarden-secret-lookup first.

## Secrets Manager: `bws run`

```sh
bws run --project-id <project-uuid> -- ./task.sh
```

- `bws run` exports **every** secret the access token can read in that project as an environment variable named
  after the secret's key. The child reads the ones it needs. Keep projects narrow: a secret that only one job
  should see belongs in its own project.
- Keys must be valid environment variable names; otherwise use `--uuids-as-keynames`.
- `bws run` runs the command through a shell (see `--shell`), so the command line is shell code. Put multi-step
  work, and anything that expands an injected `$VAR`, into a script file and run that file.
- `--no-inherit-env` keeps the caller's environment (including the access token) away from the child. What it still
  passes besides the secrets, such as `PATH`, depends on the `bws` version: use absolute paths to be safe.
- The machine account's access token is a credential too. Pass it as `BWS_ACCESS_TOKEN` in the environment from a
  protected source (OS keychain, encrypted file, CI secret variable), never as `--access-token` on the command line,
  where process listings and shell history keep it. See [references/wrappers.md](references/wrappers.md).
- One access token per consumer or device. Never copy a token file to another machine.

## Password Manager items

The `bw` CLI has no `run` command, and `bw get` prints the value. Use it only inside a wrapper script that puts
the value into the child's environment and never echoes it (example in
[references/wrappers.md](references/wrappers.md)). Unlocking the vault needs the master password: that is the
person's step. Never ask for, type, store or log the master password or a `BW_SESSION` key.

If the setup has a broker that asks the person to approve each request, prefer it for human logins. Treat any
pairing code or token it shows as a credential: save it to a mode-600 file without printing it and pass it through
the environment. When a request is denied, stop and ask; do not retry with guesses, because each guess is another
prompt on the person's screen.

Interactive prompts (`sudo`, `ssh`, installers): feed the value from the environment inside a script
(`sudo -S` reading a variable on standard input, or `expect` reading `$env(VAR)`), never as an argument.

## Scripts that receive secrets

- Start with `set -euo pipefail` and check presence before use: `: "${API_KEY:?API_KEY was not injected}"`.
- Never enable tracing (`set -x`, `bash -x`): it prints every expanded command line, values included.
- To confirm a value arrived, check its length (`${#API_KEY}`) or the result of an authenticated call, never the
  value itself.

## Never

- `bws secret get`, `bws secret list`, `bw get`, `bw list items`, `bw export`, or decrypting an encrypted file to the
  terminal: they print values.
- Print the environment next to an injected command: `env` or `printenv` without arguments, `set` without
  arguments, `export -p`, `declare -p`, `compgen -v`, or a language runtime's `os.environ` / `process.env`.
- Put a value, master password, session key or access token on a command line, in a shell profile, an `.env` file
  committed to git, or a chat message.
- Read a secret back to check it.
- Fetch the value another way after a wrapper fails. Report the error and stop.

If a value was printed anyway, follow bitwarden-secret-lifecycle, "A value was exposed".
