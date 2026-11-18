---
name: bitwarden-secret-placement
description: Decides where a credential's original belongs in Bitwarden - Password Manager or Secrets Manager, personal vault or organization, which collection or project - and how it is named and which metadata fields it carries; also moves items between vaults without breaking references. Use when adding, naming, moving or re-homing a credential in Bitwarden. Designing the whole inventory entry (copies, consumers, owner) is bitwarden-secret-classify.
---

# Where a secret belongs

If your team has written placement rules (for example a placement decision record named in CLAUDE.md), those
rules win over this page. This page is the default when there are none.

Look the secret up first (bitwarden-secret-lookup). If it is already listed, its original stays where the row
says until a deliberate move; do not re-home it as a side effect of another task.

## Password Manager or Secrets Manager

| Who reads the value | Store the original in |
|---|---|
| A person: logins, recovery codes, TOTP seeds, cards, notes | Password Manager item |
| A machine: service API keys, DSNs, deploy and CI tokens, webhook secrets | Secrets Manager secret in a project |
| Both a person and a machine | Split it: a person's login and a separate service credential, each with its own original |

A value shared by two different kinds of consumer cannot be rotated or revoked for one without breaking the other.

## Personal or organization

- **Personal vault:** only credentials that belong to one person and nobody else will ever need.
- **Organization:** anything that belongs to the team or company, is shared, or must survive the person leaving.
  Company credentials never live in a personal vault.
- Keep tenants apart: personal or family, company, and each client get separate organizations or at least separate
  collections with separate access.

## Collections and projects

- **Collection** (Password Manager): one per area of responsibility (team, product, client). Grant access to groups,
  not individuals. Put each item in exactly one collection so its access is unambiguous. Use custom fields, not
  extra collections, for metadata such as hosting type or stage.
- **Project** (Secrets Manager): one per injection boundary, because `bws run` exposes the whole project to the
  child. A common form is `<environment>-<service>`, for example `prod-billing-api`. One machine account per
  consumer, read-only on the projects it needs; write access only for automation that rotates secrets.

## Names

- Logins: `<service> · <account>`. A social login names its provider: `<service> (Google SSO) · <account>`.
  On a collision append the environment, then the host, then ` · 2`.
- Secrets Manager keys: the environment variable name the consumer expects, in `UPPER_SNAKE_CASE`, for example
  `STRIPE_API_KEY`.
- Machine-facing names (project names, secret keys, encrypted file names) are referenced by code: renaming one is
  a change to every consumer, not a cosmetic edit.

## Fields

Give every original the same metadata: custom text fields on a Password Manager item, or `key: value` lines in a
Secrets Manager secret's note.

| Field | Meaning |
|---|---|
| `owner` | person or team that renews it and answers for it |
| `issuer` | where it is issued and revoked (console URL or API) |
| `created`, `expires` | dates; `expires` empty if it does not expire |
| `stage` | `active`, `rotating`, `deprecated`, `revoked`, `archived` |
| `used_by` | consumers, matching the inventory |
| `runbook` | link to the rotation procedure, if it has its own |

Logins may also carry `login_via` (provider and account for social logins) and `recovery` (where recovery codes
are). Unknown values stay empty and are reported as "to fill", never guessed. Turn on master password re-prompt for
high-privilege and finance items.

## Move without breaking references

Move an item only when the task calls for that move.

1. List every reference first: inventory rows, scripts and wrappers that use the item or secret id, CI variables,
   cluster configuration.
2. Check what a move loses: cloning an item into another vault can drop attachments and password history, and
   the new item gets a new id. Items with attachments go on a question list instead of being cloned.
3. Never widen access in the process; the destination collection's members must be ones who already had access or
   are meant to.
4. Update the references through review, verify consumers, then move the old item to the trash (recoverable for
   30 days). Do not delete it permanently as part of a move.
5. Record old and new ids and both locations in the task record.

**Unknown group, vault or owner:** leave the item where it is, list its name, and ask for a decision.
