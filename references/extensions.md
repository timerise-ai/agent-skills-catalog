# Extensions

Designs beyond the shipped module. None of them ran as part of it; each is marked with what it needs and
what it must keep. The non-negotiables in [adaptation.md](adaptation.md) still hold in every one.

## A scheduled sync

A daily job runs `npm run skills:sync` with `GITHUB_TOKEN` and opens a pull request when `git status` is not
clean, so a release reaches the site after a review and a build, never straight from the network. Keep the
build itself offline: the job writes files; it does not deploy them.

## A Markdown copy of each skill page

For assistants that read pages as text, serve `<page>.md` (or the page with `Accept: text/markdown`) built
from the same `Skill`: title, tagline, the problem, the non-negotiables, install, prompts, the newest runs
as a table, and links to the repository. Name it from the page with
`<link rel="alternate" type="text/markdown">`, and list the skill pages in `llms.txt`. Build it from `Skill`,
never from the rendered HTML, so the two cannot drift.

## A price or a call to action per skill

A studio that also builds the module for clients adds `priceFrom`, `currency` and `days` to `meta.md`, and
a "build it with us" panel beside "build it yourself". It is the host's, not the skill's: keep it out of
`upstream/`, and leave it off pages whose `kind` is `tool`, since there is nothing to build for the reader.

## The site's assistant

A site chatbot that answers questions about the skills reads the same `getAllSkills()` (name, tagline,
backend, version, the non-negotiables) rather than a hand-written list, so it learns about a skill the day
its folder is synced. Give it summaries and one tool to read a whole skill, not every README in the prompt.

## Collecting evals across skills

An index repository can copy every skill's `evals/` into its own `evals/<skill>/` and write a table of the
newest run per skill and agent. That is the same `parseEval`, `sortRuns` and `latestRuns` over many skills;
run them in the index's own job, and keep each skill's repository the authoritative copy.

## Eval badges

A card can render the same badge a skill's README carries: label `eval` and the model, message the result,
coloured by result. Draw it from the newest prompt 1 run per agent, never from the best one.

## Category landing copy

A category page with only a description and cards is thin for search. A longer introduction per category
belongs in `SKILL_CATEGORY_META` (or a markdown file per category), written by the host; never assembled
from the skills' own text.
