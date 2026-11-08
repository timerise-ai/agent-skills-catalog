# Routes, metadata, JSON-LD, sitemap

| Route | Renders | Static params |
|---|---|---|
| `/skills` | The catalog: search, category filter, every card | none |
| `/skills/<category>` | One category: its description and its skills | every category at least one skill lists |
| `/skills/<slug>` | One skill | every skill slug |
| `/sitemap.xml` | The catalog, each category and each skill, dated from the changelog | none |

`/skills/<category>` and `/skills/<slug>` share one `[slug]` segment, so a category page sits beside the
skill pages in the URL and in the breadcrumb. The page decides which it is with `isSkillCategorySlug`, and
the validator fails a skill slug that equals a category id, which would otherwise make one page
unreachable. `dynamicParams = false` makes every page static and any other slug a 404 without running the
page; a skill added later appears with the next build that has its synced files. A host with
`cacheComponents` deletes that export: [adaptation.md](adaptation.md).

## SEO helpers

```ts
// file: lib/skills/seo.ts
import type { MetadataRoute } from "next";

import { getAllSkills, getSkillCategories } from "./catalog";
import { SKILL_CATEGORY_META, type SkillCategory, skillCategoryPath } from "./categories";
import { SKILLS_CATALOG, skillPath } from "./config";
import type { Skill } from "./types";

export function absoluteUrl(pathname: string): string {
  return `${SKILLS_CATALOG.siteUrl}${pathname}`;
}

export function skillPageTitle(skill: Skill): string {
  return `${skill.name}: ${skill.platform} Agent Skill`;
}

function organization() {
  return { "@type": "Organization", name: SKILLS_CATALOG.siteName, url: SKILLS_CATALOG.siteUrl };
}

/**
 * The page is a TechArticle about a SoftwareSourceCode: the repository,
 * version, license and runtime. Neither earns a rich result; the value is
 * attribution when an assistant or a search engine cites the page. Dates come
 * from the changelog, never from the build, so an unchanged skill keeps its
 * dateModified across deploys.
 */
export function skillJsonLd(skill: Skill) {
  const url = absoluteUrl(skillPath(skill.slug));
  return {
    "@context": "https://schema.org",
    "@type": "TechArticle",
    headline: skillPageTitle(skill),
    description: skill.seoDescription,
    inLanguage: "en",
    datePublished: skill.firstReleaseDate ?? undefined,
    dateModified: skill.releaseDate ?? undefined,
    mainEntityOfPage: { "@type": "WebPage", "@id": url },
    publisher: organization(),
    about: {
      "@type": "SoftwareSourceCode",
      name: skill.name,
      alternateName: skill.repo,
      description: skill.tagline,
      codeRepository: skill.repoUrl,
      version: skill.version ?? undefined,
      license: SKILLS_CATALOG.license,
      runtimePlatform: skill.platform,
      applicationCategory: SKILL_CATEGORY_META[skill.category].label,
      author: organization(),
    },
  };
}

/** Catalog, primary category, skill: the same trail the breadcrumb on the page shows. */
export function skillBreadcrumbJsonLd(skill: Skill) {
  const trail = [
    { name: "Agent Skills", path: SKILLS_CATALOG.basePath },
    { name: SKILL_CATEGORY_META[skill.category].label, path: skillCategoryPath(skill.category) },
    { name: skill.name, path: skillPath(skill.slug) },
  ];
  return {
    "@context": "https://schema.org",
    "@type": "BreadcrumbList",
    itemListElement: trail.map((step, index) => ({
      "@type": "ListItem",
      position: index + 1,
      name: step.name,
      item: absoluteUrl(step.path),
    })),
  };
}

export function categoryJsonLd(category: SkillCategory, skills: Skill[]) {
  const meta = SKILL_CATEGORY_META[category];
  return {
    "@context": "https://schema.org",
    "@type": "CollectionPage",
    name: `${meta.label} Agent Skills`,
    description: meta.description,
    url: absoluteUrl(skillCategoryPath(category)),
    mainEntity: {
      "@type": "ItemList",
      itemListElement: skills.map((skill, index) => ({
        "@type": "ListItem",
        position: index + 1,
        url: absoluteUrl(skillPath(skill.slug)),
        name: skill.name,
      })),
    },
  };
}

/**
 * A skill page changes when the skill releases, so its lastModified is the
 * release date; a category or the catalog changes when any of its skills does.
 */
export function skillSitemapEntries(): MetadataRoute.Sitemap {
  const skills = getAllSkills();
  const newest = (list: Skill[]) =>
    list
      .map((skill) => skill.releaseDate)
      .filter((date): date is string => date !== null)
      .sort()
      .at(-1);
  const entry = (pathname: string, date: string | undefined) => ({
    url: absoluteUrl(pathname),
    ...(date ? { lastModified: new Date(`${date}T00:00:00Z`) } : {}),
  });
  return [
    entry(SKILLS_CATALOG.basePath, newest(skills)),
    ...getSkillCategories().map((category) =>
      entry(
        skillCategoryPath(category),
        newest(skills.filter((skill) => skill.categories.includes(category))),
      ),
    ),
    ...skills.map((skill) => entry(skillPath(skill.slug), skill.releaseDate ?? undefined)),
  ];
}

/** JSON-LD inside a script tag: `<` escaped so text from a skill cannot close the tag. */
export function jsonLdScript(data: unknown): string {
  return JSON.stringify(data).replace(/</g, "\\u003c");
}
```

**Dates come from the changelog.** `datePublished` is the first release, `dateModified` and the sitemap's
`lastModified` the newest. A date from the build would change on every deploy and tell crawlers every skill
changed.

**JSON-LD is escaped.** `jsonLdScript` writes `<` as `<`, since the tagline and the description are
text from files an outsider can propose changes to, and a `</script>` in either would end the tag.

**TechArticle about a SoftwareSourceCode.** Neither earns a rich result; the value is a machine-readable
statement of the repository, version, license and runtime, so a search engine or an assistant citing the
page attributes it correctly. The category page is a `CollectionPage` with an `ItemList`.

## The catalog page

```tsx
// file: app/skills/page.tsx
import type { Metadata } from "next";

import { SkillCatalog } from "@/components/skills/SkillCatalog";
import { getAllSkills, getSkillCategories, toSkillSummary } from "@/lib/skills/catalog";
import { SKILLS_CATALOG } from "@/lib/skills/config";

export const metadata: Metadata = {
  title: `Agent Skills | ${SKILLS_CATALOG.siteName}`,
  description:
    "Agent Skills an agent installs and builds from, each with its rules, releases and agent eval results.",
  alternates: { canonical: SKILLS_CATALOG.basePath },
};

export default function SkillsPage() {
  const skills = getAllSkills().map(toSkillSummary);
  return (
    <main data-skills-page>
      <h1>Agent Skills</h1>
      <SkillCatalog skills={skills} categories={getSkillCategories()} />
    </main>
  );
}
```

## The skill and category page

```tsx
// file: app/skills/[slug]/page.tsx
import type { Metadata } from "next";
import Link from "next/link";
import { notFound } from "next/navigation";

import { EvalRuns } from "@/components/skills/EvalRuns";
import { ProvenanceLedger } from "@/components/skills/ProvenanceLedger";
import { SkillCard } from "@/components/skills/SkillCard";
import { SkillMarkdown } from "@/components/skills/SkillMarkdown";
import { SkillPrompts } from "@/components/skills/SkillPrompts";
import {
  getRelatedSkills,
  getSkill,
  getSkillCategories,
  getSkillSlugs,
  getSkillsInCategory,
  isSkillCategorySlug,
  isSkillSlug,
  toSkillSummary,
} from "@/lib/skills/catalog";
import { SKILL_CATEGORY_META, type SkillCategory, skillCategoryPath } from "@/lib/skills/categories";
import { SKILLS_CATALOG, skillPath } from "@/lib/skills/config";
import {
  categoryJsonLd,
  jsonLdScript,
  skillBreadcrumbJsonLd,
  skillJsonLd,
  skillPageTitle,
} from "@/lib/skills/seo";
import type { Skill } from "@/lib/skills/types";

type Props = { params: Promise<{ slug: string }> };

// Every page is known at build time; anything else is a 404 without running the page.
export const dynamicParams = false;

/** One segment serves a skill (`/skills/ksef`) and a category (`/skills/content`); see validate.ts. */
export function generateStaticParams() {
  return [...getSkillSlugs(), ...getSkillCategories()].map((slug) => ({ slug }));
}

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { slug } = await params;
  if (isSkillCategorySlug(slug)) {
    const meta = SKILL_CATEGORY_META[slug];
    return {
      title: `${meta.label} Agent Skills | ${SKILLS_CATALOG.siteName}`,
      description: meta.description,
      alternates: { canonical: skillCategoryPath(slug) },
    };
  }
  if (!isSkillSlug(slug)) return {};
  const skill = getSkill(slug);
  return {
    title: `${skillPageTitle(skill)} | ${SKILLS_CATALOG.siteName}`,
    description: skill.seoDescription,
    alternates: { canonical: skillPath(slug) },
  };
}

function CategoryPage({ category }: { category: SkillCategory }) {
  const meta = SKILL_CATEGORY_META[category];
  const skills = getSkillsInCategory(category);
  return (
    <main data-skills-category>
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: jsonLdScript(categoryJsonLd(category, skills)) }}
      />
      <nav aria-label="Breadcrumb">
        <Link href={SKILLS_CATALOG.basePath}>Agent Skills</Link>
      </nav>
      <h1>{meta.label} Agent Skills</h1>
      <p>{meta.description}</p>
      <ul data-skill-grid>
        {skills.map((skill) => (
          <li key={skill.slug}>
            <SkillCard skill={toSkillSummary(skill)} />
          </li>
        ))}
      </ul>
    </main>
  );
}

function SkillDetail({ skill }: { skill: Skill }) {
  const related = getRelatedSkills(skill);
  const md = (text: string, inline = false) => (
    <SkillMarkdown repoUrl={skill.repoUrl} inline={inline}>
      {text}
    </SkillMarkdown>
  );
  return (
    <main data-skill-page>
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: jsonLdScript(skillJsonLd(skill)) }}
      />
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: jsonLdScript(skillBreadcrumbJsonLd(skill)) }}
      />
      <nav aria-label="Breadcrumb">
        <ol>
          <li>
            <Link href={SKILLS_CATALOG.basePath}>Agent Skills</Link>
          </li>
          <li>
            <Link href={skillCategoryPath(skill.category)}>{SKILL_CATEGORY_META[skill.category].label}</Link>
          </li>
          <li aria-current="page">{skill.name}</li>
        </ol>
      </nav>

      <header>
        <h1>{skillPageTitle(skill)}</h1>
        <p>{skill.tagline}</p>
        <p data-skill-facts>
          {skill.version && <span>v{skill.version}</span>}
          {skill.releaseDate && <span>released {skill.releaseDate}</span>}
          {skill.backend && <span>{skill.backend}</span>}
          <a href={skill.repoUrl} rel="noopener noreferrer">
            Repository
          </a>
        </p>
      </header>

      <section id="problem" aria-labelledby="problem-title">
        <h2 id="problem-title">The problem</h2>
        {md(skill.problem)}
      </section>

      {skill.features.length > 0 && (
        <section id="builds" aria-labelledby="builds-title">
          <h2 id="builds-title">What it builds</h2>
          <ul>
            {skill.features.map((feature) => (
              <li key={feature}>{feature}</li>
            ))}
          </ul>
        </section>
      )}

      <section id="rules" aria-labelledby="rules-title">
        <h2 id="rules-title">The {skill.nonNegotiables.length} non-negotiables</h2>
        <ol>
          {skill.nonNegotiables.map((item) => (
            <li key={item.rule}>
              {/* A reason that opens with a comma continues the rule's sentence: no space before it. */}
              <strong>{md(item.rule, true)}</strong>
              {item.reason && /^[,;:]/.test(item.reason) ? "" : " "}
              {item.reason && md(item.reason, true)}
            </li>
          ))}
        </ol>
      </section>

      <section id="fit" aria-labelledby="fit-title">
        <h2 id="fit-title">When to use it</h2>
        <ul>
          {skill.whenToUse.map((item) => (
            <li key={item}>{md(item, true)}</li>
          ))}
        </ul>
        <h3>Not this</h3>
        <ul>
          {skill.notThis.length > 0
            ? skill.notThis.map((row) => (
                <li key={row.problem}>
                  {md(row.problem, true)}: {md(row.instead, true)}
                </li>
              ))
            : skill.whenNotToUse.map((item) => <li key={item}>{md(item, true)}</li>)}
        </ul>
      </section>

      <section id="install" aria-labelledby="install-title">
        <h2 id="install-title">Install</h2>
        <pre>
          <code>{`npx skills add ${SKILLS_CATALOG.org}/${skill.repo}`}</code>
        </pre>
        <p>
          Then invoke it with <code>/{skill.repo}</code> in Claude Code, <code>${skill.repo}</code> in Codex
          CLI, or from <code>/skills</code> in Gemini CLI.
        </p>
      </section>

      <section id="prompts" aria-labelledby="prompts-title">
        <h2 id="prompts-title">What do I tell my agent?</h2>
        <SkillPrompts prompts={skill.prompts} />
      </section>

      <section id="evals" aria-labelledby="evals-title">
        <h2 id="evals-title">How does it do in each agent?</h2>
        <EvalRuns runs={skill.evals} repoUrl={skill.repoUrl} />
      </section>

      {skill.provenance.groups.length > 0 && (
        <section id="provenance" aria-labelledby="provenance-title">
          <h2 id="provenance-title">Where the rules come from</h2>
          {skill.origin && md(skill.origin)}
          <ProvenanceLedger provenance={skill.provenance} repoUrl={skill.repoUrl} />
        </section>
      )}

      {skill.releases.length > 0 && (
        <section id="releases" aria-labelledby="releases-title">
          <h2 id="releases-title">Recent releases</h2>
          <ol>
            {skill.releases.slice(0, 3).map((release) => (
              <li key={release.version}>
                <strong>v{release.version}</strong>, {release.date}: {release.summary}
              </li>
            ))}
          </ol>
        </section>
      )}

      {related.length > 0 && (
        <section aria-labelledby="related-title">
          <h2 id="related-title">Related skills</h2>
          <ul data-skill-grid>
            {related.map((item) => (
              <li key={item.slug}>
                <SkillCard skill={item} />
              </li>
            ))}
          </ul>
        </section>
      )}

      {skill.sync.commit && (
        <p data-skill-sync>
          Generated from{" "}
          <a href={`${skill.repoUrl}/commit/${skill.sync.commit}`} rel="noopener noreferrer">
            {skill.sync.commit.slice(0, 7)}
          </a>
          {skill.sync.committedAt && `, committed ${skill.sync.committedAt.slice(0, 10)}`}.
        </p>
      )}
    </main>
  );
}

export default async function SkillOrCategoryPage({ params }: Props) {
  const { slug } = await params;
  if (isSkillCategorySlug(slug)) return <CategoryPage category={slug} />;
  if (!isSkillSlug(slug)) notFound();
  return <SkillDetail skill={getSkill(slug)} />;
}
```

The section order follows the questions a visitor asks: what problem it solves, what it builds, which rules
it never breaks, whether it fits, how to install it, what to tell the agent, how it did in each agent, and
where the rules come from. The install line uses the skills CLI (`npx skills add <org>/<repo>`) and names
the invocation per agent: `/<name>` in Claude Code, `$<name>` in Codex CLI, `/skills` in Gemini CLI.

## Sitemap

```ts
// file: app/sitemap.ts
import type { MetadataRoute } from "next";

import { skillSitemapEntries } from "@/lib/skills/seo";

export default function sitemap(): MetadataRoute.Sitemap {
  return [...skillSitemapEntries()];
}
```

A host with a sitemap already spreads `skillSitemapEntries()` into its own list.

## Checklist

- [ ] `npm run build` lists `/skills`, every skill and every used category as static pages
- [ ] Each skill page has one TechArticle and one BreadcrumbList script
- [ ] The sitemap dates equal the newest release dates
