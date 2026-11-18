---
name: bitwarden-secret-lifecycle
description: Guides rotating, revoking, retiring and issuing credentials kept in Bitwarden, and handling a value that was exposed (printed, committed or left in plaintext) by rotating first, then removing the plaintext, while keeping the exposure details sealed. Use for key rotation, a leaked or suspected-leaked value, a device being retired, reinstalled or lost, a client or project ending, or onboarding a new one.
---

# Credential lifecycle

Never print, repeat or copy a value at any step; use bitwarden-secret-access to hand values to the tools that
issue, deploy or test them.

## Stages

`active → rotating → active`; `active → deprecated → revoked → archived`. Archived is final. Keep the stage and
its dates in the original's metadata (see bitwarden-secret-placement, "Fields"). A person decides transitions;
automated checks only propose them.

| Transition | Condition |
|---|---|
| deprecated → revoked | no consumer still reads it (`used_by` empty) |
| revoked → archived | someone confirmed the issuer has revoked it |
| active → revoked (emergency) | device retired or lost, or a leak is suspected |

## Rotate a key

Copy this checklist:

```
Rotation:
- [ ] 1 Inventory row found: original, copies, consumers, runbook
- [ ] 2 New value issued at the issuer
- [ ] 3 Original updated first (old value still recoverable)
- [ ] 4 Every copy refreshed, consumers redeployed
- [ ] 5 Consumers verified on the new value
- [ ] 6 Old value revoked at the issuer; `rotated` date set; stage back to active
```

1. Look the secret up (bitwarden-secret-lookup). Follow the runbook the row links when there is one.
2. Issue the new value at the issuer: the person in the issuer's console, or the agent through the issuer's API
   with an injected credential. Prefer issuers that allow two valid keys at once, so consumers switch without
   downtime.
3. Update the **original** first. Make sure the old value stays recoverable until step 6: Password Manager keeps a
   password history for the login password; Secrets Manager does not keep earlier values, so keep the old one as a
   separate secret (for example `<KEY>_PREVIOUS`) or rely on a verified encrypted backup.
4. Refresh every copy the row lists (encrypted files, synced Kubernetes Secrets, CI variables) from the original,
   then redeploy or restart consumers. Deploy to production only as far as the person has approved.
5. Verify each consumer works: a health check or an authenticated call that returns a status, never the value.
6. Revoke the old value at the issuer, delete any `_PREVIOUS` copy, record the `rotated` date and return the
   stage to active.

## A value was exposed

Exposed means printed into a conversation or log, committed, pasted into a ticket or chat, or left in a plaintext
file.

1. Do not repeat, transform, hash in the open or save the value.
2. **Rotate first**, as above: new value, original, copies, verify, then revoke the old one.
3. **Then remove the plaintext** where your storage and retention rules allow: delete the file, drop the commit
   from an unmerged branch before pushing again, purge the log. Do not rewrite shared history to "fix" a leak:
   once a value was pushed, rotation is the fix. Backups and snapshots usually expire on their own schedule.
4. **Keep the details sealed.** Which secret, where it appeared and any fingerprint go only into a restricted
   record: encrypted to the owner, or an incident system with need-to-know access. Not into chat channels,
   tickets, task notes, vault notes or repository docs. The general notice says only something like
   "one exposure, rotated; sealed record exists".

A clearly verified placeholder or test fixture needs no rotation: say why and continue.

## Device events

- **Reinstalled:** issue the device a new access token for its machine account and revoke the old one.
- **Retired or lost:** revoke that device's machine-account access tokens at once (emergency path), remove its
  access, and archive the logins that only it used.
- Never move a device's token to another device; issue the other device its own.

## A client or project ends

Revoke its machine accounts' access tokens, remove external people's access to its collection, and mark its
items archived in their metadata. Keep the collection and the Secrets Manager project as an audit trail unless
your retention rules say otherwise; empty the project of live secrets.

## A new client or project

Create its collection and, if machines need secrets, its Secrets Manager project and machine account with the
least access that works (read-only unless the account itself rotates secrets). Placement and naming follow
bitwarden-secret-placement; the inventory entry follows bitwarden-secret-classify.
