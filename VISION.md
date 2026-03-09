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

### Key Concepts: Plugin vs. Skill

A **plugin** is a package — a repository with `.claude-plugin/plugin.json` that can be installed via `/plugin install`. A **skill** is a single SKILL.md file inside a plugin's `skills/` directory. One plugin can contain many skills (e.g., `upskill` is one plugin containing four skills).

Users can also have **standalone local skills** — raw `.md` files not part of any plugin. These live in two places:

- **Global skills** — `~/.claude/skills/` — personal skills available in every session
- **Project skills** — `<project>/.claude/skills/` — team skills shared via the repo, available to everyone who clones it

Project-level skills are a primary use case for `upskill`. Teams write skills in their repos (deploy helpers, review checklists, coding standards) and every team member gets them automatically. These skills are the most likely to be written quickly and never curated — which is exactly what `doctor` is for.

All three skill sources (plugin-installed, global local, project-level) are first-class citizens across all `upskill` sub-skills. `/plugin list` only shows plugins — `upskill` shows everything.

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
skill-creator  →  publisher  →  doctor  →  auditor
 (create it)       (ship it)    (improve it)  (check session health)
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
│   │   ├── SKILL.md
│   │   └── templates/          ← scaffold templates, read on demand
│   │       ├── plugin.json.template
│   │       ├── gitignore.template
│   │       ├── readme.md.template
│   │       └── release-notes.md.template
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

`manager` provides a single, unified view of every skill loaded into the current Claude Code environment — regardless of whether it came from a plugin or was created locally.

There is no built-in way to see all skills from all sources in one place. `/plugin list` shows plugins but not global or project-level skills. `manager` merges all three sources into one inventory. The differences between plugin-installed, global, and project-level skills are implementation details — from the user's perspective, there is one list.

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
- Source: `plugin` (managed by Claude Code), `local` (global skills in `~/.claude/skills/`), or `project` (project skills in `<project>/.claude/skills/`)
- Version: semver for plugins, `(untracked)` for local/project files with no git history, or git short SHA if in a git repo
- Last updated: human-readable relative time
- Update available badge (when detected)

For local `(untracked)` skills, manager surfaces that they have no version history — this is a natural prompt toward `publisher` for users who want to share or version their skills.

**2. Bulk Operations**

Manager adds value over raw `/plugin` commands by orchestrating multi-plugin operations and providing consolidated feedback:

- **Update all:** "Update all my skills" — runs updates across all installed plugins in one pass, shows a consolidated summary of what changed (including RELEASE-NOTES.md diffs where available)
- **Post-install nudge:** After any install, offer to run `doctor` on the newly installed plugin — this is the security gate moment. User can decline.

Manager does not wrap individual `/plugin install` or `/plugin remove` — those work fine as-is. Manager's value is in the operations that span multiple plugins and the post-action intelligence (summaries, doctor suggestions) that the built-in commands don't provide.

**3. Periodic Update Nudge (via Claude Code hook)**

Manager can help the user set up a Claude Code hook that checks for plugin updates at session start. The hook runs outside of the skill system and feeds its output back into the session.

When updates are available, the hook surfaces a brief, non-blocking message:

```
[upskill] 2 skill updates available. Ask manager to review.
```

**Implementation:** The skill does not run at session start automatically — skills are passive instructions, not code. Instead, manager offers to install a Claude Code hook (`~/.claude/hooks/`) that performs the update check. The hook reads `~/.claude/upskill-state.json` to determine when to check and what to report.

- Hook setup is optional and user-initiated ("set up update notifications")
- The hook respects `nudgeConfig.enabled` and `nudgeConfig.intervalSessions` from the state file
- Manager explains what the hook does before installing it

**Note:** Install, remove, and update operations are handled by Claude Code's built-in `/plugin install`, `/plugin remove`, and `/plugin update` commands. Manager does not wrap these — it focuses on the unified inventory view that those commands don't provide.

#### What manager Must Never Do

- Never modify skill files directly (that's doctor's job)
- Never surface security findings (that's doctor and auditor's job — manager just inventories)
- Never install hooks without explaining what they do and getting user confirmation

#### When to Use

The implementer should write the SKILL.md `description` frontmatter to cover these situations:

- User wants to see what skills or plugins are installed
- User asks what's loaded, what version something is, or what's available
- User asks about skill sources (plugin vs. local)
- User wants to set up or configure update notifications

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

**Templates are external files, not embedded in the SKILL.md.** Publisher's SKILL.md contains slim instructions that reference template files stored alongside it in the plugin repo. Claude reads the templates only when publisher is actually invoked — they cost zero tokens in sessions where publisher isn't used.

```
upskill/
└── skills/
    └── publisher/
        ├── SKILL.md                        ← slim instructions (loaded every session)
        └── templates/
            ├── plugin.json.template        ← read on demand
            ├── gitignore.template
            ├── readme.md.template
            └── release-notes.md.template
```

The SKILL.md instructs Claude to: read the relevant template, fill in user-provided values (name, description, author), and write the result. Publisher prompts the user for missing fields rather than guessing.

Skills are auto-discovered from the `skills/` directory — no `skills` array needed in `plugin.json`.

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

**8. Project-Level Skill Management**

For skills that live in a project's `.claude/skills/` directory, publisher operates differently — these skills already live inside a repo, so the workflow is about managing them in-place rather than creating standalone plugins:

- **Version tracking:** Publisher adds lightweight semver metadata to the skill's frontmatter (no separate `plugin.json` needed for project skills)
- **PR workflow:** When a project skill is updated, publisher creates a PR on the project repo with the changes rather than pushing to a standalone plugin repo
- **Watermark:** If opt-in, adds `maintained-with` to the skill's frontmatter

Project-level skills are the primary use case for teams — deploy helpers, review checklists, coding standards. Publisher treats them as first-class, not as second-class citizens that need to be "promoted" to plugins.

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
- User wants to manage or version a project-level skill (`.claude/skills/`)
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

> **Future consideration (not v1):** Deep scan mode — opt-in analysis of referenced script files (`.sh`, etc.) in addition to SKILL.md. Deferred because loading every script file in every plugin could be token-prohibitive. When designed, should be a separate flag (e.g. `--deep`) that the user explicitly requests, not part of default doctor behavior.

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

**The cross-skill session analyzer.**

#### Purpose

`auditor` answers one question: *"How are my loaded skills interacting with each other and affecting my session right now?"*

It does not analyze individual skill quality (that's doctor's job). It analyzes the **relationships between** loaded skills — conflicts, overlaps, token budget distribution, and overall session health.

#### How It Differs From doctor

This is the most important distinction to get right:

| | doctor | auditor |
|---|---|---|
| Scope | One skill at a time | All loaded skills together |
| Question | "Is this skill well-written and safe?" | "How do my loaded skills interact with each other?" |
| Finds | Quality issues, bad patterns, security risks in a single skill | Conflicts between skills, overlapping triggers, token bloat across the session |
| Output | Per-skill findings with fixes | Session-level snapshot with cross-skill analysis |

They are complementary: auditor might say "my-workflow and commit-helper have overlapping triggers" — then you'd run `doctor` on each individually to decide which to narrow.

#### Scope Boundary

**Auditor analyzes:**
- All loaded skills and how they relate to each other
- Token footprint of each skill and total session cost
- Cross-skill conflicts (skill A says X, skill B says Y)
- Skill vs. CLAUDE.md conflicts
- Trigger overlap (two skills firing on the same context)

**Auditor does NOT analyze:**
- Individual skill quality (no structural checks, no missing-element checks — that's doctor)
- Per-skill security patterns (no Guardian-mode checks — that's doctor)
- The content of your conversation
- Your codebase or project files
- Claude's responses or behavior in prior turns

This boundary is hard and intentional. Auditor is the cross-skill observer. Doctor is the per-skill analyst.

#### Report Structure

Auditor always produces a structured report with three sections:

**Section 1: Token Inventory**

```
LOADED SKILLS
─────────────────────────────────────────────────────────────────────
Skill                    Source    Tokens (est.)    % of total
─────────────────────────────────────────────────────────────────────
manager                  plugin    ~180             8%
publisher                plugin    ~240             11%
doctor                   plugin    ~520             23%
auditor                  plugin    ~310             14%
commit                   plugin    ~150             7%
my-custom-workflow       local     ~840             37%
─────────────────────────────────────────────────────────────────────
TOTAL                              ~2,240

[!] my-custom-workflow is consuming 37% of the skill token budget.
    Consider reviewing it with doctor.
```

Token estimates are approximate, derived from character count with a standard heuristic (4 chars ≈ 1 token). Always labeled as estimates.

**Section 2: Cross-Skill Analysis**

This is auditor's unique contribution — analysis that no single-skill tool can provide.

```
CROSS-SKILL ANALYSIS
─────────────────────────────────────────────────────────────────────
[CONFLICT] my-custom-workflow ↔ CLAUDE.md
           CLAUDE.md says "always use TypeScript".
           my-custom-workflow says "use JavaScript for scripts".
           This creates ambiguous behavior when writing scripts.

[OVERLAP]  my-custom-workflow ↔ commit
           Both trigger on "when I am working on code changes".
           These skills may compete for the same context.
           Recommendation: Narrow one or both triggers.

[OK]       No circular references detected.
[OK]       No other cross-skill conflicts found.
─────────────────────────────────────────────────────────────────────
1 conflict, 1 overlap
```

Cross-skill check categories:
- Skill vs. CLAUDE.md instruction conflicts (with specific text quoted from both sides)
- Skill vs. skill instruction conflicts
- Trigger overlap (two or more skills with near-identical activation conditions)
- Circular references (skill A loads skill B which loads skill A)

**Section 3: Session Health Summary**

```
SESSION HEALTH SUMMARY
─────────────────────────────────────────────────────────────────────
Overall rating: NEEDS ATTENTION

What's healthy:
  - 5 of 6 skills have reasonable token footprints
  - No circular references detected

What needs attention:
  - [CONFLICT] my-custom-workflow conflicts with CLAUDE.md
  - [OVERLAP]  my-custom-workflow and commit have overlapping triggers
  - [WARN]     my-custom-workflow consumes 37% of the skill token budget

Recommended actions (priority order):
  1. Run doctor my-custom-workflow to resolve the CLAUDE.md conflict.
  2. Narrow the trigger on my-custom-workflow or commit to eliminate overlap.
─────────────────────────────────────────────────────────────────────
```

Overall ratings:
- `HEALTHY` — no conflicts, no overlaps, token distribution reasonable
- `NEEDS ATTENTION` — conflicts or overlaps detected, or one skill dominates the token budget
- `CRITICAL` — CLAUDE.md conflict detected (this directly affects Claude's behavior)

#### What auditor Must Never Do

- Never perform per-skill quality analysis (no structural checks, no missing-element checks — use doctor)
- Never perform per-skill security analysis (no Guardian-mode checks — use doctor)
- Never read or reference conversation content
- Never analyze the user's code, commits, or project state
- Never modify any skill files
- Never claim certainty about token counts — always label estimates as approximate
- Never duplicate doctor's work — when a finding requires per-skill analysis, recommend running doctor

#### When to Use

The implementer should write the SKILL.md `description` frontmatter to cover these situations:

- User wants to understand what skills are loaded and their token cost
- User suspects loaded skills are conflicting with each other or with CLAUDE.md
- User notices overlapping behavior between skills and wants to identify the overlap
- User wants a session-level health check for their skill setup
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
  },
  "watermark": {
    "enabled": false
  }
}
```

### Field Descriptions

| Field | Type | Description |
|---|---|---|
| `version` | string | Schema version for migration support |
| `sessionsUntilNextNudge` | number | Decremented by the hook each session; update check fires when it reaches 0, then resets to `intervalSessions` |
| `lastUpdateCheck` | ISO 8601 | When the update-check hook last ran |
| `installedPlugins` | object | Keyed by `author/plugin-name`, tracks version state |
| `nudgeConfig.intervalSessions` | number | How many sessions between update checks (default: 10, used by the hook) |
| `nudgeConfig.enabled` | boolean | User can disable nudges entirely |
| `watermark.enabled` | boolean | Whether to add upskill attribution to managed skills and published READMEs (default: false, opt-in) |

### State File Rules

- Never commit `upskill-state.json` to any repository
- Always read before write to avoid clobbering concurrent updates
- Gracefully handle missing or malformed state files by re-initializing
- The state file is written by `manager` and the update-check hook — other sub-skills are read-only on state

---

## Future Roadmap

### The Lazy Loader / Skill Suggester (Post-v1)

**This feature is NOT in v1. It is an experimental idea for future exploration.**

The core concept: a lightweight router skill that detects what domain the user is working in and dynamically loads relevant sub-skills. Instead of loading all skills upfront, only load what's needed.

**The idea (cascading loader):**

1. A small router skill (~100 tokens) is always loaded
2. It analyzes the session context to determine the domain (skill management, code review, deployment, etc.)
3. Based on the domain, it instructs Claude to read a category-specific skill file
4. Each category file contains curated references to specific skills in that domain

**Known technical constraints:**

- **Skills are passive text, not code.** A skill cannot trigger itself or "watch" for patterns. It relies on Claude recognizing the match between its description and the user's context.
- **Dynamically loaded skills don't survive context compression.** Skills loaded via file reads mid-session may be dropped when the context window compresses. Only properly installed skills are re-injected after compression.
- **A registry is needed for ecosystem-wide suggestions.** Suggesting skills the user doesn't have requires knowing what skills exist. A curated catalog is a separate project.
- **The 100-token budget is tight.** The router would need to categorize into domains, and even a small set of categories with trigger patterns approaches the limit.

**Simpler alternatives worth exploring first:**

- Suggest skills the user already has installed but hasn't triggered
- Suggest project-level skills to teammates who haven't used them yet
- A small, hand-curated "starter pack" list of well-known community plugins, updated with each upskill release

**Why this is post-v1:** The core lifecycle tooling (publisher, doctor) must exist first. The Lazy Loader's value depends on an ecosystem of quality skills worth suggesting — which is what v1 builds toward.

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

`doctor` recommends. `publisher` guides. `manager` nudges. None of them act without user confirmation on anything meaningful. The only autonomous behavior in `upskill` is the optional update-check hook — which the user explicitly installs and can disable at any time.

Everything else requires a human keystroke.

### 5. Token Discipline

`upskill` is a skill about skills. It will be loaded alongside other skills, and its own token footprint must be minimal and appropriate.

**What "token budget" means:** Every SKILL.md is loaded into Claude's context window at the start of every session. The full text — instructions, examples, pattern tables, everything — consumes context whether or not the skill is used in that session. Tokens spent on skill instructions are tokens not available for conversation, code, and tool results. This is the fundamental cost of having a skill installed.

Token budget targets (aspirational — expect 50-100% overrun during implementation):
- `manager`: ~300 tokens (~1,200 characters)
- `publisher`: ~400 tokens (~1,600 characters)
- `doctor`: ~800 tokens (~3,200 characters) — necessarily larger due to analysis pattern library and dual-mode specification
- `auditor`: ~400 tokens (~1,600 characters)

These targets are starting points, not hard limits. During implementation, if a skill cannot meet its target without becoming too vague to be useful, the implementer should document the actual size and the reasoning for the overrun. The priority is: useful and precise instructions first, token savings second.

Each skill in `upskill` should be reviewed for token efficiency before release. If budgets need adjusting during implementation, update this document and note the reasoning.

### 6. upskill Eats Its Own Cooking

Every skill in `upskill` must pass `doctor --curator` with zero CRITICAL or HIGH findings before any release. This is a hard release gate, not a suggestion.

Corollary: the first time `doctor` is complete, the immediate next step is running it on all four `upskill` skills.

### 7. Attribution is Opt-In

Skills managed or published by upskill can optionally carry attribution in two places:

1. **SKILL.md frontmatter** — a `maintained-with` field:
   ```yaml
   maintained-with: upskill (github.com/ngouy/upskill)
   ```

2. **README footer** — for published plugins, a small attribution line:
   ```
   ---
   *Managed with [upskill](https://github.com/ngouy/upskill)*
   ```

Attribution is **opt-in**. During `publisher scaffold`, the user is asked: "Add upskill attribution? [y/N]". The answer is stored in `watermark.enabled` in `upskill-state.json` (default: `false`). Attribution is never added silently — consistent with Principle 4 (Suggest, Don't Force).

The watermark is never added to third-party skills — only to skills the user owns and manages through upskill.

### 8. Scope Boundaries Are Sacred

| Sub-skill | Analyzes | Does NOT analyze |
|---|---|---|
| manager | Unified skill inventory (plugins + local + project), bulk updates, update notifications | Skill content, per-skill quality, security analysis |
| publisher | Plugin structure, git/GitHub state, project skill PRs | Skill quality, session impact |
| doctor | Individual skill quality, structure, safety (curator + guardian) | Cross-skill analysis, session state, code, conversation |
| auditor | Cross-skill analysis: conflicts, overlaps, token budget | Per-skill quality (use doctor), conversation, code, project files |

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

### When VISION.md Is Wrong

If implementation reveals that a feature as specified is technically infeasible or contradicts another principle, do not silently skip it or hack around it. Instead: document the conflict, propose an alternative, and flag it for review. The implementer is expected to update VISION.md when the spec is demonstrably wrong — that is not "unilateral modification," it is a necessary correction. What is prohibited is adding new features or changing scope without discussion.

### Scope Creep

When implementing `doctor`, do not add features that belong in `auditor`. When implementing `publisher`, do not add analysis features that belong in `doctor`. The scope boundary table in Design Principles is non-negotiable. When in doubt, check the table.

---

*This document was last updated: 2026-03-08.*
*Maintained by: ngouy.*
*For questions, open an issue at https://github.com/ngouy/upskill/issues.*
