# upskill — Vision & Master Intent Document

> **This document is the north star for the entire `upskill` project.**
> It exists to orient both Claude Code sessions implementing sub-skills and human contributors building the plugin.
> Every design decision, capability boundary, and quality expectation is captured here.
> When in doubt, return to this document.

---

## Table of Contents

1. [What Is upskill?](#what-is-upskill)
2. [Positioning & Scope](#positioning--scope)
3. [The Problems It Solves](#the-problems-it-solves)
4. [Architecture Overview](#architecture-overview)
5. [Sub-Skills: Complete Specification](#sub-skills-complete-specification)
   - [upskill/manager](#upskillmanager)
   - [upskill/publisher](#upskillpublisher)
   - [upskill/doctor](#upskilldoctor)
   - [upskill/auditor](#upskillauditor)
6. [State Management](#state-management)
7. [Future Roadmap](#future-roadmap)
8. [Design Principles](#design-principles)
9. [Quality Bar & Reference Standards](#quality-bar--reference-standards)
10. [Plugin File Structure](#plugin-file-structure)
11. [Contributing Guidelines](#contributing-guidelines)

---

## What Is upskill?

`upskill` is a Claude Code plugin that provides the **full lifecycle layer** for skills — everything that happens after a skill is created.

Claude Code already has [skill-creator](https://github.com/anthropics/claude-code-skill-creator) (the official Anthropic plugin) for the 0→1 moment: generating a new skill from scratch. `upskill` picks up exactly where skill-creator leaves off and owns **1→∞**:

- Organizing and inventorying your installed skills
- Versioning, publishing, and sharing skills with the world
- Auditing skills for quality, safety, and behavioral correctness
- Understanding how loaded skills are affecting your current session

`upskill` does **not** create skills from scratch. That boundary is intentional and permanent.

---

## Positioning & Scope

### The Gap in Today's Ecosystem

The Claude Code skill ecosystem today has a clean 0→1 story (skill-creator) and no 1→∞ story. After a skill is created:

- It lives as a local markdown file with no version history
- It has no path to becoming an installable plugin
- You have no unified view of all your installed skills
- You can't tell if a third-party skill is safe, well-written, or conflicting with your setup
- You have no visibility into how skills are affecting your session's token budget

`upskill` fills this gap entirely.

### What upskill Is NOT

- It is not a skill creator. It never generates new skills from scratch.
- It is not a code review tool. It does not analyze your codebase, your commits, or your conversation content.
- It is not a Claude Code fork or modification. It extends Claude Code via the standard plugin API.
- It is not opinionated about what your skills *do* — only about their structure, safety, and impact.

### Relationship to skill-creator

```
skill-creator         upskill
─────────────         ───────
0 → 1                 1 → ∞
Creates skills        Manages, publishes, audits, and analyzes skills
Official Anthropic    Community plugin
No lifecycle          Full lifecycle
```

`upskill` and skill-creator are natural partners. A recommended workflow:

```
skill-creator  →  publisher  →  doctor  →  manager
 (create it)       (ship it)    (improve it)  (maintain it)
```

---

## The Problems It Solves

### For Skill Authors

**Problem 1: No versioning.**
Skills are local markdown files. When Claude edits a skill during a session, there's a diff in the conversation but no rollback path unless the skill happens to be in a GitHub repo. Most skills aren't. History is lost silently.

**Problem 2: No sharing infrastructure.**
Getting a skill from "local markdown file" to "installable plugin other people can use via `/plugin install`" requires manually creating `plugin.json`, setting up a GitHub repo, structuring the directory correctly, writing a README, and tagging releases. This is 30 minutes of boilerplate that nobody does. Skills die on local machines.

**Problem 3: Skills created on the fly are never curated.**
skill-creator is fast and frictionless, which is great. The side effect is that skills accumulate: bloated, overlapping, unfocused, or quietly dangerous. There is no curation layer.

**Problem 4: No publisher pipeline.**
Even authors who want to share their skills have no tooling for semver bumping, tagging, generating release notes, or managing the GitHub side of a plugin release.

### For Skill Consumers

**Problem 1: No unified inventory.**
Installed skills come from two sources — plugin-installed (managed by Claude Code) and local custom (managed by the user). There is no single place to see everything that's loaded, where it came from, and when it was last updated.

**Problem 2: Manual update checking.**
There is no proactive notification when an installed skill or plugin has updates. You only know if you check manually.

**Problem 3: No quality signal.**
You cannot tell from the plugin registry whether a skill is well-written, harmful, or malicious before installing it — or after.

**Problem 4: Silent session damage.**
A poorly written skill can consume hundreds of tokens of context on every session. A malicious skill can subtly alter Claude's behavior. Neither is visible without dedicated tooling.

---

## Architecture Overview

### Plugin Structure

```
upskill/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   ├── manager/
│   │   └── SKILL.md
│   ├── publisher/
│   │   └── SKILL.md
│   ├── doctor/
│   │   └── SKILL.md
│   └── auditor/
│       └── SKILL.md
├── README.md
├── RELEASE-NOTES.md
└── VISION.md
```

### Installation

```bash
/plugin install github:ngouy/upskill
```

### Available Skills After Installation

| Skill | Purpose |
|---|---|
| `manager` | List, install, update, and remove skills — unified inventory |
| `publisher` | Scaffold, version, and ship skills to GitHub |
| `doctor` | Analyze skills for quality, safety, and bad patterns |
| `auditor` | Point-in-time session snapshot through the lens of loaded skills |

Skills are model-invoked — Claude activates them automatically when the context matches, or when you explicitly ask for them (e.g. "run doctor on my-skill").

### State File

All persistent state is stored in `~/.claude/upskill-state.json`. This file is created on first use and updated automatically. It is never committed to any repository. See [State Management](#state-management) for the full schema.

---

## Sub-Skills: Complete Specification

Each sub-skill section below is written to serve as the primary reference for the Claude session implementing it. It defines: the complete capability set, invocation patterns, expected behaviors, edge cases, output formats, and what the sub-skill must never do.

---

### upskill/manager

**The unified inventory layer for all installed skills.**

#### Purpose

`manager` provides a single, unified view of every skill loaded into the current Claude Code environment — regardless of whether it came from a plugin or was created locally. It also handles install/remove/update operations and delivers periodic update nudges.

The bifurcation between plugin-installed skills and local custom skills is an implementation detail. From the user's perspective, there is one inventory.

#### Capability Set

**1. List / Inventory**

Displays all installed skills with rich metadata:

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

Metadata shown per skill:
- Skill name and namespace
- Source: `plugin` (managed by Claude Code) or `local` (raw markdown in `~/.claude/skills/`)
- Version: semver for plugins, `(untracked)` for local files with no git history, or git short SHA if in a git repo
- Last updated: human-readable relative time
- Update available badge (when detected)

**2. Install**

Wraps `/plugin install` with a cleaner interface and post-install feedback:

```
manager install github:author/plugin-name
```

After install: confirms what was added, lists newly available skills, and offers to run `doctor` on the newly installed plugin.

**3. Remove**

Wraps `/plugin remove` with a confirmation step:

```
manager remove plugin-name
```

Always prompts for confirmation before removing. Never silently removes anything.

**4. Update**

Wraps `/plugin update` with change summary:

```
manager update           # update all
manager update plugin-name  # update one
```

After update: shows what changed (if RELEASE-NOTES.md is available in the plugin).

**5. Periodic Update Nudge**

Every ~10 sessions (configurable, tracked in `upskill-state.json`), `manager` proactively checks whether any installed plugins have newer versions available and surfaces a brief summary at the start of the session:

```
[upskill] 2 skill updates available. Run manager to review.
```

This is the **only** proactive/automatic behavior in all of `upskill`. Everything else is manual and on-demand.

Implementation note: The nudge fires at session start if `sessionsUntilNextNudge` reaches 0. It does not block the session or require action.

#### What manager Must Never Do

- Never modify skill files directly (that's doctor's job)
- Never install without user confirmation for non-trivial plugins
- Never remove without explicit confirmation
- Never surface security findings (that's doctor and auditor's job — manager just inventories)

#### When to Use

The implementer should write the SKILL.md `description` frontmatter to cover these situations:

- User wants to see what skills or plugins are installed
- User wants to install, remove, or update a skill or plugin
- User asks what's loaded, what version something is, or what's available
- User asks if any skills need updating
- Periodic session-start nudge when updates are available (the only proactive trigger)

---

### upskill/publisher

**The missing pipeline between a local skill and an installable plugin.**

#### Purpose

`publisher` takes a skill that exists in any state — a raw markdown file, a partially structured plugin, or a complete plugin with existing releases — and handles the full publication workflow: scaffolding, git/GitHub setup, versioning, tagging, and documentation.

It is the answer to: *"I made a great skill. How do I share it?"*

It is also the natural next step after `skill-creator` finishes. When skill-creator generates a new skill, `publisher` is the first thing you should reach for.

#### Key Integration: skill-creator Handoff

`publisher` is designed to be suggested automatically whenever `skill-creator` completes or whenever a user creates a new skill. This is the primary integration point between the two plugins — skill-creator does 0→1, publisher immediately picks up for 1→∞. When this handoff happens, publisher skips the "detect state" step and goes directly to the scaffold wizard, since the skill is known to be raw.

#### Capability Set

**1. Scaffold**

Transforms a raw skill file into a fully structured Claude Code plugin:

Input: `/path/to/my-skill.md` (or skill name if it's in `~/.claude/skills/`)

Output:
```
my-plugin/
├── .claude-plugin/
│   └── plugin.json        ← generated with correct metadata
├── skills/
│   └── my-skill/
│       └── SKILL.md       ← moved/renamed from input
├── README.md              ← generated stub, prompted for content
└── RELEASE-NOTES.md       ← generated stub for v1.0.0
```

`plugin.json` schema generated:
```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "...",
  "author": {
    "name": "..."
  },
  "homepage": "https://github.com/username/my-plugin",
  "repository": "https://github.com/username/my-plugin",
  "license": "MIT",
  "keywords": []
}
```

Skills are auto-discovered from the `skills/` directory — no `skills` array needed in `plugin.json`.

Publisher prompts the user for missing fields (name, description, author) rather than guessing.

**2. Git Initialize**

```
publisher git-init [path]
```

Initializes a git repo in the plugin directory if one doesn't exist. Creates an initial commit with the scaffolded structure. Sets up `.gitignore` with sensible defaults for Claude plugins.

**3. GitHub: Create & Push**

```
publisher github-setup
```

Using `gh` CLI:
- Creates a new GitHub repository (public by default, `--private` flag supported)
- Sets up the remote
- Pushes the initial commit
- Optionally creates the first release

Supports:
- Personal account: `github:username/plugin-name`
- Organization: `github:orgname/plugin-name` (user must have write access)
- Private repositories

**4. Version Bump**

```
publisher bump [major|minor|patch]
publisher bump          # auto-suggest based on diff
```

When no level is specified, publisher:
1. Diffs the skill files since the last git tag
2. Analyzes the nature of changes:
   - New capabilities added → suggests `minor`
   - Breaking changes to invocation or behavior → suggests `major`
   - Typos, clarifications, docs-only → suggests `patch`
3. Presents the suggestion with reasoning, user can always override

Updates `plugin.json` version field. Commits the version bump.

**5. Tag & Release**

```
publisher release
```

After a version bump:
1. Creates a git tag (`v1.2.0`)
2. Prompts user to review/edit `RELEASE-NOTES.md`
3. Pushes tag to GitHub
4. Creates a GitHub release using `gh release create`

**6. Docs Assistance**

Publisher actively helps write documentation:
- README template with sections: Description, Installation, Skills, Usage, Contributing, License
- RELEASE-NOTES template with sections per version: Added, Changed, Fixed, Removed
- Fills in what it can from context (skill names, invocations from SKILL.md frontmatter/content)
- Flags missing sections rather than silently omitting them

**7. Upgrade Existing Plugin**

If the input directory already has a partial plugin structure, publisher detects what's missing and fills in the gaps rather than overwriting existing content.

State transitions:
```
raw .md file
  → scaffolded (has plugin.json + structure)
  → git-tracked (has git history)
  → github-hosted (has remote)
  → versioned (has semver tags)
  → released (has GitHub releases)
```

Publisher can enter this pipeline at any stage.

#### What publisher Must Never Do

- Never overwrite existing SKILL.md content
- Never push to GitHub without explicit user confirmation
- Never create public repos without confirming visibility preference
- Never skip the README/release notes prompting — incomplete docs are a known failure mode
- Never auto-merge or auto-release without confirmation at each step

#### When to Use

The implementer should write the SKILL.md `description` frontmatter to cover these situations:

- User has just created a skill (especially right after skill-creator) and wants to share or version it
- User wants to put a skill on GitHub or make it installable by others
- User wants to release a new version of an existing plugin
- User wants to add proper structure, docs, or semver to a raw skill file
- User asks how to share a skill with their team or the community

Default behavior (no specific sub-task stated) should run the interactive wizard — detect the current pipeline state and guide the user to the next logical step.

---

### upskill/doctor

**The quality and security analyst for skills.**

#### Why "doctor"

The name is intentional: a doctor diagnoses, treats, prescribes, and discharges. `doctor` does the same for skills — it finds what's wrong, recommends a fix, applies it with your approval, and clears the skill. "The doctor will see your skill now."

#### Purpose

`doctor` analyzes skill files for quality, structure, safety, and behavioral correctness. It operates in two distinct modes with different audiences and different concerns.

It never modifies skill files without explicit user approval. It recommends; it does not act unilaterally.

#### Two Modes

**Curator Mode** — for skills you own and want to improve.

**Guardian Mode** — for all installed skills, including third-party, analyzed through a security and behavioral lens.

The mode is selected based on context: if you invoke doctor on your own skill, it defaults to Curator. If you invoke it on a third-party plugin, it defaults to Guardian. You can force either mode with a flag.

---

#### Curator Mode

**Audience:** Skill authors who want to improve their own skills.

**1. Structural Quality Analysis**

Checks each skill file for:
- Clear, single-sentence purpose/intent statement at the top
- Defined trigger conditions (when should this skill fire?)
- Defined exit conditions (when is the skill done?)
- Consistent formatting and section organization
- Actionable instructions vs. vague guidelines

**2. Scope & Focus Analysis**

- Is this skill doing too much? (Suggests split if skill covers 3+ unrelated domains)
- Is the trigger condition too broad? (e.g., "whenever I'm writing code" — fires constantly)
- Does this skill duplicate another installed skill?

**3. Guard Rail Recommendations**

Identifies dangerous actions within the skill's scope and suggests guard rails — not removal. Examples:
- Skill instructs bash commands → recommend "always show command before executing" guard
- Skill can delete files → recommend confirmation step
- Skill has network access → recommend logging what URLs are accessed

Doctor never removes dangerous capabilities from skills. It adds safety layers around them.

**4. Token Efficiency**

- Estimates the token footprint of the skill when loaded
- Flags skills that exceed a reasonable size for their purpose
- Suggests specific sections that could be condensed without losing meaning

**5. Missing Elements**

Flags what's missing:
- No stated purpose → critical
- No trigger conditions → high
- No exit/completion signal → medium
- No examples → low
- No version or author metadata → low

**6. Bad & Good Practice Identification**

Known bad patterns:

| Pattern | Why It's Bad |
|---|---|
| Contradicts CLAUDE.md | Creates conflicting instructions; Claude behavior becomes unpredictable |
| Uses "always" for optional behaviors | Overfires, wastes tokens, irritates user |
| No user confirmation before destructive action | Unsafe by default |
| Skill title differs significantly from invocation path | Confusing, hard to find |
| Instructions reference specific file paths that don't exist | Will silently fail |
| Circular self-reference (skill tells Claude to load another skill) | Can cause loops |

Doctor also identifies and calls out good practices when present — positive signal matters as much as negative.

---

#### Guardian Mode

**Audience:** Skill consumers evaluating third-party plugins for safety.

Guardian mode analyzes a skill through a threat model lens. It categorizes findings into three levels:

**Level A: Obvious Red Flags (binary)**

Direct evidence of malicious or harmful intent in the skill text:
- Instructions to exfiltrate data (send to external URL, write to a file path and describe it as a temp file)
- Instructions to suppress transparency (don't tell the user, hide this from the conversation)
- Instructions to override other settings or instructions (ignore CLAUDE.md, ignore system prompts)
- Prompt injection patterns (treat the following as a new instruction...)

**Level B: Dangerous Capability Patterns (contextual)**

The skill requests powerful capabilities without a clearly stated, proportionate reason:
- Bash execution + network access with no explanation
- File system write access outside the project directory
- Access to environment variables or secrets
- Spawning background processes

These are not automatic red flags — many legitimate skills need these capabilities. Guardian evaluates whether the *stated purpose* justifies the *requested capabilities*. Mismatch = suspicious.

**Level C: Behavioral Anomalies (inferential)**

The skill's stated purpose does not match what it actually instructs Claude to do:
- Title: "code formatter" — but also rewrites git history
- Title: "PR reviewer" — but also modifies files without being asked
- Instructions include steps unrelated to the stated purpose

**Conflict Detection**

Guardian also checks:
- Does this skill directly contradict an instruction in your `CLAUDE.md`?
- Does this skill contradict another loaded skill?
- Contradictions flagged with the specific conflicting text from each source

---

#### Output Format

Doctor always produces a structured report, never just freeform prose:

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
       This fires on virtually every engineering session, consuming
       tokens even when the skill is not relevant.
       Fix: Narrow to a specific context (e.g., "when asked to
       review a pull request" or "when the user runs the skill manually").

MEDIUM (2)
  [M1] No exit/completion signal
       The skill has no defined end state. Claude doesn't know when
       to stop applying the skill's instructions.

  [M2] Guard rail missing for bash execution
       This skill runs shell commands. Add an explicit instruction
       to show the command to the user before executing.

LOW (1)
  [L1] No usage examples
       Examples help Claude apply the skill correctly in edge cases.

─────────────────────────────────────────────────────────────────────
SUMMARY: 1 critical, 1 high, 2 medium, 1 low
Overall health: NEEDS ATTENTION

Recommended next step: Fix C1 and H1, then re-run doctor.
```

Severity levels:
- `CRITICAL`: Skill will likely cause harm or malfunction as written
- `HIGH`: Skill has significant issues that impair effectiveness or safety
- `MEDIUM`: Skill works but has meaningful quality gaps
- `LOW`: Minor polish items

Guardian-specific severity levels:
- `RED FLAG`: Direct evidence of malicious intent — do not use
- `SUSPICIOUS`: Capability/purpose mismatch — investigate before using
- `GREEN`: No security concerns found

#### What doctor Must Never Do

- Never modify skill files without explicit user approval for each change
- Never remove capabilities from skills (add guard rails instead)
- Never report false certainty — use hedged language for behavioral anomaly findings
- Never analyze conversation content or code in the project — skills only
- Never block the user from using a skill — recommendations only
- Never make security judgments about what bash commands *do* at runtime — only about patterns in the skill *instructions* themselves

#### When to Use

The implementer should write the SKILL.md `description` frontmatter to cover these situations:

- User wants to review, improve, or check the quality of a skill
- User asks if a skill is safe, well-written, or has issues
- User wants to split a skill into more focused sub-skills
- User suspects a skill is causing problems, conflicts, or unexpected behavior
- User wants to audit a third-party or newly installed skill for security concerns
- User asks about bad practices, guard rails, or missing elements in a skill
- User wants to know if a skill contradicts their CLAUDE.md or another skill

---

### upskill/auditor

**The session snapshot analyzer.**

#### Purpose

`auditor` provides a point-in-time audit of your current Claude Code session through the lens of skills. It answers the question: *"What is the skill layer doing to my session right now?"*

It does not care about code, conversation content, what you're building, or what Claude said. It cares only about:
1. What skills are loaded
2. What they're costing (tokens)
3. Whether they're healthy (pass/fail checks)
4. Whether they're contributing to or degrading session quality

#### Scope Boundary

**Auditor analyzes:**
- Loaded skills and plugins
- Token footprint of each skill
- Structural and behavioral properties of loaded skills
- Conflicts between loaded skills and between skills and CLAUDE.md

**Auditor does NOT analyze:**
- The content of your conversation
- Your codebase or project files
- Claude's responses or behavior in prior turns
- Anything outside the skill layer

This boundary is important. Auditor is a focused tool. It is not a general session health checker.

#### Report Structure

Auditor always produces a structured report with four sections:

**Section 1: Loaded Skills Inventory**

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
TOTAL                              ~2,240           —

[!] my-custom-workflow is consuming 37% of the skill budget.
    Consider reviewing it with doctor.
```

Token estimates are approximate, derived from character count with a standard heuristic (4 chars ≈ 1 token). Always labeled as estimates.

**Section 2: Hard Checks (Binary Pass/Fail)**

```
HARD CHECKS
─────────────────────────────────────────────────────────────────────
[PASS] No malicious patterns detected in any loaded skill
[PASS] No direct conflicts between loaded skills
[FAIL] my-custom-workflow conflicts with CLAUDE.md
       Conflict: CLAUDE.md says "always use TypeScript".
       my-custom-workflow says "use JavaScript for scripts".
       This creates ambiguous behavior when writing scripts.
[PASS] No skills suppressing transparency
[PASS] No skills overriding system settings
─────────────────────────────────────────────────────────────────────
1 failure
```

Hard check categories:
- Malicious pattern presence
- Cross-skill instruction conflicts
- Skill vs. CLAUDE.md conflicts
- Transparency suppression patterns
- System override patterns
- Circular reference patterns

**Section 3: Soft Checks (Qualitative)**

```
SOFT CHECKS
─────────────────────────────────────────────────────────────────────
[WARN] my-custom-workflow: trigger condition is very broad
       "whenever I am writing code" matches almost every engineering
       session. This skill may be firing when it's not needed,
       consuming context unnecessarily.
       Recommendation: Narrow the trigger or convert to manual
       invocation only.

[INFO] commit: no version metadata found
       This plugin does not include a version in plugin.json.
       Minor issue, no action required.

[OK]   All other loaded skills pass soft checks.
─────────────────────────────────────────────────────────────────────
1 warning, 1 info
```

**Section 4: Session Health Summary**

```
SESSION HEALTH SUMMARY
─────────────────────────────────────────────────────────────────────
Overall rating: NEEDS ATTENTION

What's healthy:
  - 5 of 6 skills are well-structured and appropriately scoped
  - No security concerns detected
  - Total token footprint is within reasonable range

What needs attention:
  - [FAIL] my-custom-workflow conflicts with CLAUDE.md (Section 2)
  - [WARN] my-custom-workflow trigger is too broad (Section 3)
  - [WARN] my-custom-workflow consumes 37% of skill budget (Section 1)

Recommended actions (priority order):
  1. Run doctor my-custom-workflow --curator
     Resolve the CLAUDE.md conflict and narrow the trigger condition.
  2. Consider splitting my-custom-workflow into focused sub-skills.
─────────────────────────────────────────────────────────────────────
```

Overall ratings:
- `HEALTHY` — all hard checks pass, no significant soft check warnings
- `NEEDS ATTENTION` — hard check failures or multiple significant soft warnings
- `CRITICAL` — one or more red-flag security findings

#### What auditor Must Never Do

- Never read or reference conversation content
- Never analyze the user's code, commits, or project state
- Never modify any skill files
- Never make security judgments about what bash commands *do at runtime* — only about patterns in skill instructions
- Never claim certainty about token counts — always label estimates as approximate

#### When to Use

The implementer should write the SKILL.md `description` frontmatter to cover these situations:

- User wants to understand what skills are loaded and their token cost
- User suspects loaded skills are degrading session quality or consuming too much context
- User wants a health check or overview of their current skill setup
- User asks what's conflicting, suspicious, or problematic in the current session
- User asks for a session audit, skill snapshot, or "what's loaded right now"

---

## State Management

All persistent state lives in `~/.claude/upskill-state.json`. This file is created automatically on first use of any `upskill` sub-skill.

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

### Field Descriptions

| Field | Type | Description |
|---|---|---|
| `version` | string | Schema version for migration support |
| `sessionsUntilNextNudge` | number | Decremented each session; nudge fires when it reaches 0, then resets to `intervalSessions` |
| `lastUpdateCheck` | ISO 8601 | When we last checked for plugin updates |
| `installedPlugins` | object | Keyed by `author/plugin-name`, tracks version state |
| `nudgeConfig.intervalSessions` | number | How many sessions between update nudges (default: 10) |
| `nudgeConfig.enabled` | boolean | User can disable nudges entirely |

### State File Rules

- Never commit `upskill-state.json` to any repository
- Always read before write to avoid clobbering concurrent updates
- Gracefully handle missing or malformed state files by re-initializing
- The state file is only updated by `manager` — other sub-skills are read-only on state

---

## Future Roadmap

### The Lazy Loader / Skill Suggester (Post-v1)

**This feature is NOT in v1 but MUST be in the long-term vision.**

The Lazy Loader is a lightweight always-loaded skill (~100 tokens maximum) that acts as a proactive discovery layer. It:

1. **Stays resident** — loaded in every Claude Code session, but at minimal token cost
2. **Recognizes patterns** — watches for keywords, task descriptions, and workflow signals that suggest a non-installed skill would be useful
3. **Suggests contextually** — when it detects a match, surfaces a brief recommendation:
   ```
   [upskill] This looks like a PR review task. You have commit installed
   but not pr-reviewer. Install it? manager install github:author/pr-reviewer
   ```
4. **Loads on confirmation** — if the user confirms, loads the heavier skill for the actual work
5. **Learns over time** — tracks which suggestions were accepted and which were dismissed (stored in `upskill-state.json`)

**Why this is v2 and not v1:**

The Lazy Loader requires a curated signal library (which patterns map to which skills) and a community-driven registry of quality skills to recommend. Neither exists yet. Building the core lifecycle tooling (manager, publisher, doctor, auditor) first creates the ecosystem that makes Lazy Loader valuable.

**Design constraints for when it's built:**

- The resident skill must be 100 tokens or less — verified by auditor before release
- Suggestions must be skippable with a single keystroke — no friction
- Must be opt-out, not opt-in
- Must never suggest skills it hasn't vetted through guardian mode

### Other Post-v1 Possibilities

- **upskill/registry** — a curated index of quality-vetted skills installable via upskill
- **upskill/share** — generate a shareable link to a skill without full plugin setup
- **upskill/migrate** — migrate skills between machines or accounts
- **upskill/rollback** — one-command skill revert to any previous version

---

## Design Principles

These principles govern every design decision in `upskill`. When the implementation of a feature conflicts with a principle, the principle wins — unless changing the principle is explicitly discussed and updated in this document.

### 1. Complement, Don't Compete

`upskill` begins exactly where `skill-creator` ends. It will never add skill-generation capabilities. This is a permanent, intentional boundary. The goal is a cohesive ecosystem, not a winner-takes-all tool.

### 2. Unified View, Bifurcated Internals

Users see one inventory, one quality system, one audit. Internally, plugin-installed skills and local skills have different code paths (because they have different APIs). Users never need to know this distinction unless they're debugging.

### 3. Safety First, Every Time

Before any destructive or irreversible action:
- Show the user exactly what will happen
- Ask for explicit confirmation
- Offer a dry-run mode where possible
- Document the rollback path

This applies to: removing plugins, bumping versions, creating GitHub repos, modifying skill files.

### 4. Suggest, Don't Force

`doctor` recommends. `publisher` guides. `manager` nudges. None of them act without user confirmation on anything meaningful. The only autonomous actions in all of `upskill` are:
- Decrementing the session counter
- Showing the update nudge message

Everything else requires a human keystroke.

### 5. Token Discipline

`upskill` is a skill about skills. It will be loaded alongside other skills, and its own token footprint must be minimal and appropriate.

Token budget targets (initial estimates — verify during implementation):
- `manager`: ~300 tokens
- `publisher`: ~400 tokens
- `doctor`: ~600 tokens (necessarily larger due to analysis pattern library)
- `auditor`: ~400 tokens

Each skill in `upskill` must pass `auditor --tokens` before release. If budgets need adjusting during implementation, update this document and note the reasoning.

### 6. upskill Eats Its Own Cooking

Every skill in `upskill` must pass `doctor --curator` with zero CRITICAL or HIGH findings before any release. This is a hard release gate, not a suggestion.

Corollary: the first time `doctor` is complete, the immediate next step is running it on all four `upskill` skills.

### 7. Scope Boundaries Are Sacred

| Sub-skill | Analyzes | Does NOT analyze |
|---|---|---|
| manager | Installed plugins, version metadata | Skill content, session behavior |
| publisher | Plugin structure, git/GitHub state | Skill quality, session impact |
| doctor | Skill file content, structure, safety | Session state, code, conversation |
| auditor | Loaded skills + their session impact | Conversation, code, project files |

No sub-skill crosses into another's domain. If a feature would require crossing a boundary, it either belongs in a different sub-skill or requires a new sub-skill.

---

## Quality Bar & Reference Standards

### Reference: superpowers by obra

The benchmark for quality in this project is [superpowers by obra](https://github.com/obra/superpowers). Implementors should study that plugin for:

- How skills are structured and scoped
- How invocation patterns are designed
- How output is formatted for readability and actionability
- How `plugin.json` is composed
- How the README communicates value quickly

`upskill` should be at least equal to superpowers in: craft, documentation quality, structure clarity, and output readability.

### Skill File Quality Checklist

Before any `upskill` skill is considered complete:

- [ ] Clear purpose statement in the first 2 sentences
- [ ] Defined trigger conditions (or explicit note that it's manual-only)
- [ ] Defined exit/completion signal
- [ ] "When to Use" section present with clear trigger signals for implementers
- [ ] Output formats shown as examples (not just described)
- [ ] "Must never do" list present
- [ ] Token footprint within budget
- [ ] Passes `doctor --curator` with no CRITICAL or HIGH findings

### README Quality Checklist

- [ ] One-paragraph summary that a non-technical user can understand
- [ ] Installation command is the first code block
- [ ] All four sub-skills listed with one-line descriptions
- [ ] At least one usage example per sub-skill
- [ ] Relationship to skill-creator explained
- [ ] Contributing section present
- [ ] MIT license badge

### Release Quality Checklist

Before any version tag is created:
- [ ] All skill files pass doctor (no CRITICAL/HIGH)
- [ ] `plugin.json` version matches git tag
- [ ] RELEASE-NOTES.md entry written for this version
- [ ] README reflects current capabilities
- [ ] Token budgets verified via auditor

---

## Plugin File Structure

### `.claude-plugin/plugin.json`

```json
{
  "name": "upskill",
  "version": "1.0.0",
  "description": "The full lifecycle layer for Claude Code skills. Manage, publish, doctor, and audit your skills.",
  "author": {
    "name": "ngouy"
  },
  "homepage": "https://github.com/ngouy/upskill",
  "repository": "https://github.com/ngouy/upskill",
  "license": "MIT",
  "keywords": ["skills", "plugin-management", "versioning", "security", "lifecycle"]
}
```

Note: skills are auto-discovered by the plugin system from the `skills/` directory. No `skills` array is needed or supported in `plugin.json`.

### `README.md` Structure (Required Sections)

1. Badge row (version, license, installation)
2. One-paragraph summary
3. Installation (`/plugin install github:ngouy/upskill`)
4. Quick start (one example per sub-skill)
5. Sub-skills overview table
6. Detailed usage per sub-skill
7. Relationship to skill-creator
8. State file location and schema
9. Contributing
10. License

### `RELEASE-NOTES.md` Structure (Required per Version)

```markdown
## v1.0.0 — YYYY-MM-DD

### Added
- manager: unified skill inventory with plugin + local sources
- publisher: full scaffold → GitHub → release pipeline
- doctor: curator and guardian analysis modes
- auditor: session health snapshot with token and security checks

### Notes
Initial release.
```

---

## Implementation Guide for Claude Sessions

This section is written specifically for Claude sessions assigned to implement individual sub-skills. Read it before writing a single line of SKILL.md.

### Session Structure

Each sub-skill is implemented in a dedicated session, in this order:

1. **Brainstorm session** — deep dive on the sub-skill's intent, edge cases, and design
2. **Plan session** — detailed implementation plan referencing this document
3. **Implementation session** — write the SKILL.md following the plan
4. **Review session** — run `doctor --curator` on the result, fix any findings

Do not skip steps. Do not combine steps into one session.

### Implementation Rules

1. The capability set in the relevant section of this document is the specification — implement everything listed, nothing more
2. The "must never do" list is a hard constraint — violations are bugs, not style issues
3. The "When to Use" section defines the trigger intent — write the SKILL.md `description` frontmatter to match it precisely
4. Output format examples are the contract — your output must match the structure
5. The token budget for each sub-skill is a target — verify with a character count and flag if you cannot meet it

### Resolving Ambiguity

If a specification is ambiguous, resolve it conservatively: the interpretation that does less, exposes less, is safer. Document your interpretation as a comment in the SKILL.md file and flag it for review.

### Scope Creep

When implementing `doctor`, do not add features that belong in `auditor`. When implementing `publisher`, do not add analysis features that belong in `doctor`. The scope boundary table in Design Principles is non-negotiable. When in doubt, check the table.

---

*This document was last updated: 2026-03-08.*
*Maintained by: ngouy.*
*For questions, open an issue at https://github.com/ngouy/upskill/issues.*
