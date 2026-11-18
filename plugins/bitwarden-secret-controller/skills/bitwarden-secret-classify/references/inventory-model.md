# Inventory model: an example

One workable shape for a secret inventory. Every name, group and file layout here is an example; keep the ideas
(one original, few copies, generated index, drift check) and adapt the rest.

## Contents

- Example groups
- Ledger address forms
- Ledger entry
- Group rules
- Generated index
- Drift checks
- Fingerprints
- A minimal lookup command

## Example groups

| Group | Typical credentials | Original | Allowed copies | Default owner |
|---|---|---|---|---|
| `human-login` | web consoles, admin UIs, SaaS accounts | Password Manager item in the area's collection | none | the account holder |
| `service-key` | API keys and DSNs used by deployed services | Secrets Manager secret in the service's project | Kubernetes Secret synced by an operator | the service's team |
| `ci-deploy` | registry, deploy and package-publish tokens | Secrets Manager project for CI | masked CI variable | platform team |
| `signing` | code-signing keys, certificates, notarization credentials | Password Manager item (attachment) or a hardware key | build machine keychain | release owner |
| `machine-token` | Secrets Manager access tokens of devices and services | none outside the consumer: issued per consumer | an encrypted file on that one consumer | device or service owner |
| `encryption-key` | keys that decrypt the inventory's encrypted copies | Password Manager item, owner-only | none | security owner |
| `personal` | one person's private accounts | that person's personal vault | none | that person |

## Ledger address forms

| Form | Meaning |
|---|---|
| `pm:<vault-or-collection>/<item-id>#login.username` | a login's own username slot |
| `pm:<vault-or-collection>/<item-id>#login.password` | a login's own password slot |
| `pm:<vault-or-collection>/<item-id>#field:<name>` | a custom field |
| `pm:<vault-or-collection>/<item-id>#attachment:<file>` | an attachment |
| `sm:<project>/<secret-key>` | a Secrets Manager secret |

Use ids, not display names, where a name can change; keep a label map from id to readable name for the index.

## Ledger entry

One entry per field. JSON, YAML or TOML all work; keep it in a reviewed repository.

```json
{
  "id": "billing-api/stripe-key",
  "original": "sm:prod-billing-api/STRIPE_API_KEY",
  "copies": [
    { "kind": "k8s-secret", "where": "prod/billing/billing-api-secrets#STRIPE_API_KEY", "via": "secrets-operator" }
  ],
  "consumers": [
    { "who": "billing-api (prod)", "reads": "env STRIPE_API_KEY" }
  ],
  "fingerprint": "sha256:3f9a1c2b",
  "owner": null,
  "runbook": null
}
```

`owner` and `runbook` are `null` when the group default applies.

## Group rules

Map entries to groups by rules, first match wins, so new entries rarely need a group written by hand:

```json
[
  { "match": { "original_prefix": "sm:prod-" }, "group": "service-key" },
  { "match": { "original_prefix": "sm:ci-" }, "group": "ci-deploy" },
  { "match": { "original_prefix": "pm:" }, "group": "human-login" }
]
```

Each group also names its default owner, runbook, name form and allowed copy kinds.

## Generated index

A generator reads the ledger, the group rules and facts from the systems (Secrets Manager project and secret
names and ids, operator maps from the cluster repository, encrypted file paths) and writes one table, one row per
field:

| Secret | Group | Original | Copies | Read by | Owner | Runbook |
|---|---|---|---|---|---|---|
| billing-api/stripe-key | service-key | `sm:prod-billing-api/STRIPE_API_KEY` | k8s `prod/billing/billing-api-secrets` | billing-api: env `STRIPE_API_KEY` | billing team | group default |

Human docs keep a single pointer to this index instead of their own lists of where secrets live.

## Drift checks

The generator's check mode exits non-zero when:

- a copy exists in a system (an encrypted file, an operator map entry, a Secrets Manager secret) with no ledger
  entry;
- a ledger entry's original or copy no longer exists;
- an entry matches no group rule, or a copy kind the group does not allow;
- the committed index differs from a fresh generation.

Run it before merging changes to the inventory and on a schedule.

## Fingerprints

Optional. A fingerprint shows whether a copy matches its original without revealing either. Compute it only inside
a child that received the value (bitwarden-secret-access), for example the first 8 hex characters of a SHA-256
of the value, and print only that prefix. Never compute it in the conversation from a printed value.

A plain hash of a low-entropy value (a short password or PIN) can be brute-forced offline. For those, use a keyed
hash (HMAC with a key kept out of the inventory) or record no fingerprint.

## A minimal lookup command

Searching the generated index is enough:

```bash
#!/usr/bin/env bash
# Usage: secret-where <term>
set -euo pipefail
index="${SECRET_INDEX:-docs/secret-index.md}"
head -n 2 "$index"
grep -i -F -e "$1" "$index" || { echo "no row matches: $1" >&2; exit 1; }
```
