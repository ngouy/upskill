# upskill

[![Version](https://img.shields.io/github/v/tag/ngouy/upskill?label=version)](https://github.com/ngouy/upskill/releases)
[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Claude Code Plugin](https://img.shields.io/badge/claude--code-plugin-purple.svg)](https://github.com/anthropics/claude-code)
[![Install](https://img.shields.io/badge/install-%2Fplugin%20install-orange.svg)](#installation)

**The full lifecycle layer for Claude Code skills — everything that happens after a skill is created.**

---

[skill-creator](https://github.com/anthropics/claude-code-skill-creator) takes you from 0 to 1: a new skill on your machine. `upskill` takes you from 1 to infinity: organizing your skill inventory, versioning and publishing skills so others can install them, auditing third-party skills for safety and quality, and understanding precisely what your loaded skills are costing your session. If skill-creator is the moment of birth, upskill is everything that keeps a skill alive, healthy, and shareable.

---

## Table of Contents

1. [Installation](#installation)
2. [Quick Start](#quick-start)
3. [Sub-Skills Overview](#sub-skills-overview)
4. [Detailed Usage](#detailed-usage)
   - [manager — Unified Inventory](#manager--unified-inventory)
   - [publisher — Ship Your Skills](#publisher--ship-your-skills)
   - [doctor — Quality and Safety Analysis](#doctor--quality-and-safety-analysis)
   - [auditor — Session Snapshot](#auditor--session-snapshot)
5. [Relationship to skill-creator](#relationship-to-skill-creator)
6. [State File](#state-file)
7. [Contributing](#contributing)
8. [License](#license)

---

## Installation

```
/plugin install github:ngouy/upskill
```

That's it. All four sub-skills become available immediately in the current session and in every future session.

---

## Quick Start

```
# See everything installed
"Show me all my installed skills"

# Publish a skill you just created
"I just made a new skill, help me put it on GitHub"

# Check a skill for problems
"Run doctor on my-custom-workflow"

# Audit your current session
"Audit my loaded skills"
```

Claude activates the appropriate sub-skill automatically based on context, or you can invoke them explicitly by name.

---

## Sub-Skills Overview

| Sub-skill | One-line description |
|---|---|
| `manager` | Unified inventory of all installed skills — list, install, update, remove |
| `publisher` | Scaffold, version, and ship any skill to GitHub as an installable plugin |
| `doctor` | Analyze skill files for quality, structure, and safety — Curator and Guardian modes |
| `auditor` | Point-in-time session snapshot: token cost, hard checks, and health rating for all loaded skills |

---

## Detailed Usage

### manager — Unified Inventory

`manager` is your single pane of glass for everything installed. It unifies plugin-installed skills (managed by Claude Code) and local custom skills (raw markdown files in `~/.claude/skills/`) into one view. From the user's perspective, there is one inventory — not two.

#### List All Installed Skills

Ask: *"What skills do I have installed?"* or *"Show me my skill inventory."*

```
┌─────────────────────────────────────────────────────────────────────┐
│ INSTALLED SKILLS                                          2 plugins  │
├──────────────────────┬────────────┬────────────┬───────────────────┤
│ Skill                │ Source     │ Version    │ Last Updated       │
├──────────────────────┼────────────┼────────────┼───────────────────┤
│ manager              │ plugin     │ 1.2.0      │ 3 days ago         │
│ publisher            │ plugin     │ 1.2.0      │ 3 days ago         │
│ doctor               │ plugin     │ 1.2.0      │ 3 days ago         │
│ auditor              │ plugin     │ 1.2.0      │ 3 days ago         │
│ my-custom-workflow   │ local      │ (untracked)│ 12 days ago        │
│ commit               │ plugin     │ 2.0.1      │ 1 week ago         │
└──────────────────────┴────────────┴────────────┴───────────────────┘

[!] 1 update available: commit (2.0.1 → 2.1.0)
```

Local skills without git history show version as `(untracked)`. Skills in a git repo show the short SHA.

#### Install, Remove, Update

```
# Install a plugin
"Install github:author/plugin-name"

# Remove a plugin (always confirms before removing)
"Remove the commit plugin"

# Update all plugins, or a specific one
"Update all my skills"
"Update the commit plugin"
```

After an update, manager shows a summary of what changed if the plugin includes `RELEASE-NOTES.md`.

#### Periodic Update Nudges

Every 10 sessions (configurable in the state file), manager surfaces a brief message at session start:

```
[upskill] 2 skill updates available. Run manager to review.
```

This is the only proactive, automatic behavior across all of upskill. Everything else is on-demand.

---

### publisher — Ship Your Skills

`publisher` bridges the gap between a local skill file and an installable plugin that anyone can use with `/plugin install`. It handles every step: structuring the directory, generating `plugin.json`, initializing git, creating the GitHub repo, bumping versions, tagging releases, and writing documentation stubs.

#### Natural Integration with skill-creator

When skill-creator finishes generating a new skill, publisher is the natural next step. Tell Claude: *"I just created a new skill, help me publish it."* Publisher will skip straight to the scaffold wizard, knowing the skill is raw.

#### Scaffold a Raw Skill File

```
"Take my skill at ~/.claude/skills/my-workflow.md and scaffold it into a plugin"
```

Output structure:

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json        ← generated with correct metadata
├── skills/
│   └── my-workflow/
│       └── SKILL.md       ← moved from original location
├── README.md              ← generated stub
└── RELEASE-NOTES.md       ← stub for v1.0.0
```

Publisher prompts for missing fields (name, description, author) rather than guessing. It never overwrites your existing skill content.

#### GitHub Setup

```
"Set up a GitHub repo for this plugin"
```

Using the `gh` CLI, publisher creates the repository, configures the remote, and pushes the initial commit. Supports personal accounts, organizations, and private repositories. Never pushes without explicit confirmation.

#### Bump Version and Release

Publisher analyzes your git diff since the last tag and suggests an appropriate semver bump:

```
Changes since v1.1.0:
  - Added new capability in SKILL.md (3 files changed)

Suggested bump: minor (1.1.0 → 1.2.0)
Reason: new capabilities added, no breaking changes.

Proceed? [Y/n/major/patch]
```

After confirmation: updates `plugin.json`, commits the bump, creates the git tag, prompts you to review `RELEASE-NOTES.md`, and creates the GitHub release. You always have the final say on the version number.

#### Pipeline Stages

Publisher detects where you are and guides you to the next step:

```
raw .md file
  → scaffolded (has plugin.json + structure)
  → git-tracked (has git history)
  → github-hosted (has remote)
  → versioned (has semver tags)
  → released (has GitHub releases)
```

---

### doctor — Quality and Safety Analysis

`doctor` analyzes skill files for quality, structure, safety, and behavioral correctness. It operates in two modes: **Curator** for skills you own, and **Guardian** for third-party skills you're evaluating.

The name is intentional: a doctor diagnoses, treats, prescribes, and discharges. doctor does the same for skills — it finds what's wrong, recommends a fix, applies it with your approval, and clears the skill. *The doctor will see your skill now.*

Doctor never modifies skill files without explicit approval for each change. It recommends — it does not act unilaterally.

#### Curator Mode — Improve Your Own Skills

```
"Run doctor on my-custom-workflow"
"Check the quality of my commit skill"
```

Curator analyzes for:
- Clear purpose statement and scope
- Trigger conditions (when should this skill fire?) and exit signals (when is it done?)
- Token footprint and condensation opportunities
- Skills doing too much — should be split into sub-skills
- Missing guard rails around destructive actions
- Known bad patterns

| Pattern | Why It's Bad |
|---|---|
| Contradicts CLAUDE.md | Creates conflicting instructions; Claude behavior becomes unpredictable |
| Uses "always" for optional behaviors | Overfires, wastes tokens, irritates user |
| No confirmation before destructive actions | Unsafe by default |
| References file paths that don't exist | Will silently fail |
| Circular self-reference to another skill | Can cause loops |
| Trigger condition too broad | Fires constantly, consuming context unnecessarily |

Doctor also calls out good practices when it finds them — positive signal matters as much as negative.

#### Guardian Mode — Evaluate Third-Party Skills

```
"Is this third-party skill safe to use?"
"Run guardian mode on the newly installed plugin"
```

Guardian evaluates through three levels:

- **Level A — Red Flags (binary):** Direct evidence of malicious intent — data exfiltration, suppressing transparency, overriding other instructions, prompt injection patterns.
- **Level B — Suspicious Patterns (contextual):** Powerful capabilities without a clearly stated, proportionate reason. Not automatic red flags — Guardian evaluates whether the stated purpose justifies the requested capabilities.
- **Level C — Behavioral Anomalies (inferential):** Stated purpose does not match what the skill actually instructs Claude to do.

Guardian also detects conflicts between the third-party skill and your `CLAUDE.md` or other loaded skills.

#### Report Format

```
DOCTOR REPORT — skills/my-skill/SKILL.md
Mode: Curator
─────────────────────────────────────────────────────────────────────

CRITICAL (1)
  [C1] Missing purpose statement
       Every skill must open with a 1-2 sentence statement of what
       it does. This is the first thing Claude reads and sets context
       for all subsequent instructions.
       Fix: Add a "## Purpose" section at the top.

HIGH (1)
  [H1] Trigger condition too broad
       Current: "when I am working on any code"
       This fires on virtually every engineering session.
       Fix: Narrow to a specific context.

MEDIUM (2)
  [M1] No exit/completion signal
  [M2] Guard rail missing for bash execution

LOW (1)
  [L1] No usage examples

─────────────────────────────────────────────────────────────────────
SUMMARY: 1 critical, 1 high, 2 medium, 1 low
Overall health: NEEDS ATTENTION

Recommended next step: Fix C1 and H1, then re-run doctor.
```

Severity levels: `CRITICAL` / `HIGH` / `MEDIUM` / `LOW` for Curator mode. `RED FLAG` / `SUSPICIOUS` / `GREEN` for Guardian mode.

---

### auditor — Session Snapshot

`auditor` provides a point-in-time audit of your current Claude Code session through the lens of loaded skills. It answers: *"What is the skill layer doing to my session right now?"*

Auditor analyzes loaded skills and their session impact only. It does not read your conversation, your codebase, or your project files. That boundary is hard and intentional.

#### Run an Audit

```
"Audit my session"
"What are my loaded skills costing me in tokens?"
"Is anything in my skill setup conflicting or suspicious?"
```

#### Report Structure

**1. Loaded Skills Inventory** — all skills with estimated token cost and percentage of skill budget:

```
LOADED SKILLS
─────────────────────────────────────────────────────────────────────
Skill                    Source    Tokens (est.)    % of skill budget
─────────────────────────────────────────────────────────────────────
manager                  plugin    ~180             8%
publisher                plugin    ~240             11%
doctor                   plugin    ~520             23%
auditor                  plugin    ~310             14%
commit                   plugin    ~150             7%
my-custom-workflow       local     ~840             37%
─────────────────────────────────────────────────────────────────────
TOTAL                              ~2,240

[!] my-custom-workflow is consuming 37% of the skill budget.
    Consider reviewing it with doctor.
```

Token estimates are approximate (4 chars ≈ 1 token) and always labeled as such.

**2. Hard Checks** — binary pass/fail for security and conflict categories:

```
[PASS] No malicious patterns detected in any loaded skill
[FAIL] my-custom-workflow conflicts with CLAUDE.md
       Conflict: CLAUDE.md says "always use TypeScript".
       my-custom-workflow says "use JavaScript for scripts".
[PASS] No skills suppressing transparency
[PASS] No skills overriding system settings
```

**3. Soft Checks** — qualitative warnings and info notices:

```
[WARN] my-custom-workflow: trigger condition is very broad
[INFO] commit: no version metadata found in plugin.json
[OK]   All other loaded skills pass soft checks.
```

**4. Session Health Summary** — overall rating with prioritized action list.

Overall ratings: `HEALTHY` / `NEEDS ATTENTION` / `CRITICAL`.

---

## Relationship to skill-creator

`upskill` and [skill-creator](https://github.com/anthropics/claude-code-skill-creator) are designed as partners, not competitors.

```
skill-creator         upskill
─────────────         ───────
0 → 1                 1 → ∞
Creates skills        Manages, publishes, audits, and analyzes skills
Official Anthropic    Community plugin
No lifecycle          Full lifecycle
```

The recommended workflow is a clean handoff:

```
skill-creator  →  publisher  →  doctor  →  manager
 (create it)       (ship it)    (improve it)  (maintain it)
```

upskill never generates new skills from scratch. That boundary is intentional and permanent.

---

## State File

All persistent state lives at `~/.claude/upskill-state.json`. Created automatically on first use. Never committed to any repository.

### Schema

```json
{
  "version": "1",
  "sessionsUntilNextNudge": 10,
  "lastUpdateCheck": "2026-01-15T10:30:00Z",
  "installedPlugins": {
    "ngouy/upskill": {
      "installedVersion": "1.2.0",
      "lastChecked": "2026-01-15T10:30:00Z",
      "latestKnownVersion": "1.2.0"
    }
  },
  "nudgeConfig": {
    "intervalSessions": 10,
    "enabled": true
  }
}
```

| Field | Description |
|---|---|
| `sessionsUntilNextNudge` | Decremented each session; nudge fires at 0, resets to `intervalSessions` |
| `lastUpdateCheck` | ISO 8601 timestamp of last plugin update check |
| `installedPlugins` | Per-plugin version tracking, keyed by `author/plugin-name` |
| `nudgeConfig.intervalSessions` | Sessions between update nudges (default: 10) |
| `nudgeConfig.enabled` | Set to `false` to disable all proactive nudges |

Only `manager` writes to the state file. All other sub-skills are read-only on state.

---

## Contributing

Contributions are welcome — from humans and Claude sessions alike.

### For humans

Read `VISION.md` fully before opening a PR. It is the north star for every design decision and defines the scope, principles, and quality bar for all sub-skills.

Before opening a PR, verify:

- [ ] Clear purpose statement in the first 2 sentences of the skill
- [ ] Defined trigger conditions or an explicit note that the skill is manual-only
- [ ] Defined exit/completion signal
- [ ] Output format examples shown (not just described)
- [ ] "Must never do" list present
- [ ] Token footprint within the target budget for the sub-skill
- [ ] Passes `doctor --curator` with no CRITICAL or HIGH findings

If your change adds capability to a sub-skill, update `VISION.md` in the same PR. If your change crosses a sub-skill scope boundary, open an issue to discuss before implementing.

### For Claude sessions

Read `CLAUDE.md` and `VISION.md` before writing a single line. Then follow the session structure — do not skip or combine steps:

1. **Brainstorm session** — deep dive on intent, edge cases, design
2. **Plan session** — detailed implementation plan referencing VISION.md
3. **Implementation session** — write the SKILL.md
4. **Review session** — run `doctor --curator` on the result, fix all findings

The capability set in VISION.md is the specification — implement everything listed, nothing more. The "must never do" list is a hard constraint, not a style guideline. Scope boundaries are non-negotiable.

### Scope Boundaries Reference

| Sub-skill | Analyzes | Does NOT analyze |
|---|---|---|
| manager | Installed plugins, version metadata | Skill content, session behavior |
| publisher | Plugin structure, git/GitHub state | Skill quality, session impact |
| doctor | Skill file content, structure, safety | Session state, code, conversation |
| auditor | Loaded skills + their session impact | Conversation, code, project files |

Open issues at [github.com/ngouy/upskill/issues](https://github.com/ngouy/upskill/issues).

---

## License

MIT — see [LICENSE](LICENSE) for details.

---

*Managed with [upskill](https://github.com/ngouy/upskill)*
