# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
