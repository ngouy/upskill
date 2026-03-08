# Claude Code Instructions — upskill

> Instructions for any Claude session working on this repository.
> Read this fully before touching any file.

## What This Project Is

`upskill` is a Claude Code plugin — the full lifecycle layer for skills. It picks up where `skill-creator` ends and owns **1→∞**: manage, publish, doctor, and audit skills.

The north star document is `VISION.md`. Read it before implementing anything.

## Session Structure

Every sub-skill is implemented across dedicated sessions in this order — do not skip or combine:

1. **Brainstorm session** — deep dive on intent, edge cases, design
2. **Plan session** — detailed implementation plan referencing VISION.md
3. **Implementation session** — write the SKILL.md
4. **Review session** — run `doctor --curator` on the result, fix findings

## Project Structure

```
upskill/
├── .claude-plugin/
│   └── plugin.json          # Plugin metadata — minimal, no skills array
├── skills/
│   ├── manager/SKILL.md
│   ├── publisher/SKILL.md
│   ├── doctor/SKILL.md
│   └── auditor/SKILL.md
├── CLAUDE.md                # This file
├── VISION.md                # North star — read before doing anything
├── README.md
└── RELEASE-NOTES.md
```

## Plugin System Rules

- `plugin.json` is minimal: `name`, `version`, `description`, `author` (object), `homepage`, `repository`, `license`, `keywords` — nothing else
- Skills are **auto-discovered** from `skills/` — no `skills` array in `plugin.json`
- Each skill lives at `skills/<name>/SKILL.md`
- Skills are **model-invoked** — the `description` frontmatter field determines when Claude activates them
- There are no slash commands for these skills — activation is natural language or explicit user request

## SKILL.md Format

Every SKILL.md must have frontmatter:

```markdown
---
name: skill-name
description: "Precise trigger description — when should Claude use this skill? Include specific phrases, keywords, and situations."
version: 1.0.0
---
```

The `description` field is the most important field. It must precisely cover the "When to Use" section from VISION.md for that skill.

## Scope Rules — Non-Negotiable

| Sub-skill | Owns | Never touches |
|---|---|---|
| manager | Installed plugins, version metadata | Skill content, session behavior |
| publisher | Plugin structure, git/GitHub state | Skill quality, session impact |
| doctor | Skill file content, structure, safety | Session state, code, conversation |
| auditor | Loaded skills + session impact | Conversation, code, project files |

When in doubt about scope, check this table. If a feature crosses a boundary, raise it — don't implement it silently.

## Quality Gates

Before any skill is considered done:
- [ ] Passes `doctor --curator` with zero CRITICAL or HIGH findings
- [ ] Has a clear purpose statement in the first 2 sentences
- [ ] Has a "When to Use" section matching VISION.md spec
- [ ] Has a "Must Never Do" list
- [ ] Token footprint is within budget (see VISION.md Design Principles)
- [ ] Output format examples are concrete, not abstract

## Code Style

- Instructions in SKILL.md must be unambiguous — if a sentence can be read two ways, rewrite it
- No passive voice in instructions: `do X`, not `X should be done`
- All output format examples must be actual examples, not descriptions of examples
- Never add features not listed in VISION.md without updating VISION.md first

## Committing

```bash
# Standard commit format
git commit -m "type(scope): [description]"

# Examples
git commit -m "feat(doctor): add guardian mode skeleton"
git commit -m "fix(manager): correct plugin.json scaffold format"
git commit -m "docs(vision): add deep scan future consideration note"
```

## What NOT To Do

- Never generate skills from scratch (that's skill-creator's job)
- Never cross sub-skill scope boundaries
- Never push without running the quality gate checklist
- Never modify VISION.md unilaterally — flag proposed changes and explain why
- Never implement features not in VISION.md — add them to Future Roadmap instead
