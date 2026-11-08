# Testing and the content validator

Five files, 24 tests, run by `vitest`. Four cover the readers and the parser with inline fixtures and need
no content; `catalog.test.ts` runs the validator on the synced content itself and is the one that fails
when a sync brings in a file the page cannot show.

| File | Covers |
|---|---|
| `lib/skills/__tests__/markdown.test.ts` | Fenced headings, wrapped items, blockquote rules, paragraphs, tables, `splitRule` |
| `lib/skills/__tests__/evals.test.ts` | Unquoted dates, rejection reasons, the `runUrl` allowlist, run order, `latestRuns`, `impliedResult`, prompts |
| `lib/skills/__tests__/upstream.test.ts` | Releases, the ledger from either file, the index table, features |
| `lib/skills/__tests__/catalog.test.ts` | `validateCatalog()` on `content/skills/`: no errors, every skill has rules and a version |
| `components/skills/__tests__/SkillMarkdown.test.ts` | Relative links into the repository, absolute and unsafe links |

## Setup

```bash
npm install --save-dev vitest
```

If npm answers `ERESOLVE` on `@types/node`, as it does in a fresh `create-next-app` project, run
`npm install --save-dev @types/node@latest vitest`; never `--force` or `--legacy-peer-deps`. The config below
is `.mts` on purpose: under the `.ts` name Vite warns in any host without `"type": "module"`.

```json
{
  "scripts": {
    "test": "vitest run"
  }
}
```

```ts
// file: vitest.config.mts
import { fileURLToPath } from "node:url";

import { defineConfig } from "vitest/config";

export default defineConfig({
  // The same `@/` alias as tsconfig.json; point it at `./src/` in a host that keeps code there.
  resolve: { alias: { "@": fileURLToPath(new URL("./", import.meta.url)) } },
  test: { include: ["**/__tests__/**/*.test.ts"], exclude: ["node_modules/**", ".next/**"] },
});
```

Copy this config as shipped, with its alias changed only for a `src/` host. Do not add an exclude for the
installed skill's folder: it holds no test files, so the `include` pattern finds nothing there.

The alias matters for one file: `SkillMarkdown.tsx` imports `@/lib/skills/config`, and vitest does not read
`tsconfig.json` paths on its own. The library files import each other relatively, so they need none.

## The tests

```ts
// file: lib/skills/__tests__/markdown.test.ts
import { describe, expect, test } from "vitest";

import { blockquoteItems, listItems, paragraphs, splitRule, splitSections, tableRows } from "../markdown";

describe("markdown readers", () => {
  test("splitSections ignores headings inside code fences", () => {
    const sections = splitSections("intro\n## One\n```md\n## not a heading\n```\n## Two\nbody\n");
    expect(sections.map((s) => s.title)).toEqual(["", "One", "Two"]);
  });

  test("listItems joins wrapped lines and stops at the first prose line", () => {
    const block =
      "1. **Rule one.** Wrapped\n   onto two lines.\n2. **Rule two.** Short.\n\nAfter the list.\n";
    expect(listItems(block)).toEqual(["**Rule one.** Wrapped onto two lines.", "**Rule two.** Short."]);
  });

  test("listItems skips a list inside a code fence", () => {
    expect(listItems("```\n- not an item\n```\n- item\n")).toEqual(["item"]);
  });

  test("blockquoteItems reads each quote as one rule", () => {
    expect(blockquoteItems("> **Never A.** Because\n> of B.\n\n> **Never C.**\n")).toEqual([
      "**Never A.** Because of B.",
      "**Never C.**",
    ]);
  });

  test("paragraphs drop badges, tables, quotes and lists", () => {
    const block =
      "[![badge](x)](y)\n\nFirst line\nsecond line.\n\n| a | b |\n|---|---|\n\n> quote\n\n- item\n";
    expect(paragraphs(block)).toEqual(["First line second line."]);
  });

  test("tableRows skips header and separator and keeps escaped pipes", () => {
    const rows = tableRows("| Not this | Use instead |\n|---|---|\n| A \\| B | `c` |\n| D | E |\n\nafter\n");
    expect(rows).toEqual([
      ["A | B", "`c`"],
      ["D", "E"],
    ]);
  });

  test("splitRule separates the bold lead from the reason", () => {
    expect(splitRule("**Never X.** Because Y.")).toEqual({ rule: "Never X.", reason: "Because Y." });
    expect(splitRule("No bold lead")).toEqual({ rule: "No bold lead", reason: "" });
  });
});
```

```ts
// file: lib/skills/__tests__/evals.test.ts
import { describe, expect, test } from "vitest";

import {
  type EvalRun,
  impliedResult,
  latestRuns,
  parseEval,
  parsePromptsFile,
  runNumber,
  untriedAgents,
} from "../evals";
import { SKILLS_CATALOG } from "../config";

const RUN = `---
agent: claude-code
agentVersion: 2.1.283
model: claude-opus-5-5
date: 2026-09-26
skillVersion: 0.3.1
promptIndex: 1
prompt: "Hide this whole site behind a PIN."
stack: No data store
durationMinutes: 12
interventions: 0
checks:
  typecheck: pass
  build: pass
  tests: none
result: pass
filesChanged: 9
linesAdded: 412
isolated: true
timedOut: false
---

It left the robots header off the unlock page.
`;

const FILE = "2026-09-26-claude-code-p1.md";

function run(file: string, fields: Partial<EvalRun> = {}): EvalRun {
  const parsed = parseEval(file, RUN);
  if (!parsed.ok) throw new Error(parsed.reason);
  return { ...parsed.run, ...fields, file };
}

describe("parseEval", () => {
  test("reads an unquoted YAML date as the date written, not a dropped run", () => {
    const parsed = parseEval(FILE, RUN);
    expect(parsed.ok).toBe(true);
    if (parsed.ok) {
      expect(parsed.run.date).toBe("2026-09-26");
      expect(parsed.run.checks).toEqual({ typecheck: "pass", build: "pass", tests: "none" });
      expect(parsed.run.notes).toBe("It left the robots header off the unlock page.");
      expect(parsed.run.interventions).toBe(0);
    }
  });

  test("rejects with a reason instead of dropping", () => {
    const parsed = parseEval(FILE, RUN.replace("agent: claude-code", "agent: copilot"));
    expect(parsed).toEqual({ ok: false, file: FILE, reason: 'unknown agent "copilot"' });
    expect(parseEval("notes.md", RUN).ok).toBe(false);
    expect(parseEval(FILE, RUN.replace("result: pass", "result: great")).ok).toBe(false);
    expect(parseEval(FILE, "---\nagent: [unclosed\n---\n").ok).toBe(false);
  });

  test("links only an Actions run of the organization as the agent log", () => {
    const ours = `https://github.com/${SKILLS_CATALOG.org}/some-skill/actions/runs/123`;
    const withUrl = (url: string) => RUN.replace("isolated: true", `isolated: true\nrunUrl: ${url}`);
    const good = parseEval(FILE, withUrl(ours));
    const bad = parseEval(FILE, withUrl("https://evil.example/actions/runs/1"));
    expect(good.ok && good.run.runUrl).toBe(ours);
    expect(bad.ok && bad.run.runUrl).toBe("");
  });

  test("a missing isolated flag reads as not isolated", () => {
    const parsed = parseEval(FILE, RUN.replace("isolated: true\n", ""));
    expect(parsed.ok && parsed.run.isolated).toBe(false);
  });
});

describe("ordering", () => {
  test("runNumber reads the day's run, 1 without a suffix", () => {
    expect(runNumber("2026-09-26-codex-p1.md")).toBe(1);
    expect(runNumber("2026-09-26-codex-p1-2.md")).toBe(2);
    expect(runNumber("2026-09-26-codex-p1-12.md")).toBe(12);
  });

  test("latestRuns keeps the newest run per agent and prompt, -p1-2 over -p1", () => {
    const runs = [
      run("2026-09-26-claude-code-p1.md"),
      run("2026-09-26-claude-code-p1-2.md", { result: "fail" }),
      run("2026-09-20-claude-code-p1.md", { date: "2026-09-20" }),
      run("2026-09-26-codex-p1.md", { agent: "codex" }),
    ];
    expect(latestRuns(runs).map((r) => r.file)).toEqual([
      "2026-09-26-claude-code-p1-2.md",
      "2026-09-26-codex-p1.md",
    ]);
    expect(untriedAgents(runs)).toEqual(["gemini-cli"]);
  });

  test("impliedResult follows the run rule", () => {
    expect(impliedResult(run(FILE))).toBe("pass");
    expect(impliedResult(run(FILE, { checks: { typecheck: "pass", build: "pass", tests: "fail" } }))).toBe(
      "partial",
    );
    expect(impliedResult(run(FILE, { timedOut: true }))).toBe("fail");
  });
});

describe("parsePromptsFile", () => {
  test("reads objects and bare strings and drops empty prompts", () => {
    const source = `---\nprompts:\n  - prompt: "Build it."\n    stack: Supabase\n  - Audit it.\n  - prompt: ""\n---\n\n# Prompts\n`;
    expect(parsePromptsFile(source)).toEqual([
      { prompt: "Build it.", stack: "Supabase" },
      { prompt: "Audit it.", stack: "" },
    ]);
  });
});
```

```ts
// file: lib/skills/__tests__/upstream.test.ts
import { describe, expect, test } from "vitest";

import {
  featuresFromBuilds,
  parseIndexTable,
  parseReleases,
  provenanceFromChangelog,
  provenanceFromFile,
} from "../upstream";

const CHANGELOG = `# Changelog

## [Unreleased]

## [0.2.0] - 2026-10-07

Adds tag pages.

### Fixed

- **Encryption on TEST**: the IV is never prepended.

## [0.1.0] - 2026-09-20

### Added

- First release.
`;

describe("upstream files", () => {
  test("parseReleases lists released versions newest first and skips Unreleased", () => {
    expect(parseReleases(CHANGELOG)).toEqual([
      { version: "0.2.0", date: "2026-10-07", summary: "Adds tag pages." },
      { version: "0.1.0", date: "2026-09-20", summary: "First release." },
    ]);
  });

  test("provenanceFromChangelog turns Fixed items into the ledger", () => {
    const ledger = provenanceFromChangelog(CHANGELOG, "u");
    expect(ledger.groups).toEqual([
      {
        label: "Release 0.2.0, 2026-10-07",
        note: "Adds tag pages.",
        items: [{ title: "Encryption on TEST", body: "the IV is never prepended." }],
      },
    ]);
  });

  test("provenanceFromFile groups h3 items under their h2", () => {
    const md =
      "# Provenance\n\nIntro.\n\n## Fixed in the templates\n\n### 1. The sidebar hid links\n\nBody.\n\n**Shipped:** x\n\n## Kept deliberately\n\nNone yet.\n";
    const ledger = provenanceFromFile(md, "u");
    expect(ledger.intro).toEqual(["Intro."]);
    expect(ledger.groups).toEqual([
      {
        label: "Fixed in the templates",
        note: "",
        items: [{ title: "The sidebar hid links", body: "Body." }],
      },
    ]);
  });

  test("parseIndexTable and featuresFromBuilds read the index row", () => {
    const index = parseIndexTable(
      "| Skill | Version | Builds | Backend |\n|---|---|---|---|\n| [`help`](https://x/help) | 0.2.16 | Help center: category, tag and article pages, search | Repository files |\n",
    );
    const entry = index.get("help");
    expect(entry?.backend).toBe("Repository files");
    expect(featuresFromBuilds(entry?.builds ?? "")).toEqual(["Category, tag and article pages", "Search"]);
  });
});
```

```ts
// file: lib/skills/__tests__/catalog.test.ts
import { describe, expect, test } from "vitest";

import { getAllSkills } from "../catalog";
import { validateCatalog } from "../validate";

/**
 * The content validator, on the synced content itself. The loader skips what
 * it cannot show; this is where that becomes a failure an author sees.
 */
describe("catalog content", () => {
  test("has no errors", () => {
    const errors = validateCatalog().filter((problem) => problem.level === "error");
    expect(errors.map((e) => `${e.slug}: ${e.message}`)).toEqual([]);
  });

  test("every skill reads its rules and its releases", () => {
    for (const skill of getAllSkills()) {
      expect(skill.nonNegotiables.length, skill.slug).toBeGreaterThan(0);
      expect(skill.version, skill.slug).not.toBeNull();
    }
  });

  test("prints warnings without failing", () => {
    for (const problem of validateCatalog().filter((p) => p.level === "warning")) {
      console.warn(`${problem.slug}: ${problem.message}`);
    }
  });
});
```

```ts
// file: components/skills/__tests__/SkillMarkdown.test.ts
import { describe, expect, test } from "vitest";

import { repoUrlTransform } from "../SkillMarkdown";

describe("repoUrlTransform", () => {
  const toRepo = repoUrlTransform("https://github.com/your-org/help", "references");

  test("resolves a link relative to the file's folder in the repository", () => {
    expect(toRepo("ui.md")).toBe("https://github.com/your-org/help/blob/main/references/ui.md");
    expect(toRepo("../SKILL.md")).toBe("https://github.com/your-org/help/blob/main/SKILL.md");
  });

  test("leaves absolute, site and in-page links alone and drops unsafe schemes", () => {
    expect(toRepo("https://agentskills.io")).toBe("https://agentskills.io");
    expect(toRepo("/skills/ksef")).toBe("/skills/ksef");
    expect(toRepo("#rules")).toBe("#rules");
    expect(toRepo("javascript:alert(1)")).toBe("");
  });
});
```

## Content for the tests

`catalog.test.ts` reads whatever `content/skills/` holds. With no skills it passes trivially and the pages
build empty, which proves nothing. Run the sync first, or, with no network, copy this skill's
`assets/content/` into the app root: one example skill (`content/skills/example-skill/`) with a `meta.md`,
every upstream file, two prompts and one run, which passes the validator and renders every section.

To see the validator work, add a run with no prompt and watch the suite fail with its name:

```bash
printf -- '---\nagent: claude-code\nresult: pass\ndate: 2026-10-01\n---\n' \
  > content/skills/example-skill/upstream/evals/2026-10-01-claude-code-p1.md
npm test   # fails: example-skill: evals/2026-10-01-claude-code-p1.md: no prompt
rm content/skills/example-skill/upstream/evals/2026-10-01-claude-code-p1.md
```

## Checklist

- [ ] `npm test` runs all five files and passes
- [ ] `npm run typecheck` passes with `strict` (the templates also pass `noUncheckedIndexedAccess`)
- [ ] `npm run build` passes with the content committed
- [ ] The malformed-run check above fails the suite, then passes again once the file is removed
