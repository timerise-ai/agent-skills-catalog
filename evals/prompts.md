---
prompts:
  - prompt: "Add an Agent Skills section to our site: one page listing the skills our GitHub organization publishes, with search and categories, and a page per skill showing its rules, its latest version, how to install it and how each coding agent did in its evals. The build must not call GitHub."
    stack: No data store
  - prompt: "Our skill pages still show the old version for a while after we release a skill, and one eval result we committed never shows up on the page. Find out why and fix it."
  - prompt: Audit our skills catalog against the skill's hard rules and list every place it breaks one.
---

# Prompts

What an operator types after installing this skill, in their own words. An agent eval installs the skill
into an empty Next.js app, gives the agent one of these prompts and no further help, then type-checks, builds
and tests the result; the first prompt runs before every release. The results are the other files in this
folder. Section 10 of [STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) says how a
run is made.

The skill repositories the first prompt names are not reachable in an unattended run, and neither GitHub nor
the skills CLI is an external service the build may need: the skill ships an example skill in
`assets/content/` for exactly that case, and the sync is handed over to run later.
