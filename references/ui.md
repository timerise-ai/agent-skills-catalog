# UI: markdown, cards, catalog search, eval cards, ledger

The components ship structure and semantics: headings, lists, landmarks, `aria-*`, and `data-skill-*`
attributes as styling hooks. They carry no colours, spacing or fonts; the host styles the hooks or swaps
the plain elements for its own primitives, keeping the structure.

| Component | Server or client | Receives |
|---|---|---|
| `SkillMarkdown` | Server | A markdown string and the repository URL |
| `SkillCard` | Either | One `SkillSummary` |
| `SkillCatalog` | Client | `SkillSummary[]` and the used categories |
| `EvalRuns` | Server | `EvalRun[]` and the repository URL |
| `SkillPrompts` | Server | `SkillPrompt[]` |
| `ProvenanceLedger` | Server | `SkillProvenance` and the repository URL |

## Markdown from a skill

```tsx
// file: components/skills/SkillMarkdown.tsx
import type { ReactNode } from "react";
import ReactMarkdown, { defaultUrlTransform } from "react-markdown";
import remarkGfm from "remark-gfm";

import { SKILLS_CATALOG } from "@/lib/skills/config";

/**
 * Upstream markdown links relative to its own repository (`references/x.md`,
 * `evals/prompts.md`). On this site those paths do not exist, so every
 * relative link resolves against the file's folder on GitHub; absolute and
 * in-page links pass through.
 */
export function repoUrlTransform(repoUrl: string, folder = "") {
  const base = `${repoUrl}/blob/${SKILLS_CATALOG.branch}/${folder ? `${folder.replace(/\/$/, "")}/` : ""}`;
  return (url: string): string => {
    if (/^[a-z][a-z0-9+.-]*:/i.test(url) || url.startsWith("#") || url.startsWith("/")) {
      return defaultUrlTransform(url);
    }
    return defaultUrlTransform(new URL(url, base).toString());
  };
}

interface SkillMarkdownProps {
  children: string;
  repoUrl: string;
  /** The folder of the file the text came from, e.g. "evals"; "" for the repository root. */
  folder?: string;
  /** Render inline: no wrapping paragraph, for a rule or a table cell. */
  inline?: boolean;
}

export function SkillMarkdown({ children, repoUrl, folder, inline = false }: SkillMarkdownProps) {
  return (
    <ReactMarkdown
      remarkPlugins={[remarkGfm]}
      urlTransform={repoUrlTransform(repoUrl, folder)}
      components={inline ? { p: ({ children: inner }: { children?: ReactNode }) => <>{inner}</> } : undefined}
    >
      {children}
    </ReactMarkdown>
  );
}
```

**Relative links point into the repository.** A skill's README links `references/adaptation.md`, its
ledger links `ui.md`, a run's notes link `prompts.md`. Rendered on the site, those resolve against the
site's own URL and 404. `repoUrlTransform` resolves each against the folder of the file the text came from,
on GitHub, then passes the result through react-markdown's `defaultUrlTransform`, which drops unsafe
schemes such as `javascript:`. A host with its own renderer keeps this transform.

## Card

```tsx
// file: components/skills/SkillCard.tsx
import Link from "next/link";

import { SKILL_CATEGORY_META } from "@/lib/skills/categories";
import { skillPath } from "@/lib/skills/config";
import type { SkillSummary } from "@/lib/skills/types";

/** The card links the whole tile; the category is a label here, the filter is the catalog's. */
export function SkillCard({ skill }: { skill: SkillSummary }) {
  return (
    <article data-skill-card>
      <p data-skill-card-category>{SKILL_CATEGORY_META[skill.category].label}</p>
      <h3>
        <Link href={skillPath(skill.slug)}>{skill.name}</Link>
      </h3>
      <p>{skill.tagline}</p>
      <p data-skill-card-meta>
        {skill.version ? <span>v{skill.version}</span> : <span>Unreleased</span>}
        {skill.backend && <span>{skill.backend}</span>}
      </p>
    </article>
  );
}
```

## Catalog search and filter

```tsx
// file: components/skills/SkillCatalog.tsx
"use client";

import { useMemo, useState } from "react";

import { SKILL_CATEGORY_META, type SkillCategory } from "@/lib/skills/categories";
import type { SkillSummary } from "@/lib/skills/types";

import { SkillCard } from "./SkillCard";

/** Lowercased, accents folded, punctuation as space: a query matches however it is typed. */
function fold(text: string): string {
  return text
    .normalize("NFD")
    .replace(/[\u0300-\u036f]/g, "")
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, " ")
    .trim();
}

interface SkillCatalogProps {
  /** Summaries only: a whole Skill in a client component ships every README to every visitor. */
  skills: SkillSummary[];
  categories: SkillCategory[];
}

export function SkillCatalog({ skills, categories }: SkillCatalogProps) {
  const [query, setQuery] = useState("");
  const [category, setCategory] = useState<SkillCategory | null>(null);

  const haystacks = useMemo(
    () =>
      new Map(skills.map((skill) => [skill.slug, fold(`${skill.name} ${skill.tagline} ${skill.backend}`)])),
    [skills],
  );
  const words = fold(query).split(" ").filter(Boolean);
  const shown = skills.filter(
    (skill) =>
      (category === null || skill.categories.includes(category)) &&
      words.every((word) => haystacks.get(skill.slug)?.includes(word)),
  );

  return (
    <section data-skill-catalog aria-label="Skills">
      <div data-skill-catalog-controls>
        <label>
          <span>Search skills</span>
          <input type="search" value={query} onChange={(event) => setQuery(event.target.value)} />
        </label>
        <div role="group" aria-label="Category">
          <button type="button" aria-pressed={category === null} onClick={() => setCategory(null)}>
            All
          </button>
          {categories.map((id) => (
            <button key={id} type="button" aria-pressed={category === id} onClick={() => setCategory(id)}>
              {SKILL_CATEGORY_META[id].label}
            </button>
          ))}
        </div>
      </div>
      <p aria-live="polite" data-skill-catalog-count>
        {shown.length === skills.length
          ? `${skills.length} skills`
          : `${shown.length} of ${skills.length} skills`}
      </p>
      {shown.length > 0 ? (
        <ul data-skill-grid>
          {shown.map((skill) => (
            <li key={skill.slug}>
              <SkillCard skill={skill} />
            </li>
          ))}
        </ul>
      ) : (
        <p>No skill matches. Clear the search or pick another category.</p>
      )}
    </section>
  );
}
```

**Summaries, not skills.** The catalog is the one client component and it receives `SkillSummary[]`: name,
tagline, backend, categories, version. Passing `Skill[]` would serialise every README, ledger and eval note
into the page for every visitor.

**Search folds the query.** Lowercase, accents removed, punctuation as space, every word required: "Help
center", "help-center" and "help center " find the same card. The count is announced with `aria-live`, and
the category buttons are toggles with `aria-pressed`.

## Eval cards

```tsx
// file: components/skills/EvalRuns.tsx
import { SKILLS_CATALOG } from "@/lib/skills/config";
import {
  type CheckResult,
  EVAL_AGENTS,
  type EvalResult,
  type EvalRun,
  latestRuns,
  untriedAgents,
} from "@/lib/skills/evals";

import { SkillMarkdown } from "./SkillMarkdown";

/** What each recorded result means, in words a visitor reads without the standard open. */
export const RESULT_LABELS: Record<EvalResult, string> = {
  pass: "Built, checks pass",
  partial: "Builds, tests fail",
  fail: "Did not build",
};

const CHECK_LABELS: Record<CheckResult, string> = { pass: "passed", fail: "failed", none: "not run" };

function formatDate(value: string): string {
  // UTC, or a date written as 2026-09-26 renders as the 25th west of Greenwich.
  return new Date(`${value}T00:00:00Z`).toLocaleDateString("en-US", {
    month: "short",
    day: "numeric",
    year: "numeric",
    timeZone: "UTC",
  });
}

function Check({ label, value }: { label: string; value: CheckResult }) {
  return (
    <li data-check={value}>
      {label}: {CHECK_LABELS[value]}
    </li>
  );
}

/**
 * The newest run per agent and prompt, each card stating what was measured
 * and nothing more. A failing run is shown like a passing one; an agent with
 * no run is named, not left out.
 */
export function EvalRuns({ runs, repoUrl }: { runs: EvalRun[]; repoUrl: string }) {
  const latest = latestRuns(runs);
  const untried = untriedAgents(runs).map((agent) => EVAL_AGENTS[agent]);

  return (
    <div data-skill-evals>
      {latest.length === 0 && <p>No agent eval has been run on this skill yet.</p>}
      <ul>
        {latest.map((run) => (
          <li key={run.file} data-eval-result={run.result}>
            <header>
              <p>
                <strong>{EVAL_AGENTS[run.agent]}</strong>
                {run.agentVersion && <span> {run.agentVersion}</span>}
                {run.model && <span> on {run.model}</span>}
              </p>
              <p data-eval-badge>{RESULT_LABELS[run.result]}</p>
            </header>
            <blockquote>{run.prompt}</blockquote>
            <ul aria-label="Checks">
              <Check label="Typecheck" value={run.checks.typecheck} />
              <Check label="Build" value={run.checks.build} />
              <Check label="Tests" value={run.checks.tests} />
            </ul>
            <dl>
              {run.durationMinutes !== null && (
                <>
                  <dt>Time</dt>
                  <dd>
                    {run.durationMinutes < 1 ? "under 1" : run.durationMinutes} min
                    {run.timedOut && ", stopped at the limit"}
                  </dd>
                </>
              )}
              {run.filesChanged !== null && (
                <>
                  <dt>Changed</dt>
                  <dd>
                    {run.filesChanged} files
                    {run.linesAdded !== null && `, ${run.linesAdded.toLocaleString("en-US")} lines added`}
                  </dd>
                </>
              )}
              {run.interventions !== null && run.interventions > 0 && (
                <>
                  <dt>Nudged</dt>
                  <dd>
                    {run.interventions} {run.interventions === 1 ? "time" : "times"}
                  </dd>
                </>
              )}
              {run.stack && (
                <>
                  <dt>Stack</dt>
                  <dd>{run.stack}</dd>
                </>
              )}
              {run.skillVersion && (
                <>
                  <dt>Skill</dt>
                  <dd>v{run.skillVersion}</dd>
                </>
              )}
              <dt>Run</dt>
              <dd>{formatDate(run.date)}</dd>
            </dl>
            <p>
              <a
                href={`${repoUrl}/blob/${SKILLS_CATALOG.branch}/evals/${run.file}`}
                rel="noopener noreferrer"
              >
                Result file
              </a>
              {run.runUrl && (
                <>
                  {" "}
                  <a href={run.runUrl} rel="noopener noreferrer">
                    Agent log
                  </a>
                </>
              )}
            </p>
            {run.notes && (
              <section aria-label="What the person who ran it saw">
                <SkillMarkdown repoUrl={repoUrl} folder="evals">
                  {run.notes}
                </SkillMarkdown>
              </section>
            )}
            {!run.isolated && <p>Run with the tester's own agent setup, not a clean profile.</p>}
          </li>
        ))}
      </ul>
      {untried.length > 0 && <p>Not run yet in {untried.join(" or ")}.</p>}
    </div>
  );
}
```

What a card must show: the agent and its version, the model, the recorded result in words, the prompt
verbatim, each check, the time, the size of the change, the stack and the skill version, the date, a link
to the result file and, when present, to the agent log, and the notes. What it must not do: recompute the
result, hide a failing run, or round a check into a pass. A run made with the tester's own setup says so.

**Dates are formatted in UTC.** `new Date("2026-09-26")` is midnight UTC, and formatting it in a time zone
west of Greenwich prints the 25th.

## Prompts

```tsx
// file: components/skills/SkillPrompts.tsx
import type { SkillPrompt } from "@/lib/skills/evals";

/** The prompts.md list, numbered as the result files number them, so "prompt 1" means one thing on both. */
export function SkillPrompts({ prompts }: { prompts: SkillPrompt[] }) {
  if (prompts.length === 0) return null;
  return (
    <ol data-skill-prompts>
      {prompts.map((item) => (
        <li key={item.prompt}>
          <blockquote>{item.prompt}</blockquote>
          {item.stack && <p>Stack: {item.stack}</p>}
        </li>
      ))}
    </ol>
  );
}
```

## Provenance ledger

```tsx
// file: components/skills/ProvenanceLedger.tsx
import type { SkillProvenance } from "@/lib/skills/types";

import { SkillMarkdown } from "./SkillMarkdown";

/** The skill's own ledger, grouped as it groups itself, with a link to the whole file. */
export function ProvenanceLedger({ provenance, repoUrl }: { provenance: SkillProvenance; repoUrl: string }) {
  if (provenance.groups.length === 0) return null;
  const folder = provenance.source === "provenance.md" ? "references" : "";
  return (
    <div data-skill-provenance>
      {provenance.intro.slice(0, 1).map((paragraph) => (
        <SkillMarkdown key={paragraph} repoUrl={repoUrl} folder={folder}>
          {paragraph}
        </SkillMarkdown>
      ))}
      {provenance.groups.map((group) => (
        <section key={group.label}>
          <h3>{group.label}</h3>
          {group.note && (
            <SkillMarkdown repoUrl={repoUrl} folder={folder}>
              {group.note}
            </SkillMarkdown>
          )}
          <ol>
            {group.items.map((item) => (
              <li key={item.title}>
                <p>
                  <strong>{item.title}</strong>
                </p>
                {item.body && (
                  <SkillMarkdown repoUrl={repoUrl} folder={folder}>
                    {item.body}
                  </SkillMarkdown>
                )}
              </li>
            ))}
          </ol>
        </section>
      ))}
      <p>
        <a href={provenance.url} rel="noopener noreferrer">
          The whole ledger in {provenance.source}
        </a>
      </p>
    </div>
  );
}
```

## Style hooks

| Hook | On |
|---|---|
| `data-skills-page`, `data-skills-category`, `data-skill-page` | The three page roots |
| `data-skill-catalog`, `data-skill-catalog-controls`, `data-skill-catalog-count` | The catalog |
| `data-skill-grid` | Every list of cards |
| `data-skill-card`, `data-skill-card-category`, `data-skill-card-meta` | A card |
| `data-skill-facts`, `data-skill-sync` | The skill page header facts and the "generated from" line |
| `data-skill-evals`, `data-eval-result="pass\|partial\|fail"`, `data-eval-badge`, `data-check` | Eval cards |
| `data-skill-prompts`, `data-skill-provenance` | Prompts and the ledger |

Colour the result badge by `data-eval-result` in the host's own palette; the label text carries the
meaning, so the colour is never the only signal.

## Checklist

- [ ] No component outside `SkillCatalog` has `"use client"`
- [ ] A relative link in a README renders as a GitHub URL
- [ ] Every hook above is styled or deliberately left plain
