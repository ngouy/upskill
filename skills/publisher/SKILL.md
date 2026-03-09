---
name: publisher
description: "Use when a user wants to publish, share, or version a skill — creating a plugin from a raw skill file, sharing a skill on GitHub, releasing a new version, bumping semver, or managing/versioning a project-level skill in .claude/skills/. Also use immediately after skill-creator completes."
version: 1.0.0
---

Publisher takes a skill in any state and handles the full publication pipeline: scaffolding a plugin structure, git/GitHub setup, versioning, and releasing. It is the bridge between creating a skill and sharing it with the world.

## Prerequisites

Run `gh auth status`. If it fails, stop: "Publisher requires `gh` CLI authenticated — run `gh auth login`." Get the user's GitHub username: `gh api user --jq .login`.

## Detect State

Check in order. Stop at the first "no" — you're at the stage from the previous row. If check 2 fails: raw.

| # | Check | Stage achieved |
|---|-------|---------------|
| 1 | Path inside `<project>/.claude/skills/`? | project-level mode (see below) |
| 2 | `.claude-plugin/plugin.json` exists? | scaffolded |
| 3 | `git rev-parse --git-dir` succeeds with commits? | git-tracked |
| 4 | `gh repo view <owner>/<name>` succeeds? | github-hosted |
| 5 | `git tag --list 'v*'` returns semver tags? | versioned |
| 6 | `gh release list --limit 1` returns results? | released |

Show detected state. Route by the user's stated intent. If ambiguous, ask: "What would you like to do?"

## Actions

**Scaffold (raw → scaffolded):**
Ask: "Add to an existing plugin you own, or create a new one?" List only plugins where `plugin.json` author matches the user's GitHub username. If new, ask for the destination directory (suggest cwd). Read each template from the `templates/` directory adjacent to this skill file. Prompt user for `{{PLUGIN_NAME}}`, `{{PLUGIN_DESCRIPTION}}`, `{{KEYWORDS}}`; fill `{{AUTHOR_NAME}}` and `{{GITHUB_OWNER}}` from `gh api user`. Write filled templates to target. Move skill to `skills/<name>/SKILL.md`. Check `~/.claude/upskill-state.json` for `watermark.enabled`; if absent, ask once: "Add upskill attribution? [y/N]" and store the answer. If enabled, add `maintained-with: upskill (github.com/ngouy/upskill)` to SKILL.md frontmatter.

**Git init (scaffolded → git-tracked):**
`git init`, stage all files, commit: `feat: initial plugin scaffold`.

**GitHub setup (git-tracked → github-hosted):**
Ask visibility (public / private / org). Confirm before acting. Run `gh repo create <name> --[public|private] --source=. --push`. Suggest next step: version bump.

**Version bump:**
Diff since last tag. Suggest: new capabilities → minor, breaking changes → major, docs/typos → patch. Explain reasoning. User can override. Update `plugin.json` version. Commit: `chore: bump version to vX.Y.Z`.

**Release:**
Create git tag `vX.Y.Z`. Prompt user to review/update `RELEASE-NOTES.md`. Confirm, then push tag and run `gh release create`.

## Docs & Upgrades

**Upgrade (partial plugin → complete):** If `.claude-plugin/plugin.json` exists but files are missing, generate only what's absent — never overwrite. For a new skill added to an existing plugin: create `skills/<name>/SKILL.md`, update the README skills table, suggest a minor version bump.

**Docs:** After scaffold or on request, flag any README sections still containing template stubs. Offer to fill them using context: skill names from `skills/`, descriptions from SKILL.md frontmatter, repo URL from `plugin.json`. List unfilled sections explicitly — never leave stubs silently.

## Project-Level Skills

For skills in `<project>/.claude/skills/`: version management only. Stop if the repo has uncommitted changes — ask to commit or stash first. Read `version` from frontmatter (default `1.0.0` if absent). Suggest bump level with reasoning. Update frontmatter version. Create branch `upskill/bump-<skill-name>-v<new-version>`. Spawn a sub-agent to create the PR, instructing it to read the project's CLAUDE.md and match its commit and PR conventions.

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
