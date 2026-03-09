# Release Notes

## v0.1.0-alpha — 2026-03-08

### Added
- Project vision and architecture (`VISION.md`)
- Build instructions for Claude sessions (`CLAUDE.md`)
- Plugin scaffold (`.claude-plugin/plugin.json`)

### Design Decisions (from critical review)
- v1 focuses on two skills: `publisher` and `doctor`
- `manager` and `auditor` deferred to v1.1+
- Project-level skills (`.claude/skills/`) are first-class across all sub-skills
- Watermark is opt-in (default off), asked during publisher scaffold
- Publisher uses external templates (read on demand, zero token cost when not invoked)
- Periodic update nudge implemented via Claude Code hook, not session-start magic
- Token budgets are aspirational targets, not hard limits
- Lazy Loader concept preserved for future exploration with documented constraints

### Notes
Pre-release. No skills implemented yet — this release captures the reviewed and finalized design.

---

## v1.0.0 — TBD

### Planned
- `publisher` — full scaffold → git → GitHub → versioning → release pipeline
- `doctor` — skill quality and security analyst (Curator and Guardian modes)
