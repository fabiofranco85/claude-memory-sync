# Memory File Patterns Reference

This file contains structural patterns and examples for well-formed Claude memory files.

---

## CLAUDE.md Structure

A good `CLAUDE.md` is organized by functional area and uses clear headings. It should be
readable top-to-bottom and give Claude essential project context fast.

```markdown
# Project: <Name>

## Overview
One-paragraph description: what the project does, its primary language/stack, and its purpose.

## Architecture
High-level structure - key directories, major modules, how they connect.

## Development Setup
How to install, configure, and run the project locally. Include all required env vars.

### Required Environment Variables
| Variable | Purpose | Example |
|---|---|---|
| DATABASE_URL | Postgres connection | postgres://localhost/myapp |
| JWT_SECRET | Token signing | (generate with `openssl rand -hex 32`) |

## Common Commands
```bash
npm run dev        # Start dev server
npm run test       # Run test suite
npm run build      # Production build
```

## Key Conventions
- Naming patterns, code style decisions, important architectural choices.

## External Services
Services the project depends on and how they're configured.

---
<!-- claude-memory-sync: 2025-11-04T14:32:00Z -->
```

---

## Rules File with Frontmatter

Files under `.claude/rules/` may declare scope with YAML frontmatter.

```markdown
---
applies_to: ["src/frontend/**", "*.tsx", "*.css"]
context: frontend
---

# Frontend Rules

## Component Conventions
...

---
<!-- claude-memory-sync: 2025-11-04T14:32:00Z -->
```

The `applies_to` field is a list of glob patterns. Claude Memory Sync will only update this file
if at least one changed file matches at least one of the globs.

The `context` field is a human-readable label used in consistency checks and the sync report.

---

## Scoped Rules File (no frontmatter)

If a rules file has no frontmatter, it is treated as **global** and always included in the sync
verification pass, regardless of which files changed.

```markdown
# General Code Conventions

## Commit Messages
Follow conventional commits: `feat:`, `fix:`, `chore:`, `docs:`, `refactor:`.

## PR Requirements
...

---
<!-- claude-memory-sync: 2025-11-04T14:32:00Z -->
```

---

## Timestamp Placement

The timestamp block MUST be the absolute last content in the file, after a blank line and a
final `---` separator.

**Correct:**
```markdown
...last section content...

---
<!-- claude-memory-sync: 2025-11-04T14:32:00Z -->
```

**Incorrect (no blank line before ---):**
```markdown
...last section content...
---
<!-- claude-memory-sync: 2025-11-04T14:32:00Z -->
```

**Incorrect (timestamp buried mid-file):**
```markdown
<!-- claude-memory-sync: 2025-11-04T14:32:00Z -->

## More content here
```

If a file has a timestamp anywhere other than the end, move it to the end during the sync pass.

---

## New Context File Template

When creating a new `.claude/rules/<domain>.md` file:

```markdown
---
applies_to: ["<glob1>", "<glob2>"]
context: <domain>
---

# <Domain> Context

## Overview
Brief description of this domain within the project.

## <Relevant Section>
...

---
<!-- claude-memory-sync: <ISO8601_UTC_TIMESTAMP> -->
```

Always add a cross-reference in `CLAUDE.md`:

```markdown
## Context Files
- `.claude/rules/<domain>.md` - <one-line description>
```

---

## Change Impact Classification

Use this table to decide whether a change requires a memory update:

| Change | Memory impact |
|---|---|
| New file in `src/` with new exported API | High - document in architecture section |
| Deleted file in `src/` | High - remove all references |
| New env var in `.env.example` | High - must be listed in CLAUDE.md |
| Removed env var | High - must be removed from CLAUDE.md |
| Renamed function (internal) | Low - usually not in memory |
| New `package.json` script | Medium - if user-facing, add to commands |
| Dependency version bump | Low - usually skip |
| New CI pipeline step | Medium - add if it changes dev workflow |
| New top-level directory | High - mention in architecture |
| Test file added | Low - skip unless testing approach changes |
| Config schema change (e.g., new required field) | High - document in setup section |
| Removed feature/module | High - remove all references |
