# Adaptation contract

Everywhere this module touches its host. Fill in the right-hand column from the target app before writing
code, and confirm the organization and the categories with the user: they are the two things you cannot
infer. Working unattended, use the placeholders and list them in the handover.

| Seam | This skill ships | The host supplies | Where it plugs in |
|---|---|---|---|
| Skill source | GitHub (`api.github.com` + sha-pinned raw files) and local checkouts | The organization, the branch, a token for many skills, or another `SkillSource` | `scripts/sync-skills.mjs`, `SKILLS_CATALOG.org`, `branch` |
| Index | The index README's table, optional | The index repository, or `null` | `INDEX_REPO` in the sync |
| Skill layout | The shared layout's headings | Other headings, when its skills use them | `SKILL_LAYOUT` in `layout.ts` |
| Categories | Four generic ids with labels | Its ids, labels and descriptions | `SKILL_CATEGORIES`, `SKILL_CATEGORY_META` |
| Site-side facts | `meta.md` schema | One `meta.md` per skill | `content/skills/<slug>/meta.md` |
| Site identity | `siteUrl`, `siteName`, `basePath` | Its values, as literals | `SKILLS_CATALOG` |
| Frontmatter | `gray-matter` with Date normalization | `gray-matter`, or its parser behind `readFrontmatter` | `frontmatter.ts` |
| Markdown rendering | `react-markdown` + `remark-gfm` with `repoUrlTransform` | Its renderer, keeping the transform | `SkillMarkdown.tsx` |
| UI primitives | Plain elements, ARIA, `data-skill-*` hooks | Its cards, buttons, inputs, badges | `components/skills/` |
| Styling | Layout intent only | Its design system on the hooks | the hook table in [ui.md](ui.md) |
| Page chrome | Bare `<main>` per page | Its header, footer, layout | the route group the pages live in |
| Locale | English page text | Its i18n, if the site is translated | the page and component strings |
| Tests | `vitest` files | `vitest` or `bun test` | `*.test.ts` |

No auth, no tenancy, no database, no object storage: the module is public, read-only and file-backed, and
the only write is the sync on a person's machine or in a job.

## Host probe

```bash
grep -E '"(next|react|gray-matter|react-markdown|remark-gfm|vitest)"' package.json
ls app src/app 2>/dev/null                      # where routes live, route groups, a [lang] segment
ls content 2>/dev/null                          # where other markdown lives
grep -rln "react-markdown\|MarkdownRenderer" --include=*.tsx . 2>/dev/null | grep -v node_modules | head -3
grep -rn "sitemap" --include=sitemap.ts -l . 2>/dev/null | grep -v node_modules
cat CLAUDE.md AGENTS.md 2>/dev/null | head -60
```

Read the host's closest content section end to end (a blog, a docs section) and copy its file layout, its
naming and how its pages call `generateMetadata`. A host with `src/` puts `lib/`, `components/` and `app/`
under it and points the vitest alias at `./src/`.

**Add no dependency the host lacks without asking.** Working unattended, install `gray-matter`,
`react-markdown`, `remark-gfm` and, as a dev dependency, `vitest` from npm, and list them in the handover.

## Renames

| Canonical | The host may rename to | Never renamed |
|---|---|---|
| `/skills` | `/agent-skills`, `/modules`, under a locale prefix | |
| Category ids and labels | Its own | |
| `Skill`, `SkillSummary`, `EvalRun` | Its naming convention, in every file at once | |
| | | `meta.md` field names, `upstream/` file names, eval frontmatter fields: they are the skills' own contract |

A rename is all-or-nothing: grep every reference for the old name and change them together.

## A locale prefix

A host whose routes sit under `app/[lang]/` keeps the pages there, adds `lang` to `generateStaticParams`
for each locale, and prefixes `skillPath` and `skillCategoryPath`. The upstream text is in the language the
skill is written in; translating it is out of scope, so a translated site shows the skill's text as it is
and marks the page with that language.

## Order of work

1. Probe, then settle the organization, branch, categories and `siteUrl`.
2. Config, categories, types ([data-model.md](data-model.md)).
3. Sync and content: real or the example ([sync.md](sync.md)).
4. Readers, eval parser, loader, validator ([parsing.md](parsing.md), [evals.md](evals.md),
   [loader.md](loader.md)).
5. Routes and SEO ([routes.md](routes.md)), then UI ([ui.md](ui.md)).
6. Tests, typecheck, build ([testing.md](testing.md)), then the handover in the last step of the quick start.

## The eight non-negotiables

The same list, in the same order, as the hard rules in `SKILL.md` and the README.

1. **Never fetch a skill at build or request time.** The sync writes the files; the build reads the disk.
2. **Read every file of a skill at one commit.** Head sha first, every read at that sha.
3. **Swap a skill's mirror in whole or not at all.** Stage, rename; a failed sync changes nothing.
4. **Never drop an eval file silently.** `parseEval` returns the reason; `validateCatalog()` fails the tests.
5. **Show the recorded result of the newest run per agent and prompt.** Failing runs included, untried agents
   named.
6. **Normalize YAML dates before reading frontmatter as text.** Through `readFrontmatter`, always.
7. **Never retype a skill's text on the site.** Upstream text from `upstream/`, site-side facts in `meta.md`.
8. **Only summaries cross to the client.** `SkillSummary[]` to the catalog, never `Skill`.
