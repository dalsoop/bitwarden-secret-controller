# Example wrappers

These are starting points, not requirements. Adapt names, paths and the token source to your setup. The person
installs them once; the agent only calls them.

## Contents

- `bws-env`: supply the Secrets Manager access token without putting it on the command line
- `bw-exec`: hand one Password Manager field to one command
- Checklist for any wrapper

## `bws-env`

Reads the machine account's access token from a protected store and runs `bws` with the token only in its own
environment.

```bash
#!/usr/bin/env bash
# Usage: bws-env run --project-id <uuid> -- ./task.sh
set -euo pipefail

# Pick one source. Each one returns the token on stdout, which the assignment captures, so nothing is printed.
# macOS keychain (item created once by the person):
BWS_ACCESS_TOKEN="$(security find-generic-password -s bws-access-token -a "$USER" -w)"
# Linux Secret Service:
#   BWS_ACCESS_TOKEN="$(secret-tool lookup service bws-access-token)"
# age-encrypted file:
#   BWS_ACCESS_TOKEN="$(age -d -i "$HOME/.config/age/identity.txt" "$HOME/.config/bws/token.age")"

export BWS_ACCESS_TOKEN
exec bws "$@"
```

Then: `bws-env run --project-id <project-uuid> -- ./deploy.sh`.

Without `--no-inherit-env`, `bws run` passes its own environment to the child, so `./deploy.sh` also receives
`BWS_ACCESS_TOKEN`. Add `--no-inherit-env`, or have the child `unset BWS_ACCESS_TOKEN` first, when it should only see
the project's secrets.

Run the token lookup only inside such a script: run on its own, `security … -w` prints the token.

## `bw-exec`

Hands one field of one Password Manager item to one command as an environment variable. It expects an unlocked
session (`BW_SESSION`) that the person provided; it never prompts for or reads the master password.

```bash
#!/usr/bin/env bash
# Usage: bw-exec <item-id> <field: password|username|totp> <ENV_NAME> -- <command> [args...]
set -euo pipefail
[ "$#" -ge 5 ] && [ "$4" = "--" ] || { echo "usage: bw-exec <item-id> <field> <ENV_NAME> -- <command>" >&2; exit 2; }
item="$1" field="$2" name="$3"
shift 4

case "$field" in password|username|totp) ;; *) echo "bw-exec: unsupported field" >&2; exit 2 ;; esac
case "$name" in *[!A-Za-z0-9_]*|[0-9]*|'') echo "bw-exec: bad variable name" >&2; exit 2 ;; esac

value="$(bw get --nointeraction "$field" "$item")"
export "$name=$value"
unset value
unset BW_SESSION   # the child gets one field, not the unlocked vault
exec "$@"
```

Then: `bw-exec 1f0c…-item-id password DB_PASSWORD -- ./migrate.sh`.

## Checklist for any wrapper

- The value travels only through a variable assignment, the child's environment or standard input; never argv,
  never stdout.
- No `set -x`, no debug logging of the environment, no echo of the value on error.
- Fail closed: a missing token or a locked vault ends with an error message that names the cause, not the value.
- The wrapper runs one child per call and does not cache the value on disk.
