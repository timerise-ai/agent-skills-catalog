# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

An [Agent Skill](https://agentskills.io) package: markdown, plus one example content folder in `assets/`.
There is no `package.json` here and nothing in this repository executes. It teaches an agent how to build a
public catalog of Agent Skills (a sync script, a loader, catalog, category and skill pages, agent eval
results, JSON-LD, sitemap, a content validator) in a **Next.js App Router** app, with the skill source and
the section layout as seams.

Keep the two straight: the commands and code in `references/` describe the app the agent will generate, not
this repository. The sync script, the loader, the pages and the tests all run in that generated app.

The skill was written by the engineer who has shipped this module; `references/provenance.md` is the
engineering ledger, thirteen entries recording what the audit of the earlier implementation changed and how
the templates verify it and what was kept on purpose, then what was designed in the skill. That file is the
rationale layer: read it before "simplifying" anything.

This is one of the Timerise Skills. Layout, writing style, releases and evals follow the index's
[STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md); a checkout of the index usually
sits beside this repository as `../skills`. Section 8 there describes this file, section 9 the release
procedure, section 10 the eval procedure and section 12 the pre-release checklist.

## Structure

- `SKILL.md`: the entry point, loaded whole on every activation, so it stays at 130 to 160 lines. The
  frontmatter `description` is the trigger surface. The body carries the architecture, the critical facts,
  the hard rules, the quick start, the reference directory table mapping trigger keywords to `references/`,
  and a closing line linking the skills index.
- `references/*.md`: one topic per file, loaded on demand. `adaptation.md` (the seam contract and the
  non-negotiables restated) and `data-model.md` (config, categories, types) are the design entry points;
  `sync.md`, `parsing.md`, `evals.md`, `loader.md`, `routes.md`, `ui.md` and `testing.md` carry the
  templates; `extensions.md` the designs beyond the module; `provenance.md` the ledger.
- `assets/content/skills/example-skill/`: a placeholder skill with every file the loader reads, copied into
  the app when the sync cannot run. Its `upstream/sync.json` has a null commit and an empty file list on
  purpose: no sync wrote it. It must pass the validator; change it together with the loader.
- `README.md`: the human front door, in the fixed section order of the index's STANDARD.md. `CHANGELOG.md`:
  Keep a Changelog, newest first.
- `evals/`: `prompts.md` holds what an operator types after installing, in their words; the first prompt is
  the agent eval run before every release. Every other file there is one eval run: measured frontmatter that
  is never edited, then the notes of the person who ran it. Add a prompt rather than rewording one that has
  results. The procedure is section 10 of the index's STANDARD.md.
- `.github/workflows/agent-eval.yml`: the caller of the index's reusable eval workflow, copied verbatim from
  STANDARD.md. Never edit it and never add a trigger.

## Editing conventions

- **Code blocks are compiled.** Every ` ```ts ` / ` ```tsx ` / ` ```js ` block whose first line is
  `// file: <path>` is copied into a scratch Next.js app together with `assets/content/`, type-checked under
  `strict` and `noUncheckedIndexedAccess`, built, and the `*.test.ts` blocks are run. Keep that first line,
  keep imports complete, and re-run the check (below) after editing any block.
- **Identifiers are shared across files.** `SKILLS_CATALOG`, `SKILL_CATEGORIES`, `SKILL_LAYOUT`,
  `readFrontmatter`, `parseEval`, `EvalParse`, `EvalRun`, `latestRuns`, `sortRuns`, `runNumber`,
  `impliedResult`, `getSkill`, `getAllSkills`, `validateCatalog`, `SkillSummary`, `toSkillSummary`,
  `repoUrlTransform` and `SkillSource` appear in several references. Rename in all of them or none.
- **Keep three lists in sync with `references/` and `assets/`**: the reference directory table in
  `SKILL.md`, the quick start in `SKILL.md`, and the file table in `README.md`. Links are relative:
  `[x.md](references/x.md)` from `SKILL.md`, `[x.md](x.md)` between references.
- **The odd-looking parts stay.** The sha read before any file, the staging folder and rename, the literal
  content path in `catalog.ts`, `runNumber`, the `runUrl` allowlist, Date normalization in
  `readFrontmatter`, the comma rule when joining a rule to its reason: each is a ledger entry or a measured
  behaviour. Check `provenance.md` first; a change to a template adds an entry there.
- **Measured numbers are load-bearing.** `max-age=300` on branch-name raw URLs, 60 and 5,000 API requests an
  hour, two API calls per skill, 155 characters for a search snippet: verified facts and design parameters;
  do not restate them loosely and do not invent new ones. Corpus counts of the earlier implementation stay
  out of every file.
- **Additions are marked as additions.** Anything designed in the skill and never run in production goes
  under *Added* in `provenance.md`; new capability goes there too, or in `extensions.md` as a design.
- **The eight non-negotiables are never optional.** They are the hard rules in `SKILL.md`, the list in
  `README.md` and the list in `adaptation.md`, in the same order: no fetch at build time, one commit per
  skill, swap in whole, no silent eval drops, the recorded newest result, date normalization, no retyped
  skill text, summaries to the client. Never present any of them as optional in a reference.
- **The host renames the categories and the route; the files stay.** Category ids are generic on purpose.
  `meta.md` fields, `upstream/` file names and eval frontmatter fields are the skills' own contract and are
  never renamed.
- **The description is the trigger surface.** If the skill's scope changes, update its trigger keywords and
  the *When to use* and *When NOT to use* sections together.
- **Plain punctuation, prose wrapped at 110 columns.** No em-dashes, en-dashes, arrows, middle dots or smart
  quotes anywhere in the markdown, code comments included; a rendered UI glyph is written as a JavaScript
  escape.

### Checks

There is no build, lint or test step in this repository, so these run by hand from its root.

- **Punctuation.** Prints nothing when the repository is clean:

  ```bash
  perl -CSD -ne 'print "$ARGV:$.: $_" if /[\x{2010}-\x{2015}\x{2018}\x{2019}\x{201C}\x{201D}\x{2026}\x{2190}-\x{21FF}\x{00B7}]/; close ARGV if eof' $(find . -name '*.md' -not -path './.git/*')
  ```

- **Line width and `SKILL.md` length.** Table rows, commands, YAML frontmatter and code blocks may run past
  110, so read what the first command prints instead of treating every hit as a failure. The second must be
  between 130 and 160.

  ```bash
  awk 'length > 110 {print FILENAME":"FNR": "length}' SKILL.md README.md CHANGELOG.md references/*.md
  wc -l SKILL.md
  ```

- **The three lists.** Compare `ls references/` with the files named by the reference directory and the
  quick start in `SKILL.md` and the file table in `README.md`:

  ```bash
  grep -o 'references/[a-z-]*\.md' SKILL.md README.md | sort | uniq -c
  ```

- **Code blocks.** Create a scratch Next.js app with TypeScript and the App Router and no `src/` folder,
  set `strict` and `noUncheckedIndexedAccess` in its `tsconfig.json`, install `gray-matter`,
  `react-markdown` and `remark-gfm`, and `vitest` as a dev dependency (with `@types/node@latest` on
  `ERESOLVE`). Write every `// file: <path>` block from `references/` to that path, copy `assets/content/`
  to `content/` (the loader reads `content/skills/`), add the `test` script from `references/testing.md` and
  a `typecheck` script that runs `tsc --noEmit`. On a project whose config sets `cacheComponents`, apply the
  variant in `references/adaptation.md`. Then run `npm run typecheck`, `npm run build` and `npm test`; the
  five test files pass with 24 tests. The sync is not part of this check.

### Commits and releases

- **Conventional Commits** between releases: `docs(readme): ...`, `feat(references): ...`,
  `fix(templates): ...`. Eval prompts and results land as `chore(evals): ...` and never bump the version.
  No attribution trailers, generated-by lines or session links, in files or in commit messages.
- **Semantic Versioning** as STANDARD.md section 9 defines it. Wording, `README.md` or `CLAUDE.md` alone is
  a PATCH. A new reference, extension or asset is a MINOR. A change to the seam contract, to canonical
  vocabulary the host relies on, or to a non-negotiable is a MAJOR, and the changelog says how to migrate.
- **The version lives in three places** that must agree: the newest `CHANGELOG.md` section, the current
  release line in the README's *Install* section, and the git tag `vX.Y.Z`. The release commit is
  `chore(release): X.Y.Z` and contains only those two files. The GitHub Release is titled `vX.Y.Z` with
  that changelog section as its notes; publishing it runs the agent evals through the workflow above.
