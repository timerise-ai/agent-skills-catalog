# agent-skills-catalog

[![Agent Skills](https://img.shields.io/badge/Agent_Skills-open_format-059669)](https://agentskills.io)
[![skills.sh](https://img.shields.io/badge/skills.sh-npx_skills_add-059669)](https://www.skills.sh)
[![Claude Code](https://img.shields.io/badge/Claude_Code-compatible-059669)](https://docs.claude.com/en/docs/claude-code/skills)
[![Codex CLI](https://img.shields.io/badge/Codex_CLI-compatible-059669)](https://developers.openai.com/codex/skills)
[![Gemini CLI](https://img.shields.io/badge/Gemini_CLI-compatible-059669)](https://github.com/google-gemini/gemini-cli/blob/main/docs/cli/skills.md)

An [Agent Skill](https://agentskills.io) that teaches an agent to build a public catalog of Agent Skills in a
**Next.js App Router** site, generated at build time from each skill repository's own files: a sync script
that mirrors `SKILL.md`, `README.md`, `CHANGELOG.md`, `references/provenance.md` and `evals/` into the
site's repository, a loader that reads rules, releases and the provenance ledger from them, a catalog page
with search and a category filter, one page per category and per skill, the agent eval results, JSON-LD,
sitemap entries and a content validator.

**A skills catalog is a set of pages about files that live somewhere else, so the page must say exactly what
the skill's repository says, at one commit, and nothing else.** Every template is shaped around that: the
sync reads every file at one commit and swaps it in whole, the build never touches the network, and nothing
from a skill is retyped on the site.

This skill was written by the engineer who has shipped this module; the earlier implementation it was
audited against was the skills catalog of a marketing site. The templates hold the idea above as verified
properties: every file of a skill is read at the sha the sync records, a failed sync leaves the previous copy
untouched, every eval file either renders or fails `npm test` with its reason, and each eval card shows the
recorded result of the newest run per agent and prompt, failing runs included. The parser, eval and content
suites cover each; [references/provenance.md](references/provenance.md) carries the record.

## Install

One command, via the [skills.sh](https://www.skills.sh) CLI, which installs the skill into every
skills-compatible agent it detects, including Claude Code, Codex CLI and Gemini CLI:

```bash
npx skills add timerise-ai/agent-skills-catalog
```

Name the agents instead with `-a`, for example `npx skills add timerise-ai/agent-skills-catalog -a claude-code -a codex`.

### Manual install

Nothing here is Claude-specific: the skill is a plain [Agent Skills](https://agentskills.io) folder,
`SKILL.md` plus markdown references and one example content folder, with no file that calls a model, so
cloning it into an agent's skills directory is all an install is. For Claude Code:

```bash
git clone https://github.com/timerise-ai/agent-skills-catalog.git ~/.claude/skills/agent-skills-catalog
```

To scope it to a single project instead, clone it into that project's `.claude/skills/` directory. For another
agent, clone into that agent's skills directory, or symlink the Claude Code copy so one `git pull` updates
every agent:

```bash
mkdir -p ~/.agents/skills
ln -s ~/.claude/skills/agent-skills-catalog ~/.agents/skills/agent-skills-catalog
```

Update the skill with `git pull` in its directory. The current release is **0.1.2**. See
[CHANGELOG.md](CHANGELOG.md). The [skills index](https://github.com/timerise-ai/skills) lists the other
Timerise Skills and how to install them all at once.

## Activation

The skill activates automatically when a task matches its description: listing an organization's Agent
Skills on a site with their rules, versions, install commands and agent eval results, adding eval results
or a provenance ledger to existing skill pages, or fixing skill pages that show an old version after a
release. Invoke it explicitly with `/agent-skills-catalog` in Claude Code, `$agent-skills-catalog` in Codex
CLI, or from `/skills` in Gemini CLI.

Each host matches a task against the description its own way, so invoke the skill explicitly on a first run
rather than assuming it fired. Only `SKILL.md` is read up front; the `references/` files load on demand, so
the skill stays cheap in context until a topic is actually needed.

## What's inside

| File | Contents |
|---|---|
| `SKILL.md` | Entry point: architecture, critical facts, hard rules, quick start, and the reference directory |
| `README.md` | This front door |
| `CHANGELOG.md` | Keep a Changelog, one section per release, newest first |
| `CLAUDE.md` | What this repository is and the conventions for editing the skill itself |
| `LICENSE` | MIT |
| `references/adaptation.md` | The seam contract with the host app: skill source, layout, categories, rendering, renames, order of work, the non-negotiables restated |
| `references/data-model.md` | Folders, `meta.md`, config, categories and types |
| `references/sync.md` | The sync script: one commit per skill, staged swap, the GitHub and local sources, rate limits, offline use |
| `references/parsing.md` | Frontmatter with date normalization, the heading map, markdown readers, releases, the ledger, the index table |
| `references/evals.md` | Prompts and eval run files, the parser, ordering, the newest run per agent and prompt |
| `references/loader.md` | The catalog loader and the content validator |
| `references/routes.md` | Catalog, category and skill pages, JSON-LD, sitemap |
| `references/ui.md` | Markdown with repository links, cards, catalog search, eval cards, prompts, ledger, style hooks |
| `references/testing.md` | `vitest` setup and the five test files |
| `references/extensions.md` | Scheduled sync, Markdown copies, prices, the site assistant, eval collection, badges |
| `references/provenance.md` | The engineering ledger: what the audit of the earlier implementation changed and how the templates verify it, what was kept on purpose, what is new |
| `assets/content/` | One example skill with every file the loader reads, for a build with no network |
| `evals/` | The prompts an operator types after installing (`prompts.md`) and one file per agent eval: the skill installed into an empty Next.js app, one prompt, no help, then type-checked, built and tested |
| `.github/workflows/agent-eval.yml` | Runs the agent evals on every published release through the index's reusable workflow; copied verbatim from the standard |

The skill source is a seam, not a premise: the sync talks to an object with three methods, `head`, `read`
and `list`, and ships one for GitHub and one for local checkouts. The section layout the loader reads is one
table, `SKILL_LAYOUT`. The module is public, read-only and file-backed: no auth, no tenancy, no database, no
object storage. The host app's styling, renderer and page chrome stay the host app's;
`references/adaptation.md` is where you wire them in.

## The eight non-negotiables

These travel with the module and are never optional. They are the hard rules in `SKILL.md`, in the same
order, and `references/adaptation.md` restates them.

1. **Never fetch a skill at build or request time.** The sync writes files into the repository and the build
   reads the disk, so a deploy never depends on GitHub's rate limit or its cache; the loader imports no
   network code and the build passes offline.
2. **Read every file of a skill at one commit.** The head sha is resolved first and each file is read at
   that sha, so the page and the "generated from" line agree; the sync names the sha in every raw URL.
3. **Swap a skill's mirror in whole or not at all.** The folder is staged and renamed into place, so a
   failed sync leaves the previous copy and a file deleted upstream disappears; a second run of an
   unchanged repository leaves `git status` clean.
4. **Never drop an eval file silently.** `parseEval` returns the reason it cannot show a file, and
   `validateCatalog()` turns each into an error; the content suite fails on a run with no prompt.
5. **Show the recorded result of the newest run per agent and prompt.** Failing runs render like passing
   ones, nothing is recomputed or picked for looking better, and an agent with no run is named; the eval
   suite covers the order and the selection.
6. **Normalize YAML dates before reading frontmatter as text.** An unquoted date is a Date, and a Date read
   as text rejects the run; the eval suite's fixture writes its date unquoted.
7. **Never retype a skill's text on the site.** Rules, releases, prompts and results come from `upstream/`,
   `meta.md` carries only what the repository does not, and the sync never writes `meta.md`.
8. **Only summaries cross to the client.** The catalog search receives `SkillSummary[]`, so no README is
   serialised into the page; the one client component's props are typed that way.

Everything else is the host app's: styling, naming, renderer, page chrome.

## Requirements

- Next.js App Router with React 19 and TypeScript.
- Node 22 or later for the sync script.
- `gray-matter`, `react-markdown`, `remark-gfm`, and `vitest` for the tests.
- Skill repositories in the shared layout, or a `SKILL_LAYOUT` adjusted to theirs.

## Not this

| Not this | Use instead |
|---|---|
| Writing a new skill or hardening one from its eval runs | The skill's own repository; the sibling [`skill-eval-loop`](https://github.com/timerise-ai/skill-eval-loop) skill for the eval loop |
| Running agent evals | The eval harness of the skills index; this module only reads the results |
| A blog or a help center from markdown the site owns | The sibling [`blog-markdown`](https://github.com/timerise-ai/blog-markdown) and [`help-center-markdown`](https://github.com/timerise-ai/help-center-markdown) skills |
| A registry with accounts, uploads and downloads | A package registry; the trust model differs |

## Contributing

Issues and pull requests are welcome here. Pure markdown, with no build or lint step, but the code blocks are
checked: every ` ```ts `, ` ```tsx ` and ` ```js ` block starting with `// file: <path>` is copied into a
scratch Next.js app with `assets/content/`, type-checked under `strict` and `noUncheckedIndexedAccess`,
built, and the `*.test.ts` blocks are run; the sync is run against public skill repositories. Claims in this
skill are meant to be verifiable: if you change a factual claim, say how you verified it, whether against the
Next.js, gray-matter or react-markdown documentation, the GitHub REST API documentation, response headers
from `raw.githubusercontent.com`, or a reproduction.

Adding, removing or renaming a file in `references/` or `assets/` means updating the quick start and the
reference directory table in `SKILL.md`, the file table above, and any relative cross-links. Every
odd-looking part of the templates is there for a reason, and `references/provenance.md` is the ledger that
must stay truthful: read it before simplifying anything, and add an entry for anything you change. Commits
follow Conventional Commits and releases follow
[STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) in the index; `CLAUDE.md` carries
the full editing conventions.

## Part of the Timerise Skills

This is one of the [Timerise Skills](https://github.com/timerise-ai/skills): modules for **Next.js App
Router** apps written by our own senior engineers from the modules they have shipped, not synthetic, each
published as its own repository and indexed there. They share one layout, so an agent that has read one knows
how to read the next: a `SKILL.md` entry point, `references/` loaded on demand, and a seam contract carrying
the module's non-negotiables.

## Author

Built and maintained by [Timerise](https://timerise.ai).

## License

MIT. See [LICENSE](LICENSE).
