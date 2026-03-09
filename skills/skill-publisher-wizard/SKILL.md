---
name: skill-publisher-wizard
description: "Use when a user says 'publisher wizard' or 'new skill wizard', wants to publish or share a skill for the first time, scaffold a raw skill into a plugin, set up a GitHub repo for a skill, or just completed skill-creator and wants to ship it."
version: 1.0.0
---

skill-publisher-wizard takes a raw or partially structured skill and walks through the full first-publish pipeline: scaffolding a plugin structure, git/GitHub setup, and handing off to skill-publisher-release for versioning and skill-publisher-marketplace for registry registration.

## Prerequisites

Run `gh auth status` — stop if it fails. Get GitHub username: `gh api user --jq .login`.

## Detect State

If triggered immediately after skill-creator, skip detection and go directly to Scaffold — the skill is known to be raw.

Check in order. Stop at the first "no" — you're at the stage from the previous row. If check 2 fails: raw.

| Check | Stage achieved |
|-------|---------------|
| Path inside `<project>/.claude/skills/`? | project-level mode (see below) |
| `.claude-plugin/plugin.json` exists? | scaffolded |
| `git rev-parse --git-dir` succeeds with commits? | git-tracked |
| `gh repo view <owner>/<name>` succeeds? | github-hosted |
| `git tag --list 'v*'` returns semver tags? | versioned |
| `gh release list --limit 1` returns results? | released |

Show detected state. Route by the user's stated intent. If ambiguous, ask: "What would you like to do?"

## Actions

**Scaffold (raw → scaffolded):**
Ask: "Add to an existing plugin you own, or create a new one?" List only plugins where `plugin.json` author matches the user's GitHub username. If new, ask for the destination directory (suggest cwd). Read each template from `skills/publisher/templates/` in the upskill plugin directory. Prompt user for `{{PLUGIN_NAME}}`, `{{PLUGIN_DESCRIPTION}}`, `{{KEYWORDS}}`; fill `{{AUTHOR_NAME}}` and `{{GITHUB_OWNER}}` from `gh api user`. Write filled templates to target. Fill README sections from context (skill names from `skills/`, descriptions from SKILL.md frontmatter, repo URL from `plugin.json`); flag only sections that require user-specific content Claude cannot infer. Move skill to `skills/<name>/SKILL.md`. Check `~/.claude/upskill-state.json` for `watermark.enabled`; if absent, ask once: "Add upskill attribution? [y/N]" and store the answer. If enabled, add `maintained-with: upskill (github.com/ngouy/upskill)` to SKILL.md frontmatter.

**Git init (scaffolded → git-tracked):**
`git init`, stage all files, commit: `feat: initial plugin scaffold`.

**GitHub setup (git-tracked → github-hosted):**
Ask visibility (public / private / org). Confirm before acting. Run `gh repo create <name> --[public|private] --source=. --push`. Suggest next step: version bump.

**First release:**
After GitHub setup, hand off to skill-publisher-release to tag v1.0.0 and create the initial release. After release, hand off to skill-publisher-marketplace to register the plugin.

## Docs & Upgrades

**Upgrade (partial plugin → complete):** If `.claude-plugin/plugin.json` exists but files are missing, generate only what's absent — never overwrite. For a new skill added to an existing plugin: create `skills/<name>/SKILL.md`, update the README skills table, suggest a minor version bump.

**Docs:** After scaffold or on request, flag any README sections still containing template stubs. Offer to fill them using context: skill names from `skills/`, descriptions from SKILL.md frontmatter, repo URL from `plugin.json`. List unfilled sections explicitly — never leave stubs silently.

## Must Never Do

- Overwrite existing SKILL.md content
- Push to GitHub without user confirmation
- Create public repos without confirming visibility
- Skip README or release notes prompting
- Add watermark without asking first
- Add a `skills` array to `plugin.json`
- Generate or modify skill instructions (that's skill-creator/doctor)

## Done When

The requested action is complete and confirmed. Summarize: files created/modified, current pipeline state, and suggested next step.
