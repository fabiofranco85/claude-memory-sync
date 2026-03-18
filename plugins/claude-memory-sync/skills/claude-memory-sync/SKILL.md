---
name: claude-memory-sync
description: >
  Keeps Claude's project memory files (CLAUDE.md, .claude/rules/*.md, and related files) synchronized
  with the actual state of the project. Use this skill whenever the user asks to "sync Claude memory",
  "update CLAUDE.md", "refresh Claude's project knowledge", "check if Claude files are up to date",
  or says something like "the project has changed and Claude doesn't know about it". Also trigger when
  the user mentions running a memory audit, syncing .claude rules, or reviewing stale Claude context.
  The skill detects what changed in the project since the last memory update, verifies every affected
  memory file, removes stale content, adds new information, and performs a final consistency review.
---

# Claude Memory Sync

This skill keeps Claude's project memory files accurately synchronized with the real state of the
codebase. It scans for changes since the last memory update, evaluates every relevant memory file,
surgically updates content (additions, modifications, and removals), and ends with a consistency
review. Creating new dedicated context files when warranted is also part of the process.

The ultimate goal: Claude's memory files must reflect the project as it IS, not as it WAS.

---

## Phase 0: Locate and Parse Memory Files

Start by finding the project root (look for `.git`, `CLAUDE.md`, or `.claude/` in the current
directory or ancestors), then change into it so all subsequent relative paths are consistent.

```bash
# Find root and cd into it - all subsequent commands assume cwd is project root
ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
cd "$ROOT"

# List candidate memory files (deduplicated - .claude/*.md would also match rules/*.md)
find . -maxdepth 6 \( \
  -name "CLAUDE.md" \
  -o -path "./.claude/rules/*.md" \
  -o \( -path "./.claude/*.md" ! -path "./.claude/rules/*" \) \
\) | sort
```

Read each discovered file and extract the **last-sync timestamp** from the trailing metadata block:

```
<!-- claude-memory-sync: 2025-11-04T14:32:00Z -->
```

If the timestamp is missing from a file, treat its last-sync date as **unknown** and mark it for
a **full re-verification pass** (every section checked against current project state, independent
of git diff results).

Collect the **oldest** last-sync timestamp across all memory files that *have* one. This becomes
`SYNC_BASELINE`. All change detection is relative to this date.

If **all** memory files lack timestamps, set `SYNC_BASELINE` to the first git commit date
(`git log --reverse --format="%aI" | head -1`), or to 30 days ago as a fallback for non-git
projects:

```bash
# 30-days-ago fallback (portable: works on Linux and macOS)
python3 -c "from datetime import datetime,timezone,timedelta; print((datetime.now(timezone.utc)-timedelta(days=30)).strftime('%Y-%m-%dT%H:%M:%SZ'))"
```

If **some** files have timestamps and others don't, the timestamped files drive `SYNC_BASELINE`
for the git diff window, but timestamp-less files are always fully re-verified regardless of
what that window contains. The two tracks are independent.

> Why the oldest? If any file is behind, it might be missing changes. Using the minimum ensures no
> change slips through the git diff window for timestamped files.

---

## Phase 1: Detect Changes Since SYNC_BASELINE

Run two independent change-detection strategies and merge the results.

### 1a. Git log (preferred when available)

```bash
git log --name-status --since="<SYNC_BASELINE>" --format="---COMMIT--- %H %aI %s" -- .
```

Parse this to produce a list of `(filepath, change_type)` pairs where `change_type` is one of
`A` (added), `M` (modified), `D` (deleted), or `R`-prefixed (renamed, e.g. `R100`). Rename
entries have two tab-separated paths; all others have one. Treat any entry starting with `R` as
a rename regardless of the similarity score suffix.

> Edge case: `--since` is inclusive, so commits at exactly `SYNC_BASELINE` may appear. Filter
> out any commit whose author date exactly equals `SYNC_BASELINE` unless its timestamp is strictly
> newer (i.e., `commit_time > SYNC_BASELINE`). When in doubt, including a false positive is safer
> than missing a real change.

If git is unavailable, skip to 1b only.

### 1b. Filesystem mtime scan

```bash
# Create a reference file at SYNC_BASELINE timestamp (portable: Linux + macOS)
python3 -c "
import os, datetime
open('/tmp/sync_ref', 'w').close()
t = datetime.datetime.fromisoformat('<SYNC_BASELINE>'.replace('Z', '+00:00')).timestamp()
os.utime('/tmp/sync_ref', (t, t))
"

find . \
  -not -path "./.git/*" \
  -not -path "./node_modules/*" \
  -type f \
  -newer /tmp/sync_ref | sort
```

This scan intentionally includes memory files. Partition results into two lists:

- **Memory files** (`.claude/` files, or files named `CLAUDE.md` at any path): handle
  separately. If one of these appears here but already had a valid timestamp in Phase 0, it was
  manually edited after its last sync - mark it for full re-verification in Phase 3.
- **Project files** (everything else): feed into Phase 2 for change analysis.

Merge the git and mtime results into two final lists, applying the same partition rule to both
sources:

- **Memory files** (named `CLAUDE.md` or under `.claude/`): route to the memory-file track.
- **Project files** (everything else): merge and deduplicate into the project change list.

Deleted files come from git log only (mtime cannot detect them).

### 1c. Summarize changes

Group changed files by category before proceeding:

| Category | Examples |
|---|---|
| Source code | `src/`, `lib/`, `app/`, `*.ts`, `*.py`, `*.java` |
| Config / infra | `*.json`, `*.yaml`, `Dockerfile`, `terraform/` |
| Dependencies | `package.json`, `pyproject.toml`, `pom.xml`, `go.mod` |
| Documentation | `README.md`, `docs/`, `*.md` (non-Claude) |
| Tests | `test/`, `spec/`, `__tests__/`, `*.test.*` |
| CI/CD | `.github/`, `.gitlab-ci.yml`, `Jenkinsfile` |
| Claude memory | `.claude/`, `CLAUDE.md` (track but handle separately) |

If **zero non-memory files changed** AND **no memory files are on the full re-verification
track**, report that no sync is needed and stop (unless the user asked for a forced full review).

If there are memory files on the full re-verification track (no timestamp, or manually edited),
continue to Phase 3 regardless - those files need checking even if the project itself didn't change.

---

## Phase 2: Read and Understand the Changes

For each changed file, do enough reading to understand the nature of the change. Use judgment:

- **Trivial changes** (whitespace, comments, version bumps in lock files): note them briefly,
  unlikely to require memory updates.
- **Structural changes** (new modules, renamed functions, changed APIs, new CLI commands,
  new env vars, removed features): read the diff carefully. These WILL require memory updates.
- **Deleted files**: check if anything in memory references the deleted path or concept.
- **Renamed files** (`R` prefix in git log, e.g. `R100` with a similarity score): the entry
  has two tab-separated paths - old path and new path. Treat as a deletion of the old path plus
  an addition of the new path. Any memory reference to the old path is stale and must be updated
  to the new path. Check both the old name and the new name against all memory files.

```bash
# Read a file's diff since the sync baseline (timestamp-based)
git log --since="<SYNC_BASELINE>" --follow -p -- <filepath>

# For a tighter diff (shows only what changed, not full history):
BEFORE=$(git rev-list -n1 --before="<SYNC_BASELINE>" HEAD)
if [ -n "$BEFORE" ]; then
  git diff "$BEFORE" HEAD -- <filepath>
else
  # No commit predates the baseline - show the full file as added
  git show HEAD:<filepath> 2>/dev/null || cat <filepath>
fi

# For untracked / non-git: just read the file
cat <filepath>
```

Produce an **internal change summary** (not shown to user yet) with these fields per changed file:

```
filepath: src/api/auth.ts
change_type: M
impact: High - JWT secret rotation added, new env var JWT_ROTATION_ENABLED
memory_relevant: yes
```

Flag all `memory_relevant: yes` entries. The rest can be ignored for memory purposes.

---

## Phase 3: Map Changes to Memory Files

For each memory-relevant change, determine which memory file(s) might be affected.

Read the full content of each candidate memory file now (if not already in context).

**Files on the full re-verification track** - those with no timestamp (from Phase 0) or those
flagged by Phase 1b as manually edited after their last sync - skip the change-mapping step
below and go directly to Phase 4 with scope set to "entire file". Every section is checked,
not just sections related to detected changes. Treat these files as if every line might be stale.

Also scan for explicit cross-references from memory files to other files - both `.claude/`
paths and general markdown links or path mentions:

```bash
# .claude/ internal references
grep -rEoh '\.claude/[a-zA-Z0-9/_-]+\.md' .claude/ CLAUDE.md 2>/dev/null | sort -u

# General file references in markdown links or bare paths (e.g. docs/ARCHITECTURE.md)
# Requires a slash so bare 'README.md' is skipped; excludes dependency dirs and URLs
grep -rEoh '[a-zA-Z0-9._/-]+\.md' .claude/ CLAUDE.md 2>/dev/null \
  | grep '/' \
  | grep -v '^\.' \
  | grep -Ev '(node_modules|vendor|dist|\.min\.)' \
  | grep -Ev '^(http|https)' \
  | sort -u
```

For each referenced path that exists in the project, read it if it contains information that
Claude's memory files describe or depend on. Do not update these external files themselves -
only use them as ground truth when verifying memory file accuracy.

Build a mapping:

```
change: src/api/auth.ts (JWT rotation)
  → CLAUDE.md § "Authentication"     (likely stale)
  → .claude/rules/security.md        (may need new rule)
```

### Frontmatter / context filtering

Some memory files contain YAML frontmatter that restricts their scope:

```yaml
---
applies_to: ["src/frontend/**", "*.tsx"]
context: frontend
---
```

Only include such files in the update plan if the changed files overlap with their declared scope.
If unsure, include the file but note the uncertainty.

---

## Phase 4: Verify Each Memory File

For every memory file in scope, go section by section and evaluate:

### Accuracy check (per section)

- Does the described behavior match what the code currently does?
- Are referenced file paths still valid?
- Are described commands still correct (check `package.json` scripts, `Makefile`, etc.)?
- Are env var names still accurate?
- Are architectural descriptions still current?

### Staleness check

- Are there references to deleted files or removed features?
- Are version numbers outdated?
- Are workflow steps that no longer apply still documented?

### Gap check

- Do the memory-relevant changes introduce concepts not yet mentioned anywhere?
- Are new required env vars documented?
- Are new commands or scripts mentioned?

Produce a **proposed change list** per file with the following types:

| Type | Description |
|---|---|
| `ADD` | New content to add |
| `UPDATE` | Existing content that needs correction |
| `REMOVE` | Content that is now stale and should be deleted |
| `MOVE` | Content that belongs in a different file |

---

## Phase 5: Safety Review Before Writing

After completing the proposed change list for all files, scan it for any entry that meets
these criteria:

- Any `REMOVE` or `UPDATE` operation targets a **major concept** (e.g., removing an entire
  architecture section, changing the described tech stack, deleting a rules file section).
- Any change would affect more than **30% of a file's lines** (use line count as the measure).
- A proposed removal targets content that could still be valid in a context not covered by the
  detected changes (e.g., a feature only partially removed).

For each flagged entry, before presenting it to the user:

1. Describe exactly what will be removed and why you believe it is stale.
2. Check one more time: re-read the relevant source file or git log to confirm.
3. If still uncertain after the second check, mark it as "needs user decision" rather than
   proceeding or silently skipping.

**Present all flagged changes as a single consolidated list** - do not ask about each one
separately. Wait for one response that covers all of them. The user may confirm all, reject
all, or selectively approve. Apply only what was approved; flag the rest in the final report.

For small, clearly correct additions (new env var documented, new script added): proceed
without confirmation - do not include them in the consolidated review list.

---

## Phase 6: Apply Updates

Apply changes file by file. For each file:

1. Read the current content in full.
2. Make the minimum changes needed (surgical edits, not rewrites).
3. Preserve the file's existing style, heading hierarchy, and tone.
4. Update the last-sync timestamp at the end of the file.

### Timestamp block format

Every memory file must end with this exact block (create it if missing):

```markdown

---
<!-- claude-memory-sync: <ISO8601_UTC_TIMESTAMP> -->
```

Get the current UTC timestamp to fill in:

```bash
python3 -c "from datetime import datetime, timezone; print(datetime.now(timezone.utc).strftime('%Y-%m-%dT%H:%M:%SZ'))"
```

The blank line before `---` is required. If a timestamp block exists anywhere other than the
very end of the file, move it to the end during this pass.

### Creating new context files

If the changes introduce a concept or subsystem large enough to warrant its own dedicated file
(heuristic: more than ~40 lines of new relevant content, or a clearly distinct domain like
`database`, `auth`, `deployment`, `i18n`), create a new file at:

```
.claude/rules/<domain>.md
```

Use frontmatter to declare scope if applicable. Add a cross-reference to it from `CLAUDE.md`.

---

## Phase 7: Final Consistency Review

After all updates are written, do one final pass across ALL memory files as a set:

### Consistency checks

- **No contradictions**: do two files say conflicting things about the same concept?
- **No orphaned references**: does any file reference another `.claude/*.md` file that no longer
  exists?

  ```bash
  grep -rEoh '\.claude/[a-zA-Z0-9/_-]+\.md' .claude/ CLAUDE.md 2>/dev/null | sort -u | while read ref; do
    [ -f "./$ref" ] || echo "ORPHANED: $ref"
  done
  ```
- **No duplicated content**: is the same detail explained in two places when one would do?
- **Correct scope**: is content in the right file given its frontmatter/context declarations?
- **Timestamp alignment**: every file that was written or created in Phase 6 must carry the
  timestamp from this run. Files that were reviewed but not changed keep their existing timestamp
  - do not flag those.

### Relevance check

Quickly scan each file for sections that seem unrelated to the project's actual domain. If a
section looks like a leftover from a template or boilerplate, flag it for the user.

---

## Phase 8: Report to User

Produce a concise sync report:

```
## Claude Memory Sync Report
**Sync baseline:** <SYNC_BASELINE>
**Files scanned:** <N> project files changed since baseline
**Memory files updated:** <list>

### Changes made
- CLAUDE.md: Added JWT_ROTATION_ENABLED env var; updated auth section
- .claude/rules/security.md: Removed reference to deprecated /api/v1/login
- .claude/rules/database.md: CREATED - new file for Prisma migration workflow

### Removed content
- <description of what was removed and why>

### Items flagged for your review
- <anything uncertain that was left as-is>

### No changes needed
- <files reviewed but confirmed accurate>
```

If nothing changed, say so clearly: "All memory files are consistent with the current project state."

---

## Important Principles

**Remove aggressively, add carefully.** Stale information misleads Claude more than missing
information. If you're 90%+ confident something is stale, remove it. If you're only 70% confident,
flag it instead.

**Don't rewrite, patch.** Preserve existing voice and structure. Only touch what the changes
actually affect.

**Follow the evidence.** Every update should be traceable to a specific file change or deletion.
Don't add information based on inference unless it's clearly implied by the diff.

**Scope gates are real.** If a memory file says `applies_to: ["src/backend/**"]` and only frontend
files changed, leave it alone.

**The timestamp is a contract.** Update it on every file you touch. A missing or stale timestamp
means the next sync will re-verify the whole file from scratch.

---

## Reference: Common Memory File Patterns

See `references/memory-file-patterns.md` for examples of well-structured CLAUDE.md sections,
rules files with frontmatter, and timestamp placement.
