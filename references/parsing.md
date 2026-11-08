# Parsing: frontmatter, the layout, markdown readers, upstream files

The loader reads a skill's own markdown and shows parts of it. It does that with small readers, each for
one shape the shared skill layout guarantees, rather than a markdown parser and a guess. A section that is
not in the expected shape comes back empty, and the validator in [loader.md](loader.md) says so.

## Frontmatter

```ts
// file: lib/skills/frontmatter.ts
import matter from "gray-matter";

export type FrontmatterData = Record<string, unknown>;

/**
 * YAML reads an unquoted `date: 2026-09-26` as a Date. Every field the catalog
 * reads is text, and a Date read as text is "", which drops an eval run, so
 * dates go back to the string the author wrote, with a time only when one was
 * written.
 */
function normalize(value: unknown): unknown {
  if (value instanceof Date) {
    const iso = value.toISOString();
    return iso.endsWith("T00:00:00.000Z") ? iso.slice(0, 10) : iso;
  }
  if (Array.isArray(value)) return value.map(normalize);
  if (value !== null && typeof value === "object") {
    return Object.fromEntries(Object.entries(value).map(([key, item]) => [key, normalize(item)]));
  }
  return value;
}

export function readFrontmatter(source: string): { data: FrontmatterData; content: string } {
  const parsed = matter(source);
  return { data: normalize(parsed.data) as FrontmatterData, content: parsed.content };
}

export function asString(value: unknown): string {
  if (typeof value === "string") return value.trim();
  return typeof value === "number" ? String(value) : "";
}

export function asNumber(value: unknown): number | null {
  return typeof value === "number" && Number.isFinite(value) ? value : null;
}

export function asStringArray(value: unknown): string[] {
  return Array.isArray(value) ? value.filter((item): item is string => typeof item === "string") : [];
}
```

**YAML turns an unquoted date into a Date.** `date: 2026-09-26` in an eval run is read by `gray-matter`
(js-yaml, YAML 1.1 timestamps) as a `Date`, and `asString` of a `Date` is `""`, which rejects the run as
having no date. Normalizing in the one reader every file goes through makes the date the string the author
wrote. Nested objects are normalized too, since `checks:` is one. Verified by `evals.test.ts`, whose fixture
writes the date unquoted.

A file whose YAML does not parse throws from `readFrontmatter`: for `meta.md` that fails the build with the
file named, for an eval run `parseEval` catches it and returns the reason.

## The layout

```ts
// file: lib/skills/layout.ts
/**
 * The section headings the loader reads, by file. The defaults follow the
 * layout every skill of one organization shares (fixed headings, rules as a
 * numbered list with the rule in bold, one table each for "Not this" and
 * "What's inside"). A skill that strays from it loses that section on the
 * page and the validator reports it; nothing is guessed.
 */
export const SKILL_LAYOUT = {
  skill: {
    whenToUse: /^When to use\b/i,
    whenNotToUse: /^When NOT to use\b/i,
    hardRules: /^Hard rules\b/i,
    sources: /^Sources\b/i,
  },
  readme: {
    nonNegotiables: /non-negotiables/i,
    notThis: /^Not this\b/i,
    requirements: /^Requirements\b/i,
    files: /^What'?s inside\b/i,
    /** The intro paragraph that says who wrote the skill and from what. */
    origin: /\bwritten by\b/i,
  },
  provenanceFile: "references/provenance.md",
} as const;
```

These are the headings of the shared skill layout: `SKILL.md` with When to use, When NOT to use, Hard rules
and an optional Sources; `README.md` with an intro paragraph saying who wrote the skill, a numbered
non-negotiables list, an optional Requirements list, a What's inside table and a Not this table. A host
whose skills use other headings changes the patterns here and nowhere else.

| Page part | File | Read as |
|---|---|---|
| Description | `SKILL.md` frontmatter | `description`, whitespace collapsed |
| When to use, When NOT to use | `SKILL.md` | List items, or paragraphs when written as prose |
| The non-negotiables | `README.md` non-negotiables list; else `SKILL.md` hard rules as a list; else as blockquotes | `splitRule` on each item |
| Not this | `README.md` | First table, two columns |
| What's inside | `README.md` | First table: path, contents |
| Requirements | `README.md` | List items |
| Origin | `README.md` intro | The first paragraph matching `origin` |
| Sources | `SKILL.md` | Paragraphs |
| Releases | `CHANGELOG.md` | `## [x.y.z] - YYYY-MM-DD` blocks |
| Ledger | `references/provenance.md`, else `CHANGELOG.md` | See below |

## Markdown readers

```ts
// file: lib/skills/markdown.ts
/**
 * Small readers for the fixed layout of a skill's markdown. Not a markdown
 * parser: each one reads one shape the layout guarantees, and returns nothing
 * rather than a guess when the shape is not there.
 */

export interface Section {
  title: string;
  body: string;
}

const FENCE = /^\s*(```|~~~)/;
const ITEM_START = /^(?:[-*]|\d+\.)\s+/;

/** Splits on `## ` headings outside code fences; the chunk before the first heading has an empty title. */
export function splitSections(markdown: string): Section[] {
  const sections: Section[] = [];
  let current: Section = { title: "", body: "" };
  let inFence = false;

  for (const line of markdown.split("\n")) {
    if (FENCE.test(line)) inFence = !inFence;
    if (!inFence && line.startsWith("## ")) {
      sections.push(current);
      current = { title: line.slice(3).trim(), body: "" };
      continue;
    }
    current.body += `${line}\n`;
  }
  sections.push(current);
  return sections;
}

export function findSection(sections: Section[], pattern: RegExp): string {
  return sections.find((section) => pattern.test(section.title))?.body ?? "";
}

/**
 * The items of the first list in a block, continuation lines joined. Skills
 * wrap prose at a fixed width, so an item usually spans several indented lines.
 */
export function listItems(block: string): string[] {
  const items: string[] = [];
  let inFence = false;

  for (const line of block.split("\n")) {
    if (FENCE.test(line)) {
      inFence = !inFence;
      continue;
    }
    if (inFence) continue;
    if (ITEM_START.test(line)) {
      items.push(line.replace(ITEM_START, "").trim());
      continue;
    }
    const last = items.length - 1;
    if (last < 0 || line.trim() === "") continue;
    if (/^\s+\S/.test(line)) {
      items[last] = `${items[last]} ${line.trim()}`;
      continue;
    }
    break;
  }
  return items;
}

/** Each `> ` blockquote of a block as one item: the other way a skill writes its hard rules. */
export function blockquoteItems(block: string): string[] {
  const items: string[] = [];
  let current: string[] = [];
  const flush = () => {
    if (current.length > 0) items.push(current.join(" "));
    current = [];
  };
  for (const line of block.split("\n")) {
    const match = line.match(/^\s*>\s?(.*)$/);
    if (match) {
      const text = (match[1] ?? "").trim();
      if (text === "") flush();
      else current.push(text);
    } else {
      flush();
    }
  }
  flush();
  return items;
}

/** Prose paragraphs of a block: no lists, tables, fences, headings, quotes or badge lines. */
export function paragraphs(block: string): string[] {
  const result: string[] = [];
  let inFence = false;
  let current: string[] = [];
  const flush = () => {
    if (current.length > 0) result.push(current.join(" "));
    current = [];
  };

  for (const line of block.split("\n")) {
    if (FENCE.test(line)) {
      inFence = !inFence;
      flush();
      continue;
    }
    if (inFence) continue;
    const trimmed = line.trim();
    if (
      trimmed === "" ||
      trimmed.startsWith("#") ||
      trimmed.startsWith("|") ||
      trimmed.startsWith(">") ||
      trimmed.startsWith("[![") ||
      trimmed.startsWith("![") ||
      ITEM_START.test(trimmed)
    ) {
      flush();
      continue;
    }
    current.push(trimmed);
  }
  flush();
  return result;
}

/** A section's list items, or its paragraphs when it is written as prose. */
export function listOrParagraphs(block: string): string[] {
  const items = listItems(block);
  return items.length > 0 ? items : paragraphs(block);
}

/** Body rows of the first pipe table in a block, cells trimmed; `\|` inside a cell stays in the cell. */
export function tableRows(block: string): string[][] {
  const rows: string[][] = [];
  let seenHeader = false;
  for (const line of block.split("\n")) {
    const trimmed = line.trim();
    if (!trimmed.startsWith("|")) {
      if (seenHeader) break;
      continue;
    }
    if (/^\|[\s:|-]+\|?$/.test(trimmed)) continue;
    const inner = trimmed.slice(1, trimmed.endsWith("|") ? -1 : undefined);
    const cells = inner.split(/(?<!\\)\|/).map((cell) => cell.replace(/\\\|/g, "|").trim());
    if (!seenHeader) {
      seenHeader = true;
      continue;
    }
    rows.push(cells);
  }
  return rows;
}

export interface SkillRule {
  /** The bold lead of a rule, e.g. "Never sort by order alone." */
  rule: string;
  /** What follows it; inline markdown. */
  reason: string;
}

/** `**Rule.** The reason.` becomes { rule, reason }; an item without a bold lead is all rule. */
export function splitRule(item: string): SkillRule {
  const match = item.match(/^\*\*([\s\S]+?)\*\*\s*([\s\S]*)$/);
  if (!match) return { rule: item.trim(), reason: "" };
  return { rule: (match[1] ?? "").trim(), reason: (match[2] ?? "").trim() };
}

/** Links, bold and code ticks dropped, for text that is shown or compared as plain text. */
export function stripInline(text: string): string {
  return text
    .replace(/\[([^\]]+)\]\([^)]+\)/g, "$1")
    .replace(/\*\*([^*]+)\*\*/g, "$1")
    .replace(/`([^`]+)`/g, "$1")
    .trim();
}
```

Why each one is shaped the way it is:

- **Fences are skipped everywhere.** A `## ` or a `- ` inside a fenced example is code, not structure; a
  skill's README shows its own install commands in fences.
- **Wrapped items are joined.** Skills wrap prose at a fixed width, so a numbered rule usually spans
  several indented lines; `listItems` joins an indented line onto the item above and stops at the first
  unindented prose line, which is where the list ends.
- **Rules have two shapes.** A README lists its non-negotiables as `1. **Rule.** Reason.`; a `SKILL.md`
  may write its hard rules as blockquotes. `blockquoteItems` reads the second, so a skill with no README
  list still shows its rules.
- **A reason may continue the rule's sentence.** `**Tokens are compared in constant time**, never with
  ===` gives a reason that starts with a comma; the page joins it without a space ([routes.md](routes.md)).
- **`\|` stays in its cell.** A table cell that names a union type or a shell pipe escapes the pipe.

## Upstream files

```ts
// file: lib/skills/upstream.ts
import { listItems, paragraphs, splitRule, splitSections, stripInline } from "./markdown";
import type { SkillProvenance, SkillProvenanceGroup, SkillRelease } from "./types";

const RELEASE_HEADER = /^\[(\d+\.\d+\.\d+)\]\s*-\s*(\d{4}-\d{2}-\d{2})/;

interface ReleaseBlock {
  version: string;
  date: string;
  body: string;
}

/** Every `## [x.y.z] - YYYY-MM-DD` block of a Keep a Changelog file, in file order (newest first). */
function releaseBlocks(changelog: string): ReleaseBlock[] {
  const blocks: ReleaseBlock[] = [];
  for (const block of splitSections(changelog).slice(1)) {
    const header = block.title.match(RELEASE_HEADER);
    if (!header?.[1] || !header[2]) continue; // `## [Unreleased]` and prose headings
    blocks.push({ version: header[1], date: header[2], body: block.body });
  }
  return blocks;
}

/** The text before a release's first `### ` heading: the sentence that says what the release is. */
function releaseIntro(body: string): string {
  return body.split(/^### /m)[0] ?? "";
}

export function parseReleases(changelog: string): SkillRelease[] {
  return releaseBlocks(changelog).map(({ version, date, body }) => ({
    version,
    date,
    summary: stripInline(paragraphs(releaseIntro(body))[0] ?? listItems(body)[0] ?? ""),
  }));
}

/**
 * The ledger of references/provenance.md: each h2 a group ("Fixed in the
 * templates", "Kept deliberately", "Added"), each h3 an item with its first
 * paragraph as the body.
 */
export function provenanceFromFile(markdown: string, url: string): SkillProvenance {
  const sections = splitSections(markdown.replace(/^# .*\n/, ""));
  const groups: SkillProvenanceGroup[] = [];

  for (const section of sections.slice(1)) {
    const [lead = "", ...parts] = section.body.split(/^### /m);
    const items = parts.map((part) => {
      const newline = part.indexOf("\n");
      const heading = newline === -1 ? part : part.slice(0, newline);
      const body = newline === -1 ? "" : part.slice(newline + 1);
      return {
        title: stripInline(heading.replace(/^\d+\.\s*/, "")),
        body: paragraphs(body)[0] ?? "",
      };
    });
    if (items.length > 0) groups.push({ label: section.title, note: paragraphs(lead)[0] ?? "", items });
  }

  return { source: "provenance.md", url, intro: paragraphs(sections[0]?.body ?? ""), groups };
}

/**
 * A skill written from a vendor specification may keep its ledger in the
 * changelog instead. Then the releases with a `### Fixed` section are the
 * ledger: each fixed item is a rule that integrating the API changed.
 */
export function provenanceFromChangelog(changelog: string, url: string): SkillProvenance {
  const groups: SkillProvenanceGroup[] = [];
  for (const { version, date, body } of releaseBlocks(changelog)) {
    const fixed = body.split(/^### /m).find((part) => /^Fixed\b/.test(part));
    if (!fixed) continue;
    const items = listItems(fixed.slice(fixed.indexOf("\n") + 1)).map((item) => {
      const { rule, reason } = splitRule(item);
      // "**Title**: body" is how a changelog item reads, so the colon opens the reason.
      return { title: stripInline(rule.replace(/[:.]$/, "")), body: reason.replace(/^[:.]\s*/, "") };
    });
    if (items.length === 0) continue;
    groups.push({
      label: `Release ${version}, ${date}`,
      note: paragraphs(releaseIntro(body))[0] ?? "",
      items,
    });
  }
  return { source: "CHANGELOG.md", url, intro: [], groups };
}

export interface IndexEntry {
  name: string;
  builds: string;
  backend: string;
}

/** The `| [`name`](url) | version | builds | backend |` rows of the index README's skills table. */
export function parseIndexTable(markdown: string): Map<string, IndexEntry> {
  const entries = new Map<string, IndexEntry>();
  const row = /^\|\s*\[`([^`]+)`\]\([^)]+\)\s*\|\s*[^|]*\|\s*([^|]*)\|\s*([^|]*)\|\s*$/;
  for (const line of markdown.split("\n")) {
    const match = row.exec(line.trim());
    if (!match?.[1]) continue;
    entries.set(match[1], {
      name: match[1],
      builds: (match[2] ?? "").trim(),
      backend: (match[3] ?? "").trim(),
    });
  }
  return entries;
}

/**
 * The parts of a module from a one-line "Builds" cell: "Self-service kiosk:
 * walk-up flow, on-screen keyboard, pay-by-QR". The list starts after the
 * first colon, or after " with " where there is none, and splits on commas.
 */
export function featuresFromBuilds(builds: string): string[] {
  const colon = builds.indexOf(": ");
  const withAt = builds.indexOf(" with ");
  const start = colon !== -1 ? colon + 2 : withAt !== -1 ? withAt + 6 : -1;
  if (start === -1) return [];
  const parts = builds
    .slice(start)
    .replace(/\.$/, "")
    .split(/,\s+/)
    .map((part) => stripInline(part.replace(/^and\s+/, "")))
    .filter((part) => part.length > 0);

  // "category, tag and article pages" is one part: a lone word followed by an
  // "x and y" part is the start of that part, not a part of its own.
  const merged: string[] = [];
  for (const part of parts) {
    const last = merged.length - 1;
    const previous = merged[last];
    if (previous !== undefined && !previous.includes(" ") && / and /.test(part)) {
      merged[last] = `${previous}, ${part}`;
    } else {
      merged.push(part);
    }
  }
  return merged.map((part) => part.charAt(0).toUpperCase() + part.slice(1));
}
```

**The ledger has two homes.** A skill written from its own module keeps `references/provenance.md` with
h2 groups (Fixed in the templates, Kept deliberately, Added) of h3 items; a skill written from a vendor
specification may keep the record in its changelog instead. `provenanceFromChangelog` then reads each
release's `### Fixed` items, written `**Title**: body`, as the ledger. The loader takes the file when it
exists and the changelog otherwise, and `SkillProvenance.source` says which, so the page links the right
file.

**Releases come from the changelog, never from git.** The version, the release date and the first release
date drive the page header, JSON-LD `datePublished` and `dateModified`, and the sitemap. `## [Unreleased]`
and any heading that is not a release are skipped.

**The index table is optional.** An organization with an index repository lists each skill in a README
table, `| [`name`](url) | version | builds | backend |`. `featuresFromBuilds` splits the "Builds" cell
into the "What it builds" list and keeps "category, tag and article pages" as one part. Without an index,
`features` is empty and the page leaves the section out.

## Checklist

- [ ] Every file is read through `readFrontmatter`, never `gray-matter` directly
- [ ] `SKILL_LAYOUT` matches the headings the host's skills use
- [ ] A skill missing a section shows the page without it, and the validator reports it
