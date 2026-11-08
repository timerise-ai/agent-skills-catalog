# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.2] - 2026-11-08

From scoring the prompt-1 agent eval runs of 0.1.1.

### Changed

- The handover step in `SKILL.md` now requires each item in the final message in full, with its file or
  number, even when a README repeats it.

## [0.1.1] - 2026-11-08

Fix release, from scoring the prompt-1 agent eval runs of 0.1.0 and checking the templates on a fresh
`create-next-app` project.

### Fixed

- `splitRule` no longer uses the `s` regex flag, which failed the type-check at the `ES2017` target of a
  fresh project. Apps built from 0.1.0 with that target need the new pattern from `references/parsing.md`.
- The test config is `vitest.config.mts`, because the `.ts` name draws a Vite warning in a host without
  `"type": "module"`.

### Added

- A Cache Components variant in `references/adaptation.md`: a host with `cacheComponents` deletes
  `dynamicParams`, and an unknown slug then answers `200` with the not-found page and `noindex`.
- The `ERESOLVE` fix for installing `vitest` in `references/testing.md`.

### Changed

- The handover step in `SKILL.md` now names the placeholders left in place and every dependency added.
- A page intro or notice goes in a `layout.tsx` beside the pages, which stay as shipped (`SKILL.md`,
  `references/adaptation.md`).

## [0.1.0] - 2026-11-08

First release: an Agent Skills catalog for a Next.js App Router site, generated from each skill
repository's own files.

### Added

- A sync script that reads every file of a skill at one commit, stages it and swaps it in whole, with a
  GitHub source and a local-checkout source.
- Readers for frontmatter (with YAML date normalization), the section layout (`SKILL_LAYOUT`), lists,
  blockquote rules, tables, releases, the provenance ledger from its own file or the changelog, and the
  index table.
- An eval parser that rejects a malformed run with its reason, run ordering, and the newest run per agent
  and prompt.
- The catalog loader and a content validator whose errors fail `npm test`.
- Catalog, category and skill pages with JSON-LD and sitemap entries, and the components behind them.
- Five test files, 24 tests, and an example skill in `assets/content/` for a build with no network.
- `evals/prompts.md` and the agent eval workflow caller.
