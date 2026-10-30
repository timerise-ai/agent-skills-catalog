# Loader and validator

`lib/skills/catalog.ts` is the only module that reads `content/skills/`; every page, the sitemap and the
validator go through it. It never fetches. `lib/skills/validate.ts` lists everything the loader forgives,
so an author hears about it in `npm test` rather than from a missing card.

## The loader

```ts
// file: lib/skills/catalog.ts
import fs from "node:fs";
import path from "node:path";
import { cache } from "react";

import { parseSkillCategories, SKILL_CATEGORIES, type SkillCategory } from "./categories";
import { SKILLS_CATALOG, skillRepoUrl } from "./config";
import { type EvalParse, parseEval, parsePromptsFile, PROMPTS_FILE, sortRuns } from "./evals";
import { asString, asStringArray, readFrontmatter } from "./frontmatter";
import { SKILL_LAYOUT } from "./layout";
import {
  blockquoteItems,
  findSection,
  listItems,
  listOrParagraphs,
  paragraphs,
  splitRule,
  splitSections,
  stripInline,
  tableRows,
} from "./markdown";
import type { Skill, SkillKind, SkillSummary, SkillSync } from "./types";
import {
  featuresFromBuilds,
  type IndexEntry,
  parseIndexTable,
  parseReleases,
  provenanceFromChangelog,
  provenanceFromFile,
} from "./upstream";

/**
 * The catalog, read at build time from `content/skills/`: per skill a
 * `meta.md` the host writes and an `upstream/` folder that only
 * `npm run skills:sync` writes. Nothing here fetches; nothing on a page is
 * retyped from a skill.
 */

// A literal, equal to SKILLS_CATALOG.contentDir: the build's file tracer scopes to a literal folder, and
// a path read from a variable makes it trace the whole project.
const root = path.join(process.cwd(), "content", "skills");

function readIfExists(file: string): string | null {
  return fs.existsSync(file) ? fs.readFileSync(file, "utf8") : null;
}

function upstreamPath(slug: string, file: string): string {
  return path.join(root, slug, "upstream", file);
}

/** A skill has a page once it has both halves: its meta.md and a synced SKILL.md. */
export const getSkillSlugs = cache((): string[] => {
  if (!fs.existsSync(root)) return [];
  return fs
    .readdirSync(root, { withFileTypes: true })
    .filter(
      (entry) =>
        entry.isDirectory() &&
        !entry.name.startsWith("_") &&
        fs.existsSync(path.join(root, entry.name, "meta.md")) &&
        fs.existsSync(upstreamPath(entry.name, "SKILL.md")),
    )
    .map((entry) => entry.name)
    .sort();
});

export function isSkillSlug(slug: string): boolean {
  return getSkillSlugs().includes(slug);
}

const getIndex = cache((): Map<string, IndexEntry> => {
  const readme = readIfExists(path.join(root, "_index", "README.md"));
  return readme ? parseIndexTable(readme) : new Map();
});

/** The commit the last sync read every file at; a malformed sync.json only loses the "as of" line. */
function readSync(slug: string): SkillSync {
  const raw = readIfExists(upstreamPath(slug, "sync.json"));
  try {
    const parsed = raw ? (JSON.parse(raw) as { commit?: { sha?: unknown; date?: unknown } | null }) : null;
    return {
      commit: asString(parsed?.commit?.sha) || null,
      committedAt: asString(parsed?.commit?.date) || null,
    };
  } catch {
    return { commit: null, committedAt: null };
  }
}

/** Every result file in upstream/evals/, parsed or rejected with its reason. */
export const readEvalFiles = cache((slug: string): EvalParse[] => {
  const dir = upstreamPath(slug, "evals");
  if (!fs.existsSync(dir)) return [];
  return fs
    .readdirSync(dir)
    .filter((name) => name.endsWith(".md") && name !== PROMPTS_FILE)
    .sort()
    .map((name) => parseEval(name, fs.readFileSync(path.join(dir, name), "utf8")));
});

function skillKind(value: unknown, source: string): SkillKind {
  if (value === undefined || value === null || value === "module") return "module";
  if (value === "tool") return "tool";
  throw new Error(`${source}: unknown kind "${String(value)}"`);
}

export const getSkill = cache((slug: string): Skill => {
  const metaFile = `${SKILLS_CATALOG.contentDir}/${slug}/meta.md`;
  const meta = readFrontmatter(fs.readFileSync(path.join(root, slug, "meta.md"), "utf8"));
  const fm = meta.data;

  const skillMd = readFrontmatter(readIfExists(upstreamPath(slug, "SKILL.md")) ?? "");
  const readme = readIfExists(upstreamPath(slug, "README.md")) ?? "";
  const changelog = readIfExists(upstreamPath(slug, "CHANGELOG.md")) ?? "";
  const provenanceMd = readIfExists(upstreamPath(slug, SKILL_LAYOUT.provenanceFile));
  const prompts = readIfExists(upstreamPath(slug, `evals/${PROMPTS_FILE}`));

  const repo = asString(fm.repo) || slug;
  const repoUrl = skillRepoUrl(repo);
  const categories = parseSkillCategories(fm.category, metaFile);
  const index = getIndex().get(repo);
  const releases = parseReleases(changelog);
  const skillSections = splitSections(skillMd.content);
  const readmeSections = splitSections(readme);

  const readmeRules = listItems(findSection(readmeSections, SKILL_LAYOUT.readme.nonNegotiables));
  const hardRulesBlock = findSection(skillSections, SKILL_LAYOUT.skill.hardRules);
  const hardRules = listItems(hardRulesBlock);
  const rules =
    readmeRules.length > 0 ? readmeRules : hardRules.length > 0 ? hardRules : blockquoteItems(hardRulesBlock);
  const tagline = asString(fm.tagline);

  return {
    slug,
    repo,
    repoUrl,
    name: asString(fm.name) || repo,
    category: categories[0],
    categories,
    kind: skillKind(fm.kind, metaFile),
    tagline,
    seoDescription: asString(fm.seoDescription) || tagline,
    platform: asString(fm.platform) || SKILLS_CATALOG.defaultPlatform,
    related: asStringArray(fm.related),
    problem: meta.content.trim(),
    description: asString(skillMd.data.description).replace(/\s+/g, " "),
    features: featuresFromBuilds(index?.builds ?? ""),
    backend: stripInline(index?.backend ?? ""),
    origin: paragraphs(readmeSections[0]?.body ?? "").find((p) => SKILL_LAYOUT.readme.origin.test(p)) ?? "",
    version: releases[0]?.version ?? null,
    releaseDate: releases[0]?.date ?? null,
    firstReleaseDate: releases.at(-1)?.date ?? null,
    whenToUse: listOrParagraphs(findSection(skillSections, SKILL_LAYOUT.skill.whenToUse)),
    whenNotToUse: listOrParagraphs(findSection(skillSections, SKILL_LAYOUT.skill.whenNotToUse)),
    nonNegotiables: rules.map(splitRule),
    notThis: tableRows(findSection(readmeSections, SKILL_LAYOUT.readme.notThis)).flatMap(
      ([problem, instead]) => (problem && instead ? [{ problem, instead }] : []),
    ),
    requirements: listItems(findSection(readmeSections, SKILL_LAYOUT.readme.requirements)),
    files: tableRows(findSection(readmeSections, SKILL_LAYOUT.readme.files)).flatMap(([file, contents]) =>
      file && contents ? [{ path: stripInline(file), contents }] : [],
    ),
    sources: paragraphs(findSection(skillSections, SKILL_LAYOUT.skill.sources)),
    provenance: provenanceMd
      ? provenanceFromFile(
          provenanceMd,
          `${repoUrl}/blob/${SKILLS_CATALOG.branch}/${SKILL_LAYOUT.provenanceFile}`,
        )
      : provenanceFromChangelog(changelog, `${repoUrl}/blob/${SKILLS_CATALOG.branch}/CHANGELOG.md`),
    releases,
    prompts: prompts ? parsePromptsFile(prompts) : [],
    evals: sortRuns(readEvalFiles(slug).flatMap((parsed) => (parsed.ok ? [parsed.run] : []))),
    sync: readSync(slug),
  };
});

export function toSkillSummary(skill: Skill): SkillSummary {
  return {
    slug: skill.slug,
    name: skill.name,
    category: skill.category,
    categories: skill.categories,
    kind: skill.kind,
    tagline: skill.tagline,
    backend: skill.backend,
    version: skill.version,
    releaseDate: skill.releaseDate,
  };
}

/** Every skill with a page, by name. A host with its own order sorts here and nowhere else. */
export function getAllSkills(): Skill[] {
  return getSkillSlugs()
    .map((slug) => getSkill(slug))
    .sort((a, b) => a.name.localeCompare(b.name));
}

/** The categories at least one skill lists, in catalog order: one page each, none empty. */
export function getSkillCategories(): SkillCategory[] {
  const used = new Set(getAllSkills().flatMap((skill) => skill.categories));
  return SKILL_CATEGORIES.filter((category) => used.has(category));
}

export function isSkillCategorySlug(slug: string): slug is SkillCategory {
  return (getSkillCategories() as string[]).includes(slug);
}

export function getSkillsInCategory(category: SkillCategory): Skill[] {
  return getAllSkills().filter((skill) => skill.categories.includes(category));
}

/** Related skills that have a page, in the order meta.md lists them. */
export function getRelatedSkills(skill: Skill): SkillSummary[] {
  const slugs = new Set(getSkillSlugs());
  return skill.related
    .filter((slug) => slug !== skill.slug && slugs.has(slug))
    .map((slug) => toSkillSummary(getSkill(slug)));
}
```

**React `cache()` makes one read per build worker.** The catalog page, every `[slug]` page, the metadata
and the sitemap call `getAllSkills()` or `getSkill(slug)`; `cache()` memoises per request on the server,
and in tests and scripts it is a plain call. Each function is still pure over the files.

**The content path is a literal.** Next.js traces the files a server module reads; `path.join` with a
variable makes Turbopack warn that the whole project was traced, while a literal folder scopes the trace to
`content/skills`. `SKILLS_CATALOG.contentDir` names the same folder for messages and must stay equal.

**Problems the runtime forgives:**

| Situation | The page | The validator |
|---|---|---|
| A category id in `meta.md` that does not exist | Build fails, the file named | (never reached) |
| An unknown `kind` | Build fails, the file named | (never reached) |
| An eval file that does not parse | The run is absent | Error, with the reason |
| `prompts.md` with fewer than two prompts | Shows what it has | Error |
| No released version in the changelog | No version shown | Error |
| No frontmatter description, no tagline | Empty text | Error |
| `seoDescription` past 155 characters | Search engines cut it | Error |
| No non-negotiables in either file | Empty rules section | Error |
| A skill slug equal to a category id | One page hides the other | Error |
| `result` disagreeing with the checks | Recorded result shown | Warning |
| A prompt reworded after a run | Run still shown | Warning |
| No When to use, no What's inside table | Section empty | Warning |
| `sync.json` without a commit | No "generated from" line | Warning |
| `related` naming a slug with no page | Left out | Warning |

## The validator

```ts
// file: lib/skills/validate.ts
import { SKILL_CATEGORIES } from "./categories";
import { getAllSkills, getSkillSlugs, readEvalFiles } from "./catalog";
import { impliedResult } from "./evals";

export interface CatalogProblem {
  level: "error" | "warning";
  slug: string;
  message: string;
}

/** The longest description a search result shows whole. */
export const SEO_DESCRIPTION_MAX = 155;

/**
 * Everything the runtime forgives and an author must hear about. The loader
 * skips what it cannot show; this lists it, and the test suite fails on any
 * error, so a file nobody can see is a red build rather than a missing card.
 */
export function validateCatalog(): CatalogProblem[] {
  const problems: CatalogProblem[] = [];
  const error = (slug: string, message: string) => problems.push({ level: "error", slug, message });
  const warning = (slug: string, message: string) => problems.push({ level: "warning", slug, message });
  const slugs = new Set(getSkillSlugs());

  for (const category of SKILL_CATEGORIES) {
    if (slugs.has(category))
      error(category, "a skill slug is also a category id; one page would hide the other");
  }

  for (const skill of getAllSkills()) {
    const { slug } = skill;
    for (const parsed of readEvalFiles(slug)) {
      if (!parsed.ok) error(slug, `evals/${parsed.file}: ${parsed.reason}`);
    }
    for (const run of skill.evals) {
      const implied = impliedResult(run);
      if (implied !== run.result) {
        warning(slug, `evals/${run.file}: result is ${run.result}, the checks say ${implied}`);
      }
      const prompt = skill.prompts[run.promptIndex - 1];
      if (prompt && prompt.prompt !== run.prompt) {
        warning(slug, `evals/${run.file}: prompt ${run.promptIndex} was reworded after this run`);
      }
    }
    if (skill.prompts.length < 2) error(slug, "evals/prompts.md has fewer than two prompts");
    if (!skill.version) error(slug, "CHANGELOG.md has no released version");
    if (!skill.description) error(slug, "SKILL.md has no frontmatter description");
    if (!skill.tagline) error(slug, "meta.md has no tagline");
    if (skill.seoDescription.length > SEO_DESCRIPTION_MAX) {
      error(slug, `seoDescription is ${skill.seoDescription.length} characters, over ${SEO_DESCRIPTION_MAX}`);
    }
    if (skill.nonNegotiables.length === 0)
      error(slug, "no non-negotiables in README.md or hard rules in SKILL.md");
    if (skill.whenToUse.length === 0) warning(slug, "SKILL.md has no When to use section");
    if (skill.files.length === 0) warning(slug, "README.md has no What's inside table");
    if (!skill.sync.commit) warning(slug, "upstream/sync.json records no commit");
    for (const related of skill.related) {
      if (!slugs.has(related)) warning(slug, `related skill "${related}" has no page`);
    }
  }
  return problems;
}
```

The errors fail `catalog.test.ts` ([testing.md](testing.md)); the warnings are printed by the same suite and
fail nothing. A host with a CI step before the build may also run the suite there, so a sync that brings in
a malformed run fails before deploy.

## Checklist

- [ ] Nothing outside `catalog.ts` imports `node:fs` for skills
- [ ] `npm test` fails on a malformed eval file (try one: a run with no `prompt:`)
- [ ] `npm run build` shows no "whole project was traced" warning
