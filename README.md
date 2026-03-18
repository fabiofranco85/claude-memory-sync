# claude-memory-sync

A Claude Code plugin that keeps your project's memory files (`CLAUDE.md`, `.claude/rules/*.md`,
and related files) synchronized with the actual state of your codebase.

---

## The problem

Claude's memory files go stale. You add a new environment variable, rename a module, remove a
deprecated endpoint - but `CLAUDE.md` still describes the old world. Over time, Claude works
from an increasingly inaccurate picture of your project, missing context it should have and
confidently referencing things that no longer exist.

## What this skill does

When you ask Claude to sync its memory, this skill:

1. **Finds your memory files** - `CLAUDE.md` at any depth, `.claude/rules/*.md`, and any other
   files they cross-reference.
2. **Establishes a baseline** - reads each file's last-sync timestamp to know exactly how far
   back to look for changes.
3. **Detects what changed** - runs both a `git log` sweep and a filesystem mtime scan, then
   merges and deduplicates the results.
4. **Reads the diffs** - classifies each change as trivial or memory-relevant and summarizes
   the impact.
5. **Maps changes to memory files** - figures out which sections of which files are affected,
   respecting scope frontmatter that restricts rules files to specific parts of the codebase.
6. **Verifies each file** - goes section by section checking accuracy, staleness, and gaps.
7. **Proposes a change list** - additions, updates, removals, and moves.
8. **Safety-gates large removals** - presents risky changes as a single consolidated list for
   your review before writing anything.
9. **Applies edits surgically** - minimum changes, preserving your existing style and structure.
10. **Creates new context files** - if a new subsystem is large enough to warrant its own
    `.claude/rules/<domain>.md`, it creates one.
11. **Runs a cross-file consistency review** - checks for contradictions, orphaned references,
    duplicated content, and scope mismatches.
12. **Reports what it did** - a concise summary of every change made, removed, and flagged.

The core principle: **Claude's memory files must reflect the project as it IS, not as it WAS.**

---

## Installation

### From GitHub (recommended)

```
/plugin marketplace add <your-username>/claude-memory-sync
/plugin install claude-memory-sync@claude-memory-sync
```

Then reload:

```
/reload-plugins
```

### Manual install (no GitHub required)

Copy the skill folder directly into your global skills directory:

```bash
cp -r plugins/claude-memory-sync/skills/claude-memory-sync ~/.claude/skills/
```

---

## Usage

Once installed, trigger the skill with natural language in any Claude Code session:

```
sync Claude memory
update CLAUDE.md
refresh Claude's project knowledge
check if Claude files are up to date
the project has changed, Claude doesn't know about it
run a memory audit
```

Claude will run all phases automatically and present a sync report at the end.

### Force a full review

If you want Claude to re-verify everything even if nothing appears to have changed:

```
force a full Claude memory sync
```

---

## How the timestamp system works

Every memory file managed by this skill ends with a metadata block:

```markdown

---
<!-- claude-memory-sync: 2026-03-18T14:00:00Z -->
```

This timestamp records when the file was last verified. On the next sync, the skill uses the
**oldest** timestamp across all your memory files as the baseline for change detection - so if
any file is behind, the entire change window expands to cover it.

Files **without** a timestamp are treated as never-synced and receive a full re-verification
pass regardless of what else changed in the project.

After every sync, the timestamp is updated on every file that was touched.

---

## What gets updated

The skill handles all standard Claude Code memory file locations:

| File / pattern | Description |
|---|---|
| `CLAUDE.md` | Root and subdirectory project context files |
| `.claude/rules/*.md` | Scoped rules files with optional frontmatter |
| `.claude/*.md` | Other top-level Claude context files |
| Referenced files | Any `.md` paths mentioned in the above files |

### Scope-aware rules files

Rules files can declare their scope using YAML frontmatter:

```markdown
---
applies_to: ["src/frontend/**", "*.tsx"]
context: frontend
---

# Frontend Rules
...
```

The skill respects this - if only backend files changed, a frontend-scoped rules file is left
alone. Files without frontmatter are treated as global and always included.

---

## Safety model

The skill distinguishes between changes that are safe to apply automatically and those that need
your review.

**Applied without confirmation:**
- Adding a new env var to the documented list
- Adding a new script to the commands section
- Documenting a new top-level directory
- Any clearly additive, unambiguous change

**Presented as a consolidated list for your review:**
- Removing an entire architecture section
- Changing the described tech stack
- Any change affecting more than 30% of a file's lines
- Any removal where the content might still be valid in a context the detected changes don't cover

When the skill asks, you can confirm all, reject all, or selectively approve individual items.
Rejected items are noted in the final report under "Items flagged for your review" and left
unchanged.

---

## Sync report format

At the end of every sync, the skill produces a report like:

```
## Claude Memory Sync Report
**Sync baseline:** 2026-01-15T09:00:00Z
**Files scanned:** 12 project files changed since baseline

### Changes made
- CLAUDE.md: Added JWT_ROTATION_ENABLED env var; updated auth section
- .claude/rules/security.md: Removed reference to deprecated /api/v1/login
- .claude/rules/database.md: CREATED - new file for Prisma migration workflow

### Removed content
- CLAUDE.md § "Legacy REST endpoints" - all endpoints removed in commit a3f9d2c

### Items flagged for your review
- .claude/rules/api.md § "Rate limiting" - partially removed feature, left unchanged

### No changes needed
- .claude/rules/testing.md - confirmed accurate
```

If everything is already up to date: `"All memory files are consistent with the current project state."`

---

## Requirements

- Claude Code (any recent version)
- Python 3 in `$PATH` (used for portable timestamp operations)
- Git (optional but recommended - enables richer change detection)

---

## Repository structure

```
claude-memory-sync/
├── .claude-plugin/
│   └── marketplace.json              # Marketplace catalog
├── plugins/
│   └── claude-memory-sync/
│       ├── .claude-plugin/
│       │   └── plugin.json           # Plugin manifest
│       └── skills/
│           └── claude-memory-sync/
│               ├── SKILL.md          # Main skill instructions
│               └── references/
│                   └── memory-file-patterns.md  # Examples and patterns
└── README.md                         # This file
```

---

## Contributing

Issues and pull requests are welcome. When modifying the skill itself (`SKILL.md`), please test
against a real project with a mix of timestamped and timestamp-less memory files, and with both
git and non-git project layouts.

---

## License

MIT
