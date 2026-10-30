# Evals: prompts and agent eval runs

Each skill repository carries an `evals/` folder: `prompts.md`, the prompts an operator types after
installing the skill, and one file per agent eval run, in which the skill was installed into an empty
Next.js app, given one prompt and no help, then type-checked, built and tested. The catalog reads both and
shows them; it never runs an eval and never edits a result.

## The files

```yaml
# evals/prompts.md frontmatter; the body is prose and is not shown
prompts:
  - prompt: "Build a help center from the markdown files in content/help."
    stack: No data store          # the backend the prompt asks for, when it asks for one
  - "Audit our help center against the skill's hard rules."   # a bare string is a prompt with no stack
```

```yaml
# evals/2026-09-26-claude-code-p1-2.md: <date>-<agent>-p<prompt>[-<run of the day>].md
agent: claude-code               # claude-code | codex | gemini-cli
agentVersion: 2.1.283
model: claude-opus-5-5
date: 2026-09-26
skillVersion: 0.3.1
promptIndex: 1
prompt: "Build a help center from the markdown files in content/help."
stack: No data store
durationMinutes: 12
interventions: 0
checks: { typecheck: pass, build: pass, tests: none }
result: pass                     # pass | partial | fail
filesChanged: 9
linesAdded: 412
isolated: true
timedOut: false
runUrl: https://github.com/<org>/<repo>/actions/runs/<id>   # only on a run made by a release workflow
```

The body below the frontmatter is the notes of the person who read the session: what the agent missed.
The frontmatter is what was measured and is never edited after the run.

| `result` | Means | Implied by the checks when |
|---|---|---|
| `pass` | Built, checks pass | Every check that ran passed |
| `partial` | Builds, tests fail | Typecheck and build pass, tests fail |
| `fail` | Did not build | Typecheck or build fails, or the run timed out |

## The parser

```ts
// file: lib/skills/evals.ts
import { SKILLS_CATALOG } from "./config";
import { asNumber, asString, readFrontmatter } from "./frontmatter";

/** The agents an eval drives, by their skills CLI `-a` id. */
export const EVAL_AGENTS = {
  "claude-code": "Claude Code",
  codex: "Codex CLI",
  "gemini-cli": "Gemini CLI",
} as const;

export type EvalAgent = keyof typeof EVAL_AGENTS;

export type CheckResult = "pass" | "fail" | "none";

/** pass: every check that ran passed. partial: it builds, the tests fail. fail: it does not build. */
export type EvalResult = "pass" | "partial" | "fail";

export interface SkillPrompt {
  prompt: string;
  /** The backend the prompt asks for; empty when it asks for none. */
  stack: string;
}

export interface EvalRun {
  /** The file name in evals/, e.g. `2026-09-26-claude-code-p1-2.md`. */
  file: string;
  agent: EvalAgent;
  agentVersion: string;
  model: string;
  date: string;
  skillVersion: string;
  /** 1-based index into the prompts at the time of the run. */
  promptIndex: number;
  prompt: string;
  stack: string;
  durationMinutes: number | null;
  checks: { typecheck: CheckResult; build: CheckResult; tests: CheckResult };
  result: EvalResult;
  filesChanged: number | null;
  linesAdded: number | null;
  /** Whether the agent ran without the operator's own skills, instructions and memory. */
  isolated: boolean;
  timedOut: boolean;
  /** Times a person answered a question to keep the agent going. */
  interventions: number | null;
  /** The GitHub Actions run holding the agent's log, on a run the release workflow made; else "". */
  runUrl: string;
  /** The notes of the person who read the session; markdown, may be empty. */
  notes: string;
}

export type EvalParse = { ok: true; run: EvalRun } | { ok: false; file: string; reason: string };

export const PROMPTS_FILE = "prompts.md";

const AGENT_IDS = Object.keys(EVAL_AGENTS) as EvalAgent[];
const RESULT_FILE = /^\d{4}-\d{2}-\d{2}-(claude-code|codex|gemini-cli)-p\d+(?:-\d+)?\.md$/;

function isAgent(value: unknown): value is EvalAgent {
  return typeof value === "string" && (AGENT_IDS as string[]).includes(value);
}

function isResult(value: unknown): value is EvalResult {
  return value === "pass" || value === "partial" || value === "fail";
}

function check(value: unknown): CheckResult {
  return value === "pass" || value === "fail" ? value : "none";
}

/**
 * Only an Actions run in one of the organization's own repositories is linked
 * as the agent log. The field is free text in a file anyone can open a pull
 * request against, so anything else is dropped rather than rendered as a link.
 */
function runUrl(value: unknown): string {
  const url = asString(value);
  const prefix = `https://github.com/${SKILLS_CATALOG.org}/`;
  return url.startsWith(prefix) && /\/actions\/runs\/\d+/.test(url) ? url : "";
}

/** The `prompts` list of prompts.md; a bare string is a prompt with no stack. */
export function parsePrompts(value: unknown): SkillPrompt[] {
  if (!Array.isArray(value)) return [];
  const prompts: SkillPrompt[] = [];
  for (const item of value) {
    const entry =
      typeof item === "string"
        ? { prompt: item.trim(), stack: "" }
        : item !== null && typeof item === "object"
          ? {
              prompt: asString((item as Record<string, unknown>).prompt),
              stack: asString((item as Record<string, unknown>).stack),
            }
          : null;
    if (entry && entry.prompt.length > 0) prompts.push(entry);
  }
  return prompts;
}

export function parsePromptsFile(source: string): SkillPrompt[] {
  return parsePrompts(readFrontmatter(source).data.prompts);
}

/**
 * One result file. A file the page cannot show honestly is rejected with the
 * reason, never dropped: the validator turns every rejection into a failure.
 */
export function parseEval(file: string, source: string): EvalParse {
  const reject = (reason: string): EvalParse => ({ ok: false, file, reason });
  if (!RESULT_FILE.test(file)) return reject("name is not <date>-<agent>-p<n>[-<k>].md");

  let parsed: ReturnType<typeof readFrontmatter>;
  try {
    parsed = readFrontmatter(source);
  } catch (error) {
    return reject(`frontmatter does not parse: ${error instanceof Error ? error.message : error}`);
  }
  const fm = parsed.data;
  if (!isAgent(fm.agent)) return reject(`unknown agent "${String(fm.agent)}"`);
  if (!isResult(fm.result)) return reject(`unknown result "${String(fm.result)}"`);
  const date = asString(fm.date);
  if (!/^\d{4}-\d{2}-\d{2}$/.test(date)) return reject(`date "${date}" is not YYYY-MM-DD`);
  const prompt = asString(fm.prompt);
  if (prompt === "") return reject("no prompt");
  const checks =
    fm.checks !== null && typeof fm.checks === "object" ? (fm.checks as Record<string, unknown>) : {};

  return {
    ok: true,
    run: {
      file,
      agent: fm.agent,
      agentVersion: asString(fm.agentVersion),
      model: asString(fm.model),
      date,
      skillVersion: asString(fm.skillVersion),
      promptIndex: asNumber(fm.promptIndex) ?? 1,
      prompt,
      stack: asString(fm.stack),
      durationMinutes: asNumber(fm.durationMinutes),
      checks: {
        typecheck: check(checks.typecheck),
        build: check(checks.build),
        tests: check(checks.tests),
      },
      result: fm.result,
      filesChanged: asNumber(fm.filesChanged),
      linesAdded: asNumber(fm.linesAdded),
      isolated: fm.isolated === true,
      timedOut: fm.timedOut === true,
      interventions: asNumber(fm.interventions),
      runUrl: runUrl(fm.runUrl),
      notes: parsed.content.trim(),
    },
  };
}

/**
 * The result the checks imply, by the rule the runs are made under. The page
 * shows the recorded `result`, never this; a disagreement is a validator
 * warning for the person who wrote the file.
 */
export function impliedResult(run: EvalRun): EvalResult {
  const { typecheck, build, tests } = run.checks;
  if (run.timedOut || typecheck === "fail" || build === "fail") return "fail";
  return tests === "fail" ? "partial" : "pass";
}

/**
 * The run number within a day: `...-p1.md` is the first, `...-p1-2.md` the
 * second. Compared as text, `-p1.md` sorts after `-p1-2.md` (`.` is after
 * `-`), which would put the older run first.
 */
export function runNumber(file: string): number {
  return Number(file.match(/-p\d+-(\d+)\.md$/)?.[1] ?? 1);
}

/** Newest first: by date, then by run number within the day, then by name for a stable order. */
export function sortRuns(runs: EvalRun[]): EvalRun[] {
  return [...runs].sort(
    (a, b) =>
      b.date.localeCompare(a.date) || runNumber(b.file) - runNumber(a.file) || b.file.localeCompare(a.file),
  );
}

/**
 * The newest run per agent and prompt: what the page shows. Keyed by the
 * prompt text, which is never reworded once it has results; a new prompt is
 * added instead, so the text identifies it across renumbering. Older runs stay
 * on disk as history.
 */
export function latestRuns(runs: EvalRun[]): EvalRun[] {
  const seen = new Set<string>();
  return sortRuns(runs).filter((run) => {
    const key = `${run.agent}\u0000${run.prompt}`;
    if (seen.has(key)) return false;
    seen.add(key);
    return true;
  });
}

/** The agents with no run at all, named so the page can say so rather than leave a gap. */
export function untriedAgents(runs: EvalRun[]): EvalAgent[] {
  return AGENT_IDS.filter((agent) => !runs.some((run) => run.agent === agent));
}
```

## What the rules are for

**A rejected file is a reason, never a disappearance.** `parseEval` returns `{ ok: false, file, reason }`
for a file it cannot show honestly: a name outside the pattern, YAML that does not parse, an unknown agent
or result, a date that is not `YYYY-MM-DD`, no prompt. The loader keeps only the runs that parsed, and
`validateCatalog()` lists every rejection as an error, which fails `npm test`. A test that iterates only the
runs that parsed proves nothing about the ones that did not.

**The page shows the recorded result.** `impliedResult` exists for the validator, which warns when the
recorded `result` disagrees with the checks; the page never replaces one with the other, since the person
who wrote the file may know why (a test script that never ran is `none`, not `fail`).

**The newest run per agent and prompt, failing ones included.** `latestRuns` sorts first, then keeps the
first run per agent and prompt text. Picking the best run, hiding a `fail`, or showing only prompt 1 turns a
measurement into marketing. Older runs stay on disk and in the repository as history.

**Prompts are keyed by their text.** A prompt with results is never reworded; a changed prompt is added as
a new one. The text therefore identifies a prompt even if the list is reordered, while `promptIndex` only
records its position at the time of the run. The validator warns when the two disagree.

**`-p1.md` sorts after `-p1-2.md`.** `.` comes after `-` in ASCII, so a text sort puts the first run of
the day after the second and `latestRuns` would show the older one. `runNumber` reads the suffix.

**The agent log link is allowlisted.** `runUrl` is free text in a file anyone can propose a change to, and
the page renders it as a link. Only an Actions run URL in one of the organization's own repositories
survives; anything else becomes `""` and the card shows no log link.

**An agent with no run is named.** `untriedAgents` lets the page say "Not run yet in Gemini CLI" instead of
showing two cards and leaving the reader to wonder about the third.

## Checklist

- [ ] Every file in `upstream/evals/` other than `prompts.md` parses, or `npm test` fails naming it
- [ ] The page renders `latestRuns(skill.evals)`, never a filtered or reordered copy
- [ ] A run with a foreign `runUrl` shows no log link
