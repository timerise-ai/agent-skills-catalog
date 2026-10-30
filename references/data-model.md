# Data model: config, categories, types, folders

What the catalog is made of before any of it is read. Three files hold the shapes every other reference
uses; none of them touches the filesystem, so client components may import them.

## The folders

```
content/skills/
  _index/README.md                 optional: the index repository's README, its skills table
  <slug>/
    meta.md                        written by the host, never by the sync
    upstream/                      written by the sync, never by hand
      SKILL.md README.md CHANGELOG.md
      references/provenance.md     only when the skill has one
      evals/prompts.md
      evals/<date>-<agent>-p<n>[-<k>].md
      sync.json                    the commit every file above was read at
```

| Path | Written by | Read for |
|---|---|---|
| `<slug>/meta.md` | The host | Category, tagline, search snippet, platform, related, the problem statement |
| `upstream/SKILL.md` | Sync | Description, When to use, When NOT to use, hard rules, sources |
| `upstream/README.md` | Sync | Origin paragraph, non-negotiables, Not this, Requirements, What's inside |
| `upstream/CHANGELOG.md` | Sync | Releases, version, dates; the ledger when there is no provenance.md |
| `upstream/references/provenance.md` | Sync | The ledger: fixed, kept, added |
| `upstream/evals/prompts.md` | Sync | The prompts an operator types |
| `upstream/evals/*.md` | Sync | One agent eval run each |
| `upstream/sync.json` | Sync | The commit sha and date, for the "generated from" line |
| `_index/README.md` | Sync | The "Builds" and "Backend" cells of each skill's row |

A folder is a skill page once it has both `meta.md` and `upstream/SKILL.md`. A folder whose name starts
with `_` is never a skill. The slug is the folder name; `meta.md` names the repository when it differs.

## meta.md

The facts a skill repository does not carry and the host decides. The body is the plain-language problem
statement, written for a visitor who has never heard of the skill.

```markdown
---
repo: help-center-markdown        # optional, the folder name when absent
name: help-center-markdown        # optional, the repository name when absent
category:                         # one id, or a list with the primary first
  - content
tagline: "A help center from markdown files in the repository: search, tags, locales."
seoDescription: "..."             # optional, at most 155 characters; the tagline when absent
platform: Next.js                 # optional, SKILLS_CATALOG.defaultPlatform when absent
kind: module                      # optional: module (default) or tool
related:
  - blog-markdown
---

A help center has one hard requirement: every article must be reachable.
```

| Field | Required | Rule |
|---|---|---|
| `category` | yes | Every id must be in `SKILL_CATEGORIES`; an unknown or missing one throws at build |
| `tagline` | yes | One sentence; the card and the page header |
| `seoDescription` | when the tagline runs past 155 characters | The validator fails a longer one |
| `related` | no | Slugs; one without a page is dropped from the page and warned about |
| `kind` | no | `tool` for a skill an agent runs on something other than the app; the host decides what changes |

Frontmatter field names are the authoring contract and are never renamed; the host renames the category
ids and labels.

## Config

```ts
// file: lib/skills/config.ts
/**
 * Every value the catalog needs from the host, in one place. Nothing here
 * touches the filesystem, so client components may import it.
 */
export const SKILLS_CATALOG = {
  /** Where `npm run skills:sync` writes and the loader reads, relative to the app root. */
  contentDir: "content/skills",
  /** The GitHub organization that owns the skill repositories. */
  org: "your-org",
  branch: "main",
  /** The route the catalog lives under. */
  basePath: "/skills",
  /** Literals, never environment variables: they land in JSON-LD and the sitemap at build time. */
  siteUrl: "https://example.com",
  siteName: "Example",
  /** What a skill runs on when its meta.md does not say. */
  defaultPlatform: "Next.js",
  license: "https://opensource.org/license/mit",
} as const;

export const SKILLS_ORG_URL = `https://github.com/${SKILLS_CATALOG.org}`;

export function skillRepoUrl(repo: string): string {
  return `${SKILLS_ORG_URL}/${repo}`;
}

export function skillPath(slug: string): string {
  return `${SKILLS_CATALOG.basePath}/${slug}`;
}
```

`siteUrl` and `siteName` are literals because they end up in JSON-LD and the sitemap at build time; an
environment variable unset on one build machine would publish `undefined` in both. `org` and `branch` are
repeated in the sync script, which runs under plain Node and does not import TypeScript.

## Categories

```ts
// file: lib/skills/categories.ts
import { SKILLS_CATALOG } from "./config";

/** The host's categories, in catalog order. Rename them here; meta.md files name them by id. */
export const SKILL_CATEGORIES = ["content", "scheduling", "infrastructure", "developer-workflow"] as const;

export type SkillCategory = (typeof SKILL_CATEGORIES)[number];

export interface SkillCategoryMeta {
  label: string;
  description: string;
}

export const SKILL_CATEGORY_META: Record<SkillCategory, SkillCategoryMeta> = {
  content: {
    label: "Content",
    description: "Blogs and help centers served from markdown in the repository.",
  },
  scheduling: { label: "Scheduling", description: "Booking pages, availability and calendar sync." },
  infrastructure: {
    label: "Infrastructure",
    description: "The layer in front of the app: access, gating, operations.",
  },
  "developer-workflow": {
    label: "Developer workflow",
    description: "Commits, releases and documentation for any repository.",
  },
};

export function isSkillCategory(value: unknown): value is SkillCategory {
  return typeof value === "string" && (SKILL_CATEGORIES as readonly string[]).includes(value);
}

/**
 * The `category` field of a meta.md: one id, or a list with the primary one
 * first. An unknown or missing id throws, so a typo fails the build instead of
 * hiding the skill from its category page.
 */
export function parseSkillCategories(value: unknown, source: string): [SkillCategory, ...SkillCategory[]] {
  const values: unknown[] = Array.isArray(value) ? value : [value];
  const categories: SkillCategory[] = [];
  for (const item of values) {
    if (!isSkillCategory(item)) throw new Error(`${source}: unknown category "${String(item)}"`);
    if (!categories.includes(item)) categories.push(item);
  }
  const [primary, ...rest] = categories;
  if (!primary) throw new Error(`${source}: no category`);
  return [primary, ...rest];
}

/** `/skills/<category>` shares the `[slug]` segment with the skill pages; testing.md keeps the two apart. */
export function skillCategoryPath(category: SkillCategory): string {
  return `${SKILLS_CATALOG.basePath}/${category}`;
}

/** Repository names are kebab-case. Checked before a slug from a URL or a query touches the filesystem. */
export const SKILL_SLUG_PATTERN = /^[a-z0-9][a-z0-9-]{0,63}$/;

export function isSkillSlugShape(value: unknown): value is string {
  return typeof value === "string" && SKILL_SLUG_PATTERN.test(value);
}
```

A category id is also a URL segment beside the skill slugs (`/skills/content` beside `/skills/ksef`), so
the validator fails a skill whose slug equals a category id; see [loader.md](loader.md).
`parseSkillCategories` returns a non-empty tuple so `categories[0]` is a category under
`noUncheckedIndexedAccess`, with no assertion.

## Types

```ts
// file: lib/skills/types.ts
import type { SkillCategory } from "./categories";
import type { EvalRun, SkillPrompt } from "./evals";
import type { SkillRule } from "./markdown";

export interface SkillNotThis {
  problem: string;
  instead: string;
}

export interface SkillFile {
  path: string;
  contents: string;
}

export interface SkillRelease {
  version: string;
  date: string;
  /** The release's opening paragraph, or its first item; plain text. */
  summary: string;
}

export interface SkillProvenanceItem {
  title: string;
  body: string;
}

export interface SkillProvenanceGroup {
  /** The h2 of provenance.md ("Fixed in the templates", "Kept deliberately", ...) or a release. */
  label: string;
  note: string;
  items: SkillProvenanceItem[];
}

export interface SkillProvenance {
  /** Where the ledger lives: its own file, or the changelog's Fixed sections. */
  source: "provenance.md" | "CHANGELOG.md";
  url: string;
  intro: string[];
  groups: SkillProvenanceGroup[];
}

export interface SkillSync {
  /** The commit every synced file was read at; null before the first sync. */
  commit: string | null;
  committedAt: string | null;
}

/** A module an agent builds into an app, or a tool an agent runs on something else. */
export type SkillKind = "module" | "tool";

export interface Skill {
  slug: string;
  repo: string;
  repoUrl: string;
  name: string;
  /** The first category meta.md lists: the card label and the JSON-LD category. */
  category: SkillCategory;
  /** Every category meta.md lists, the primary one first. */
  categories: SkillCategory[];
  kind: SkillKind;
  tagline: string;
  /** At most 155 characters; the tagline when meta.md has no shorter one. */
  seoDescription: string;
  platform: string;
  related: string[];
  /** The plain-language problem statement: the meta.md body, markdown. */
  problem: string;
  /** The SKILL.md frontmatter description, whitespace collapsed. */
  description: string;
  /** The index README's "Builds" cell split into parts; empty without an index. */
  features: string[];
  /** The index README's "Backend" cell; "" without an index. */
  backend: string;
  /** The README intro paragraph that says who wrote the skill and from what. */
  origin: string;
  version: string | null;
  releaseDate: string | null;
  /** The first release: when the skill was published. */
  firstReleaseDate: string | null;
  whenToUse: string[];
  whenNotToUse: string[];
  nonNegotiables: SkillRule[];
  notThis: SkillNotThis[];
  requirements: string[];
  files: SkillFile[];
  sources: string[];
  provenance: SkillProvenance;
  releases: SkillRelease[];
  prompts: SkillPrompt[];
  /** Every run that parsed, newest first. Rejected files are in the catalog's problems, not here. */
  evals: EvalRun[];
  sync: SkillSync;
}

/** What a card sees: no upstream prose, so a client component never receives a whole skill. */
export interface SkillSummary {
  slug: string;
  name: string;
  category: SkillCategory;
  categories: SkillCategory[];
  kind: SkillKind;
  tagline: string;
  backend: string;
  version: string | null;
  releaseDate: string | null;
}
```

`Skill` is the page's whole model and stays on the server. `SkillSummary` is the only shape a client
component receives: the catalog search filters summaries, and a related-skills list builds them with
`toSkillSummary`. `EvalRun` and `SkillPrompt` live in [evals.md](evals.md); `SkillRule` in
[parsing.md](parsing.md).

## Checklist

- [ ] `SKILLS_CATALOG.org`, `branch`, `siteUrl` and `siteName` are the host's values
- [ ] `SKILL_CATEGORIES` and `SKILL_CATEGORY_META` hold the host's categories, ids kebab-case
- [ ] No category id equals a skill slug
- [ ] Every skill folder has a `meta.md` with a category and a tagline
