# example-skill

An example skill that teaches an agent nothing; it shows the layout the catalog reads.

**The catalog reads a skill's own files, so this one carries each of them.**

This skill was written by the engineer who maintains the catalog, as a placeholder; the record is in
[references/provenance.md](references/provenance.md).

## What's inside

| File | Contents |
|---|---|
| `SKILL.md` | Entry point |
| `README.md` | This front door |
| `CHANGELOG.md` | Releases, newest first |
| `references/provenance.md` | The ledger |
| `evals/` | Prompts and runs |

## The two non-negotiables

1. **Never ship the placeholder.** Delete this folder once a real skill is synced; the validator warns while
   it is the only skill.
2. **Keep its files in the layout.** It is the shape every synced skill has, so the tests read it as one.

## Not this

| Not this | Use instead |
|---|---|
| A real module | A synced skill |
