# Provenance

Written by the engineer who has shipped this module. The earlier implementation it was audited against was
the public skills catalog of a marketing site: a sync script mirroring each skill
repository into the site's repository, a loader generating a catalog page, category pages and one page per
skill from those files, and the agent eval results of each skill. The architecture is that module's; the
templates carry the changes the audit below records.

Every claim here was verified against the earlier implementation's code, or against the service it calls,
not inferred from a summary. This ledger has thirteen entries, six fixed and seven kept, then the additions.

## Fixed in the templates

### 1. A sync right after a release could mix two commits

The earlier sync read each file from `raw.githubusercontent.com` by branch name and asked the API for the
head commit afterwards. Branch-name raw URLs answer with `cache-control: max-age=300`, so within five minutes
of a push the files could be the previous commit's while `sync.json` recorded the new sha, and the page said
"generated from" a commit it was not generated from.

**Shipped:** the head sha is resolved first and every file is read at that sha; see [sync.md](sync.md).

### 2. A file deleted upstream stayed on the site

The optional `references/provenance.md` was written when present and skipped when it answered 404, never
removed. A skill that moved its ledger into the changelog kept showing the old file's ledger.

**Shipped:** each sync builds the whole `upstream/` folder anew, so only what the commit has survives.

### 3. A failure halfway through a sync emptied the evals folder

The earlier sync deleted `upstream/evals/` before fetching its files, and any non-404 answer after that
threw. The folder was left empty or partial, and the next build showed a skill with no eval runs.

**Shipped:** files are fetched into memory, written to a staging folder and renamed into place; a failure
before the rename changes nothing. See [sync.md](sync.md).

### 4. A missing head commit read as a success

When the commits API failed, the earlier sync recorded `commit: null` and carried on, so a rate-limited sync
looked like a good one with the "generated from" line missing.

**Shipped:** `head()` throws, the skill keeps its previous copy, and the exit code is 1.

### 5. A malformed eval run disappeared, and the test that guarded it could not fail

The earlier parser returned `null` for a run it could not read, the loader filtered the nulls out, and the
test "every committed run parses" iterated only the runs that had parsed. A run with a typo in `agent:` or
a missing `prompt:` was never shown and never reported.

**Shipped:** `parseEval` returns `{ ok: false, reason }`, and `validateCatalog()` turns each rejection into
an error that fails `catalog.test.ts`. See [evals.md](evals.md) and [loader.md](loader.md).

### 6. A skill without a README rules list showed its critical facts as its rules

When the README had no non-negotiables list, the earlier loader fell back to the `SKILL.md` critical facts
(with any list-style hard rules appended), and the page titled them "the rules the module never breaks".
Hard rules written as blockquotes were not read at all.

**Shipped:** the fallback is the `SKILL.md` hard rules, as a list or as blockquotes, and nothing else; the
validator fails a skill with neither. See [parsing.md](parsing.md).

## Kept deliberately

### The run number decides the order within a day

Sorting result files as text puts `-p1.md` after `-p1-2.md`. `runNumber` reads the suffix; it looks like
an avoidable helper and is not.

### The newest run is chosen per agent and prompt text

Prompts with results are never reworded, so the text identifies a prompt across renumbering, while
`promptIndex` only records a position.

### `sync.json` has no timestamp

A re-sync of an unchanged repository must produce no diff, or every sync is a commit and the history means
nothing.

### An unknown category or kind fails the build

A typo in `meta.md` would otherwise hide a skill from its category page with nothing to notice.

### Skills and categories share one route segment

`/skills/content` beside `/skills/ksef` reads well and makes the breadcrumb a path. The cost is a possible
collision, which the validator turns into an error.

### The client gets summaries only

The categories and summary types live in modules with no filesystem access, so the catalog search can
import them, and no whole skill is ever serialised into the page.

### Relative links resolve into the repository

The earlier implementation resolved a skill's relative links against its GitHub URL; the templates keep
that and also resolve `../` and the file's own folder.

## Added

Designed in the skill; the earlier implementation did not have these.

- **The local source** (`SKILLS_SOURCE=local`), reading `git show <sha>:<file>` in checkouts.
- **`SKILL_LAYOUT`**, the heading map; the earlier implementation had the patterns inline.
- **The validator's warnings**: a recorded result that disagrees with the checks (`impliedResult`), a prompt
  reworded after a run, missing sections, a `related` slug with no page.
- **A stricter `runUrl`**: an Actions run URL in the organization, where the earlier check accepted any URL
  under the organization.
- **Fence-aware list reading and escaped pipes in table cells.**
- **`jsonLdScript`**, escaping `<` in JSON-LD; the earlier page wrote it unescaped, with host-written values.
- **The example skill** in `assets/content/`, so an offline build has content.

## If you are porting the earlier implementation

Fix in this order, most damaging first: 5 (runs that vanish without a failing test), 3 (a sync that empties
a folder), 1 (mixed commits), 4, 2, 6.
