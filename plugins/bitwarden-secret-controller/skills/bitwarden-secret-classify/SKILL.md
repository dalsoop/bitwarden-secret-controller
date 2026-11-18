---
name: bitwarden-secret-classify
description: Designs a secret inventory and registers credentials in it - groups, exactly one original per secret, the few allowed copies, consumers and how each reads the value, owner and renewal, runbook - and keeps a generated, value-free index that answers "where does X live". Use when setting up an inventory, adding a new credential, giving an existing one a new copy, consumer or owner, or moving it to another group. Never handles values; the Bitwarden side is bitwarden-secret-placement and issuing values is bitwarden-secret-lifecycle.
---

# Classify and register a credential

An inventory answers, for every secret: where its **one original** is, which **copies** exist, who **reads** it
and how, who **owns** it, and how it is **rotated**. It holds names, paths, ids and optional fingerprints, never
values.

If your team already has an inventory and its own rules (named in CLAUDE.md), follow them; this page describes the
model and the default workflow.

## The model

- **Group:** a class of credentials that share who reads them, who issues them, where the original lives, the name
  form, the allowed copy kinds, the owner and the renewal procedure. Every secret is in exactly one group.
  Example groups are in [references/inventory-model.md](references/inventory-model.md); define your own.
- **One original per secret:** one authoritative place (a Password Manager item field or a Secrets Manager
  secret). Everything else is a copy refreshed from it.
- **Copies:** as few as possible. Add one only for a consumer that cannot read the original where it runs, and only
  of a kind the group allows (for example an encrypted file in a repository, a Kubernetes Secret synced from
  Secrets Manager by an operator, a masked CI variable).
- **Consumers:** each reader and exactly how it reads the value: a file path, an environment variable or a command.
- **Owner and renewal:** the group's default, overridden per row when a credential differs.
- **Generated index:** a hand-written ledger plus facts read from the systems themselves (Secrets Manager names
  and ids, operator maps, encrypted file paths) produce one index file and a check that fails on drift. People and
  agents read the index; nobody edits it by hand.

## Workflow

Copy this checklist:

```
Credential design:
- [ ] 1 Existing rows looked up
- [ ] 2 Group chosen
- [ ] 3 Original and its ledger address
- [ ] 4 Name
- [ ] 5 Copies (as few as possible)
- [ ] 6 Consumers and how each reads it
- [ ] 7 Owner and renewal
- [ ] 8 Runbook
- [ ] 9 Original exists (id known; fingerprint if you record them)
- [ ] 10 Ledger entries written
- [ ] 11 Index regenerated; check passes
- [ ] 12 Lookup shows the expected row
```

1. **Look up** the service, item, file or variable name (bitwarden-secret-lookup). A row means the secret exists;
   reclassifying or moving it is its own change. No row: it is new.
2. **Group.** Pick the one group whose readers and issuer match. A value shared by a person and a service, or by
   two unrelated systems, is split into separate credentials first. No group fits, or two do: stop and ask.
3. **Original.** The place the group names; the vault, collection or project follows bitwarden-secret-placement.
   Write its address in the ledger's address form (see the reference).
4. **Name.** The group's name form (bitwarden-secret-placement, "Names").
5. **Copies.** Start from none. For each copy record its kind, location and who can decrypt or read it.
6. **Consumers.** Name each reader and how it reads the value. Prefer reading the original directly at run time
   (bitwarden-secret-access) over adding a copy.
7. **Owner and renewal.** Group default unless this credential's expiry or issuer differs.
8. **Runbook.** Group default, or a link to its own procedure.
9. **Original exists.** Creating the item is bitwarden-secret-placement; issuing the value is
   bitwarden-secret-lifecycle. They return an id, never a value.
10. **Register** the ledger entries (one per field), with any new copy, consumer or group rule.
11. **Regenerate and check** the index. On a failure, fix the input it names and run both again. Commit the
    regenerated index together with its inputs, through review.
12. **Verify** with the lookup: exactly one row per field, with the group, original, copies, consumers, owner and
    runbook from steps 2 to 8. Two rows for one field, or a wrong original, means the registration is wrong.

## Design note

Write the design down before registering, in the task record or the change description:

```
credential:  <what it is, who issues it>
group:       <group> (why)
original:    <ledger address>
name:        <item or secret name>
copies:      none | <kind> <location> (readers)
consumers:   <reader> reads <file path | env var | command>
owner:       group default | "<text>" (why)
runbook:     group default | <link> (why)
verify:      <lookup command> <term> -> <expected row>
```

Ledger and index formats, example groups and the drift checks are in
[references/inventory-model.md](references/inventory-model.md).
