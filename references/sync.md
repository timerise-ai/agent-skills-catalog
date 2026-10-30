# Sync: skill repositories into the repository

`npm run skills:sync` is the only code that talks to the skill repositories. It runs on a person's
machine or in a scheduled job, never in the build, and its result is committed like any other content.

## What it guarantees

| Property | How |
|---|---|
| Every file of a skill comes from one commit | The head sha is resolved first; every read names that sha |
| A release shows on the next sync, not minutes later | Reads by sha bypass the branch-name cache of `raw.githubusercontent.com` |
| A failed sync changes nothing | All files are fetched into memory, written to a staging folder, then renamed into place |
| A file deleted upstream disappears here | The new folder replaces the old one whole, so nothing survives from a previous commit |
| One failing skill does not stop the others | Each skill is its own try; the exit code is 1 if any failed |
| An unchanged repository leaves no diff | `sync.json` records the commit, not the time of the sync |
| `meta.md` is never written | The sync writes only `upstream/` and `_index/` |

A branch-name URL on `raw.githubusercontent.com` answers with `cache-control: max-age=300` (verified by
reading the response headers), so a sync run within five minutes of a push can read the previous commit's
file while the API already reports the new sha. A URL that names the sha can only ever return that commit.

## The `SkillSource` seam

```
SkillSource
  head(repo)             the branch head: { sha, date }; throws when the repository or branch is missing
  read(repo, sha, file)  the file's text at that commit, null when the file is not in it
  list(repo, sha, dir)   the names in a folder at that commit, null when the folder is not in it
```

A 404 is an answer (the file is not there) and becomes `null`; any other failure throws, so a GitHub
outage never reads as "this skill has no evals". Two implementations ship:

| `SKILLS_SOURCE` | Reads | Use for |
|---|---|---|
| `github` (default) | `api.github.com` for the head and folder listings, `raw.githubusercontent.com/<org>/<repo>/<sha>/<file>` for files | The published skills |
| `local` | `git log` and `git show <sha>:<file>` in `SKILLS_LOCAL_DIR/<repo>` | Checkouts on the same machine, a private mirror, or a CI job that already cloned the repositories |

The local source reads the commit, not the working tree, so uncommitted edits in a checkout never reach the
site. Another host (GitLab, Gitea, an archive) is a third object with the same three methods.

## Rate limits

The GitHub source makes two API calls per skill (the head commit and the `evals/` listing) and one for the
index; file reads go to `raw.githubusercontent.com`, which is not counted against the API. Anonymous API
access allows 60 requests an hour per IP and a token 5,000 (verified on `api.github.com/rate_limit`), so
past about thirty skills set `GITHUB_TOKEN`. The token only raises the limit; the repositories are public.

## The script

Plain Node 22 or later, no dependency, ES modules. It lives outside the app's TypeScript program on purpose:
it never runs in the build and must work before `npm install` has finished.

```js
// file: scripts/sync-skills.mjs
/**
 * Mirrors each skill repository's public files into
 * content/skills/<slug>/upstream/, so the catalog builds from the skill's own
 * files with no network call and no drift between builds.
 *
 *   npm run skills:sync            # every skill with a meta.md
 *   npm run skills:sync -- ksef    # one skill
 *
 * Every file of one skill is read at one commit, staged in a temporary folder
 * and swapped in only when all of it arrived, so the folder always holds one
 * whole commit: never two commits mixed, never a half-written evals/. Re-run
 * after a skill release and commit the result. meta.md is never touched.
 *
 * SKILLS_SOURCE=github (default) reads github.com/<org>; GITHUB_TOKEN lifts
 * the anonymous API limit. SKILLS_SOURCE=local reads checkouts under
 * SKILLS_LOCAL_DIR/<repo>, at their HEAD.
 */
import { execFile } from "node:child_process";
import { mkdir, readdir, readFile, rename, rm, writeFile } from "node:fs/promises";
import path from "node:path";
import { promisify } from "node:util";

// ORG and BRANCH equal SKILLS_CATALOG in lib/skills/config.ts.
const ORG = "your-org";
const BRANCH = "main";
const INDEX_REPO = "skills"; // the repository whose README lists every skill; null for none
const ROOT = path.join(process.cwd(), "content/skills");

/** Files every skill carries; a missing one fails the skill. */
const REQUIRED = ["SKILL.md", "README.md", "CHANGELOG.md"];
/** Files a skill may carry; a missing one is simply absent from the mirror. */
const OPTIONAL = ["references/provenance.md"];
/** Folders mirrored whole, markdown files only. */
const FOLDERS = ["evals"];

/**
 * @typedef {{ sha: string, date: string | null }} Commit
 * @typedef {{
 *   name: string,
 *   head(repo: string): Promise<Commit>,
 *   read(repo: string, sha: string, file: string): Promise<string | null>,
 *   list(repo: string, sha: string, dir: string): Promise<string[] | null>,
 * }} SkillSource
 */

/** @returns {SkillSource} */
function githubSource() {
  const headers = { "User-Agent": `${ORG} skills sync`, Accept: "application/vnd.github+json" };
  if (process.env.GITHUB_TOKEN) headers.Authorization = `Bearer ${process.env.GITHUB_TOKEN}`;

  /** A 404 is an answer (the file is not there); anything else that is not 2xx is a failure. */
  async function get(url, init) {
    const response = await fetch(url, init);
    if (response.status === 404) return null;
    if (!response.ok) throw new Error(`${response.status} ${response.statusText} for ${url}`);
    return response;
  }

  return {
    name: "github",
    async head(repo) {
      const response = await get(`https://api.github.com/repos/${ORG}/${repo}/commits/${BRANCH}`, {
        headers,
      });
      if (!response) throw new Error(`${ORG}/${repo}: repository or branch ${BRANCH} not found`);
      const data = await response.json();
      return { sha: data.sha, date: data.commit?.committer?.date ?? null };
    },
    async read(repo, sha, file) {
      // Pinned to the sha: a raw URL on the branch name is cached by the CDN for
      // minutes after a push, while a sha URL can only ever return that commit.
      const response = await get(`https://raw.githubusercontent.com/${ORG}/${repo}/${sha}/${file}`, {
        headers: { "User-Agent": headers["User-Agent"] },
      });
      return response ? response.text() : null;
    },
    async list(repo, sha, dir) {
      const response = await get(`https://api.github.com/repos/${ORG}/${repo}/contents/${dir}?ref=${sha}`, {
        headers,
      });
      if (!response) return null;
      const entries = await response.json();
      return entries.filter((entry) => entry.type === "file").map((entry) => entry.name);
    },
  };
}

/** @returns {SkillSource} */
function localSource() {
  const base = process.env.SKILLS_LOCAL_DIR;
  if (!base) throw new Error("SKILLS_SOURCE=local needs SKILLS_LOCAL_DIR");
  const git = promisify(execFile);
  const missing = (error) => error?.code === "ENOENT";
  return {
    name: "local",
    async head(repo) {
      const cwd = path.join(base, repo);
      const { stdout } = await git("git", ["log", "-1", "--format=%H %cI"], { cwd });
      const [sha, date] = stdout.trim().split(" ");
      return { sha, date: date ?? null };
    },
    async read(repo, sha, file) {
      // `git show <sha>:<file>` reads the commit, not the working tree, so
      // uncommitted edits in the checkout never reach the site.
      try {
        const { stdout } = await git("git", ["show", `${sha}:${file}`], {
          cwd: path.join(base, repo),
          maxBuffer: 16 * 1024 * 1024,
        });
        return stdout;
      } catch (error) {
        if (missing(error) || /does not exist|exists on disk, but not in/.test(String(error?.stderr)))
          return null;
        throw error;
      }
    },
    async list(repo, sha, dir) {
      try {
        const { stdout } = await git("git", ["ls-tree", "--name-only", `${sha}:${dir}`], {
          cwd: path.join(base, repo),
        });
        return stdout.split("\n").filter(Boolean);
      } catch (error) {
        if (/Not a valid object name|not a tree object/.test(String(error?.stderr))) return null;
        throw error;
      }
    },
  };
}

/** The `repo:` of meta.md, else the folder name. */
async function repoFor(slug) {
  const meta = await readFile(path.join(ROOT, slug, "meta.md"), "utf8");
  return meta.match(/^repo:\s*["']?([\w.-]+)["']?\s*$/m)?.[1] ?? slug;
}

/** Writes `files` as a new folder at `target`, replacing the old one only once all of it is on disk. */
async function swapIn(target, files) {
  const staging = `${target}.sync-${process.pid}`;
  const previous = `${target}.old-${process.pid}`;
  await rm(staging, { recursive: true, force: true });
  try {
    for (const [file, text] of files) {
      await mkdir(path.dirname(path.join(staging, file)), { recursive: true });
      await writeFile(path.join(staging, file), text);
    }
  } catch (error) {
    await rm(staging, { recursive: true, force: true });
    throw error;
  }
  await rm(previous, { recursive: true, force: true });
  await rename(target, previous).catch((error) => {
    if (error.code !== "ENOENT") throw error;
  });
  await rename(staging, target);
  await rm(previous, { recursive: true, force: true });
}

/** @param {SkillSource} source */
async function syncSkill(source, slug) {
  const repo = await repoFor(slug);
  const commit = await source.head(repo);
  /** @type {Map<string, string>} */
  const files = new Map();

  for (const file of [...REQUIRED, ...OPTIONAL]) {
    const text = await source.read(repo, commit.sha, file);
    if (text === null && REQUIRED.includes(file))
      throw new Error(`${repo}@${commit.sha.slice(0, 7)}: ${file} is missing`);
    if (text !== null) files.set(file, text);
  }
  for (const dir of FOLDERS) {
    const names = (await source.list(repo, commit.sha, dir)) ?? [];
    for (const name of names.filter((n) => n.endsWith(".md")).sort()) {
      const text = await source.read(repo, commit.sha, `${dir}/${name}`);
      if (text === null) throw new Error(`${repo}: ${dir}/${name} was listed but cannot be read`);
      files.set(`${dir}/${name}`, text);
    }
  }

  // No timestamp: syncing an unchanged repository must leave no diff.
  const record = { repo, branch: BRANCH, commit, files: [...files.keys()] };
  files.set("sync.json", `${JSON.stringify(record, null, 2)}\n`);
  await swapIn(path.join(ROOT, slug, "upstream"), files);
  console.log(`${slug}: ${files.size - 1} files @ ${commit.sha.slice(0, 7)}`);
}

/** The index README, whose table carries each skill's "Builds" and "Backend" cells. */
async function syncIndex(source) {
  const commit = await source.head(INDEX_REPO);
  const text = await source.read(INDEX_REPO, commit.sha, "README.md");
  if (text === null) throw new Error(`${INDEX_REPO}: README.md is missing`);
  await swapIn(path.join(ROOT, "_index"), new Map([["README.md", text]]));
  console.log(`_index: README.md @ ${commit.sha.slice(0, 7)}`);
}

const source = process.env.SKILLS_SOURCE === "local" ? localSource() : githubSource();
const requested = process.argv.slice(2);
const slugs =
  requested.length > 0
    ? requested
    : (await readdir(ROOT, { withFileTypes: true }))
        .filter((entry) => entry.isDirectory() && !entry.name.startsWith("_") && !entry.name.includes("."))
        .map((entry) => entry.name);

let failed = false;
if (INDEX_REPO) {
  await syncIndex(source).catch((error) => {
    failed = true;
    console.error(`_index: ${error instanceof Error ? error.message : error}`);
  });
}
for (const slug of slugs) {
  try {
    await syncSkill(source, slug);
  } catch (error) {
    // The previous copy stays as it was; one failing skill does not stop the rest.
    failed = true;
    console.error(`${slug}: ${error instanceof Error ? error.message : error}`);
  }
}
process.exit(failed ? 1 : 0);
```

```json
{
  "scripts": {
    "skills:sync": "node scripts/sync-skills.mjs"
  }
}
```

## Running it

```bash
npm run skills:sync                        # every skill folder, plus the index
npm run skills:sync -- help-center-markdown
GITHUB_TOKEN=... npm run skills:sync       # many skills, or a shared CI address
SKILLS_SOURCE=local SKILLS_LOCAL_DIR=~/src npm run skills:sync
```

To add a skill: create `content/skills/<slug>/meta.md`, run the sync for that slug, commit both. To remove
one: delete its folder. After a skill release: run the sync and commit; the page, the sitemap date and the
JSON-LD `dateModified` all move to the new release.

A scheduled job that syncs and opens a pull request is an extension, not part of the module; see
[extensions.md](extensions.md).

## Offline

With no network (an unattended agent, a locked-down CI), the sync cannot run. Copy `assets/content/` from
this skill into the app: one example skill with every file the loader reads, so the pages, the validator
and the tests have content. Delete it after the first real sync; the validator does not know it is an
example.

## Checklist

- [ ] `ORG` and `BRANCH` in the script equal `SKILLS_CATALOG.org` and `branch`
- [ ] `INDEX_REPO` names the index repository, or is `null`
- [ ] `npm run skills:sync` exits 0 and a second run leaves `git status` clean
- [ ] `upstream/` is committed; nothing in the build reads the network
