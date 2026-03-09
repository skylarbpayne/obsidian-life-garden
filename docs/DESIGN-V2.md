# Life Garden Plugin Design V2: Vault Conventions + Palmer Integration

> **V2 Revision 2** – Addresses PR #2 review comments: simplified folder structure, Obsidian CLI integration, periodic notes, minimal templates, recurrence support, and single-direction dependency authoring.

## 1. Goals and Non-Goals

### Goals
- Provide continuous life-system insights from an Obsidian vault that is messy, incomplete, and evolving.
- Make metadata authoring and repair first-class, not optional.
- Support Palmer-driven workflows via **standard Obsidian CLI/URI schemes and direct file operations** (not custom plugin hooks).
- Keep analysis reliable with sane defaults and minimal required configuration.
- Preserve inspectability: users and Palmer can see why a note is flagged and what data is missing.

### Non-Goals (Phase 1)
- No cloud sync or remote processing.
- No autonomous destructive edits (delete/move notes without explicit command).
- No requirement for perfect metadata before analysis can run.
- No custom plugin API for note creation – use Obsidian-native tools.

## 2. End-to-End Operating Loop (User → Palmer → Vault → Plugin → Palmer)

1. **User says:** "set up date with Jacqueline."
2. **Palmer creates note** using Obsidian CLI (`obsidian://new`) or direct file write:
   ```bash
   # Option A: Obsidian URI scheme
   open "obsidian://new?vault=skyvault&file=tasks/2026-03-02-date-jacqueline&content=..."
   
   # Option B: Direct file write (Palmer has vault path)
   echo "---\ntype: [task]\nstatus: ready\n..." > ~/skyvault/tasks/2026-03-02-date-jacqueline.md
   ```
3. **Plugin re-indexes** on configurable schedule and computes insights.
4. **Plugin exposes insights** via JSON export file that Palmer can read:
   ```
   ~/skyvault/.obsidian/plugins/life-garden/insights.json
   ```
5. **Palmer transitions status** via direct frontmatter edits using standard YAML tools.
6. **Palmer runs metadata repair** on configurable cadence (daily/weekly/on-demand).
7. Loop repeats with progressively cleaner metadata and higher-quality insights.

### 2.1 Why Obsidian CLI + File Ops (Not Custom API)

**Reviewer feedback:** "most of these should just use the obsidian cli directly"

Benefits:
- **No plugin dependency for authoring** – Palmer can create notes even if plugin isn't running.
- **Standard tooling** – Leverages Obsidian's built-in URI scheme and filesystem.
- **Simpler plugin scope** – Plugin focuses on analysis and insights, not CRUD operations.
- **Portable** – Same workflows work with other tools (VS Code, shell scripts, etc.).

Palmer learns the conventions documented here and uses standard file operations. The plugin observes the vault and computes insights – it doesn't need to be the gatekeeper for note creation.

## 3. Vault Structure Conventions (Phase 0.5 Foundation)

### 3.1 Canonical Folder Hierarchy

**Note:** These are user's notes about their life, not Palmer's notes. No `Palmer/` prefix.

```text
~/skyvault/
  goals/        # long-horizon outcomes
  projects/     # multi-step initiatives with bounded scope
  tasks/        # actionable units of work
  people/       # relationship records + interactions
  habits/       # recurring behaviors/routines
  periodic/     # daily, weekly, monthly, annual notes
  reference/    # non-actionable source material
  archive/      # inactive/completed historical items
  templates/    # note templates used by Templater
```

### 3.2 Folder Purpose and Scope Rules
- `goals/`: Strategic outcomes (weeks to years). Not granular task checklists.
- `projects/`: Finite efforts with multiple tasks and explicit status.
- `tasks/`: Single actionable work items; can link to project/goal/person.
- `people/`: One primary note per person plus optional dated interaction notes.
- `habits/`: Recurring behaviors/routines with cadence tracking.
- `periodic/`: **Replaces `journal/`** – supports multiple cadences:
  - `periodic/daily/YYYY-MM-DD.md`
  - `periodic/weekly/YYYY-Www.md` (e.g., `2026-W10.md`)
  - `periodic/monthly/YYYY-MM.md`
  - `periodic/annual/YYYY.md`
- `reference/`: Background docs, not directly actionable.
- `archive/`: Excluded from default scoring but retained for context links.

### 3.3 File Naming Conventions
- Task: `tasks/YYYY-MM-DD-<slug>.md` (e.g., `tasks/2026-03-02-date-jacqueline.md`)
- Project: `projects/<slug>.md`
- Goal: `goals/<area>-<slug>.md`
- Person: `people/<name>.md`
- Habit: `habits/<slug>.md`
- Periodic: See cadence subfolders above
- Interaction note (optional): `people/<name>/YYYY-MM-DD-<slug>.md`

### 3.4 Cross-Folder Linking Conventions
- Tasks should link upward to project/goal where relevant.
- Person-related tasks include `people: [[Name]]` frontmatter.
- `archive/` notes can be linked for historical traceability but excluded from urgency ranking.
- Prefer full path disambiguation when titles collide.

## 4. Frontmatter Schema (Minimal Required + Flexible Optional)

### 4.1 Required Core Fields (Truly Minimal)

```yaml
---
type: [task]
status: ready
---
```

That's it. Two fields.

**Timestamps** (`created`, `updated`) are inferred from filesystem metadata when missing. Plugin can auto-populate them on first index.

**Rules:**
- `type` is always an array (multi-valued taxonomy).
- `status` is single-valued lifecycle state.
- If missing, plugin marks note `needsMetadataRepair` for Palmer to address.

### 4.2 Optional Enhancement Fields

```yaml
---
priority: P1
domain: [relationship, social]
people: [[Jacqueline]]
project: [[projects/relationship-investment]]
goals: [[goals/relationship-depth]]
blocks: [[tasks/find-restaurant]]     # this note blocks target
due: 2026-03-06
recurs: weekly                        # recurrence cadence
energy: low | medium | high
archive: false
---
```

### 4.3 Status Lifecycle Enum
- `idea` – captured but not actionable yet
- `ready` – actionable, waiting to start
- `in-progress` – actively working
- `blocked` – waiting on dependency
- `done` – completed
- `dropped` – abandoned/no longer relevant

### 4.4 Type Examples
- Date planning task: `type: [task, relationship]`
- Newsletter initiative: `type: [task, growth]`
- Person interaction note: `type: [relationship, log]`
- Recurring gym routine: `type: [habit, health]`

### 4.5 Recurrence Model

**Reviewer feedback:** "do we need to somehow represent 'recurring'?"

Yes. The `recurs` field indicates a task/habit regenerates after completion:

```yaml
recurs: daily | weekly | monthly | quarterly | annual | <cron>
recurs_until: 2026-12-31   # optional end date
last_recurred: 2026-03-02  # auto-updated by repair workflow
```

**Behavior:**
- When a recurring task is marked `done`, Palmer creates a new instance with fresh `created` date.
- Plugin flags recurring items nearing cadence deadline for attention.
- `habits/` folder items are implicitly recurring unless `recurs: false`.

## 5. Flexible Taxonomy and Domain Model

### 5.1 Revised Node Model

```ts
interface GardenNode {
  id: string;              // canonical: file.path
  path: string;
  title: string;
  types: string[];         // multi-valued (formerly single type)
  domains: string[];       // multi-valued (formerly single domain)
  people: string[];        // wikilink targets
  status: string;
  priority?: 'P0' | 'P1' | 'P2' | 'P3' | string;
  created?: string;        // from frontmatter or file stat
  updated?: string;        // from frontmatter or file stat
  due?: string;
  recurs?: string;
  needsMetadataRepair: boolean;
  repairReasons: string[];
  tags: string[];
}
```

### 5.2 Principles
- Treat `type` and `domain` as sets, not enums.
- Preserve unknown user values; normalize only where needed for filtering.
- Scoring uses tag groups (`relationship`, `health`, `work`) but never rejects unknown tags.

## 6. Note Templates (Minimal by Default)

**Reviewer feedback:** "this is a lot of info to fill out for tasks. I imagine 80-90% of tasks I will not fill out this data for."

Solution: Templates are **minimal**. Palmer auto-fills the rest based on context.

Templates live in `templates/` and are used by Templater. Palmer uses direct file writes.

### 6.1 `templates/Task.md` (Minimal)

```markdown
---
type: [task]
status: ready
---

# <% tp.file.title %>

## What

## Notes
```

That's it. 4 lines of frontmatter + 2 section headers.

Palmer fills in: `priority`, `domain`, `people`, `project`, `goals`, `due`, `created`, `updated` – **only when it has high confidence** from conversation context.

### 6.2 `templates/Project.md`

```markdown
---
type: [project]
status: ready
---

# <% tp.file.title %>

## Why

## Success Criteria

## Tasks
```

### 6.3 `templates/Goal.md`

```markdown
---
type: [goal]
status: in-progress
horizon: quarter
---

# <% tp.file.title %>

## Outcome

## Leading Indicators
```

### 6.4 `templates/Person.md`

```markdown
---
type: [person]
domain: [relationship]
---

# <% tp.file.title %>

## Context

## Open Loops
```

### 6.5 `templates/Habit.md`

```markdown
---
type: [habit]
status: in-progress
recurs: weekly
---

# <% tp.file.title %>

## Trigger

## Routine
```

### 6.6 `templates/Periodic-Daily.md`

```markdown
---
type: [periodic, daily]
---

# <% tp.date.now("YYYY-MM-DD dddd") %>

## Morning

## Evening

## Notes
```

## 7. Palmer Integration (via Obsidian CLI + File Ops)

**Reviewer feedback:** "i don't like the idea of having a bunch of custom hooks here; would rather use the general obsidian cli"

### 7.1 Palmer's Toolkit

Palmer uses these standard mechanisms instead of custom plugin API:

| Operation | Method |
|-----------|--------|
| Create note | `obsidian://new?vault=...&file=...&content=...` or direct file write |
| Open note | `obsidian://open?vault=...&file=...` |
| Open daily note | `obsidian://daily?vault=...` |
| Search vault | `obsidian://search?vault=...&query=...` or `rg` / file search |
| Update metadata | Direct frontmatter edit (YAML-aware) |
| Read insights | Parse `~/skyvault/.obsidian/plugins/life-garden/insights.json` |
| Transition status | Frontmatter edit: `status: ready` → `status: done` |

### 7.2 Obsidian URI Examples

```bash
# Create a new task
open "obsidian://new?vault=skyvault&file=tasks/2026-03-02-date-jacqueline&content=$(cat <<'EOF'
---
type: [task, relationship]
status: ready
people: [[Jacqueline]]
---

# Date with Jacqueline

## What
Plan and schedule a date.
EOF
)"

# Open today's daily note
open "obsidian://daily?vault=skyvault"

# Search for all blocked tasks
open "obsidian://search?vault=skyvault&query=status: blocked"
```

### 7.3 Direct File Operations (for Palmer)

When CLI isn't available or for batch operations:

```bash
# Create note with proper YAML
cat > ~/skyvault/tasks/2026-03-02-date-jacqueline.md << 'EOF'
---
type: [task, relationship]
status: ready
people: [[Jacqueline]]
created: 2026-03-02
---

# Date with Jacqueline

## What
Plan and schedule a date.
EOF

# Update status (using yq or sed)
yq -i '.status = "done"' ~/skyvault/tasks/2026-03-02-date-jacqueline.md
```

### 7.4 Plugin Output: Insights JSON

The plugin writes a structured insights file that Palmer reads:

```json
{
  "generatedAt": "2026-03-02T18:30:00Z",
  "health": "warning",
  "stats": {
    "totalNodes": 142,
    "needsRepair": 12,
    "stale": 8,
    "blocked": 3
  },
  "insights": [
    {
      "type": "stale",
      "path": "tasks/2026-02-15-tax-prep.md",
      "title": "Tax Prep",
      "daysSinceUpdate": 15,
      "reason": "No activity for 15 days, status still 'ready'"
    },
    {
      "type": "needsRepair",
      "path": "tasks/2026-03-01-something.md",
      "repairReasons": ["missing status", "type is scalar not array"]
    }
  ],
  "metadataWarnings": [
    {"path": "...", "field": "type", "issue": "scalar, expected array"}
  ]
}
```

## 8. Dependency Authoring (Single-Direction is Enough)

**Reviewer feedback:** "is it simple to maintain the bidirectional block linking? (e.g. seems like you have to edit 2 notes to add an edge)"

**Answer:** No, single direction is sufficient. The plugin infers the reverse edge automatically.

### 8.1 One-Sided Declaration

Only one side needs to declare the dependency:

```yaml
# In tasks/find-restaurant.md
blocks: [[tasks/date-jacqueline]]  # This note blocks that note
```

The plugin automatically computes:
- `find-restaurant` → blocks → `date-jacqueline`
- `date-jacqueline` ← blocked-by ← `find-restaurant` (inferred)

No need to edit the target note.

### 8.2 `blocks` vs `blocked_by`

Both are supported, but `blocks` is the canonical direction:

| Field | Meaning | Edge Direction |
|-------|---------|----------------|
| `blocks: [[B]]` | This note gates B | A → B |
| `blocked_by: [[A]]` | This note is gated by A | A → B (same result) |

Either works. Plugin normalizes to consistent edges.

### 8.3 Inline Syntax (Optional)

For quick authoring in note body:

```markdown
This task @blocks [[tasks/schedule-venue]] and must complete first.
```

Parser extracts `@blocks` markers and syncs to frontmatter if desired.

### 8.4 Command Palette (Plugin Feature)

Plugin provides:
- `Life Garden: Add Dependency` – fuzzy picker to add `blocks:` entry
- `Life Garden: View Dependencies` – see what this note blocks/is-blocked-by

## 9. Metadata Repair Strategy

### 9.1 Detection

Plugin flags notes when:
- Missing required fields (`type`, `status`)
- Scalar where array expected (`type: task` instead of `[task]`)
- Unknown status outside configured lifecycle
- Timestamps missing (can auto-fill from filesystem)

### 9.2 Configurable Repair Cadence

**Reviewer feedback:** "this might be fine to start, but should be configurable"

```ts
repairCadence: 'manual' | 'daily' | 'weekly' | 'on-index'  // default: 'weekly'
```

Palmer can also trigger repair on-demand by running a batch script.

### 9.3 Repair Workflow

1. Palmer reads `insights.json` and filters for `needsRepair` items.
2. Palmer examines each note and determines fixes:
   - Context from title, folder, content, links
   - High-confidence: apply directly
   - Medium-confidence: ask user or skip
3. Palmer writes corrected frontmatter using YAML tools.
4. Plugin re-indexes on next cycle.

### 9.4 User Review Surface (Plugin UI)

New dashboard tab: `Metadata`
- Lists notes needing repair with proposed fixes
- Shows confidence score and evidence
- Actions: `Apply`, `Skip`, `Ignore this rule`

## 10. Inclusion, Archive, and Allow/Blocklist

### 10.1 Default Inclusion

- **Include:** `goals`, `projects`, `tasks`, `people`, `habits`, `periodic`
- **Exclude:** `archive`, `reference`, `templates`, hidden folders (`.`)

### 10.2 Settings

```ts
includeFolders: string[]     // override defaults
excludeFolders: string[]     // default: ['archive', 'reference', 'templates']
excludeTags: string[]        // e.g., ['ignore-garden']
```

### 10.3 Archive Link Policy

- Active note linking to archived note: keep edge for context
- Archived nodes excluded from urgency rankings
- Show badge: `depends on archived context`

## 11. Analysis and Update Cadence

**Reviewer feedback:** "we don't need very frequent updates here. Even latency of like 1 hour between a new thing added to it being reflected is fine."

### 11.1 Cadence Settings

```ts
analysisCadence: 'manual' | 'hourly' | 'daily'  // default: 'hourly'
```

### 11.2 Behavior
- **Manual:** Only on explicit command
- **Hourly:** Background refresh, guaranteed within 1 hour
- **Daily:** Light-touch, once per session start + daily timer

### 11.3 Manual Trigger

Command: `Life Garden: Recompute Now`

## 12. Simplified Settings (7 Essential Controls)

```ts
interface LifeGardenSettingsV2 {
  includeFolders: string[];              // default: core folders
  excludeFolders: string[];              // default: ['archive', 'reference', 'templates']
  staleThresholdDays: number;            // default: 14
  analysisCadence: 'manual' | 'hourly' | 'daily';  // default: 'hourly'
  maxInsightCards: number;               // default: 25
  repairCadence: 'manual' | 'daily' | 'weekly' | 'on-index';  // default: 'weekly'
  includeWikiLinksInDependencies: boolean;  // default: false
}
```

Advanced tuning (damping, centrality limits) hidden under `Advanced`.

## 13. Edge Cases and Expected Behavior

1. **Incomplete metadata:** Include note, mark `needsMetadataRepair`, lower confidence.
2. **Mixed scalar/array frontmatter:** Normalize when safe, otherwise propose repair.
3. **Path/title collisions:** Canonical id remains `TFile.path`.
4. **Circular dependencies:** SCC condensation for critical path summaries.
5. **Archived dependencies:** Preserve context edge, exclude from urgency.
6. **Recurring tasks:** When completed, Palmer creates next instance if `recurs` set.
7. **Very large vault:** Skip expensive metrics; always return latest successful snapshot.

## 14. How V2 Addresses Critical Issues

| Issue | V1 Problem | V2 Solution |
|-------|------------|-------------|
| Metadata authoring | No clear path | Minimal templates + Palmer auto-fill + repair workflow |
| Palmer integration | Custom API bloat | Standard Obsidian CLI/URI + file ops + JSON export |
| Lifecycle management | Missing | Concrete status enum + recurrence model |
| Rigid taxonomy | Single enums | Multi-valued `type`/`domain` arrays |
| Dependency authoring | Two-sided edits | Single-direction `blocks:` with inferred reverse |
| Vault structure | Undefined | Explicit folder hierarchy + periodic notes |
| Settings complexity | Over-specified | 7 essential controls, advanced hidden |
| Folder ownership | Under `Palmer/` | Root-level folders (user's notes, not Palmer's) |

## 15. Palmer Workflow Documentation

Palmer should be taught these conventions:

### 15.1 Creating a Task

```bash
# Minimal task – just type and status
cat > ~/skyvault/tasks/$(date +%Y-%m-%d)-<slug>.md << 'EOF'
---
type: [task]
status: ready
---

# Task Title

## What
Description here.
EOF
```

### 15.2 Transitioning Status

```bash
# Using yq (or any YAML tool)
yq -i '.status = "done"' ~/skyvault/tasks/2026-03-02-date-jacqueline.md
yq -i '.updated = "2026-03-02"' ~/skyvault/tasks/2026-03-02-date-jacqueline.md
```

### 15.3 Reading Insights

```bash
# Parse insights JSON
jq '.insights[] | select(.type == "stale")' ~/skyvault/.obsidian/plugins/life-garden/insights.json
```

### 15.4 Batch Repair

```bash
# Find all notes needing repair
jq -r '.metadataWarnings[].path' ~/skyvault/.obsidian/plugins/life-garden/insights.json | while read path; do
  # Analyze and fix each note
  # ...
done
```

## 16. Open Questions (Implementation-Scoped)

1. Should `periodic/daily/` notes be included in dependency analysis or only as context?
2. What confidence threshold should auto-repair use (0.85)?
3. Should transition logs be embedded in note body or stored in plugin state?
4. How should plugin handle conflicts between `obsidian://` writes and direct file writes (timestamp races)?

## 17. Revised Implementation Plan

1. **Phase 0:** Document conventions (this document) + Palmer workflow guide
2. **Phase 1:** Basic indexer with minimal schema support
3. **Phase 2:** Insights JSON export + metadata warnings
4. **Phase 3:** Dashboard UI for viewing insights
5. **Phase 4:** Metadata repair proposals + review UI
6. **Phase 5:** Command palette dependency authoring
7. **Phase 6:** Recurrence tracking + periodic note integration

---

## Appendix A: Obsidian CLI Reference

### URI Scheme Actions

| Action | URI Format |
|--------|------------|
| Open vault | `obsidian://open?vault=<name>` |
| Open file | `obsidian://open?vault=<name>&file=<path>` |
| Open daily | `obsidian://daily?vault=<name>` |
| New file | `obsidian://new?vault=<name>&file=<path>&content=<encoded>` |
| Search | `obsidian://search?vault=<name>&query=<query>` |

### Command Line (macOS)

```bash
# Via open command
open "obsidian://..."

# Via osascript for scripting
osascript -e 'open location "obsidian://..."'
```

## Appendix B: Example Frontmatter Patterns

### Minimal Task
```yaml
type: [task]
status: ready
```

### Full Task
```yaml
type: [task, relationship]
status: in-progress
priority: P1
domain: [relationship, social]
people: [[Jacqueline]]
project: [[projects/relationship-investment]]
blocks: [[tasks/book-venue]]
due: 2026-03-15
created: 2026-03-02
updated: 2026-03-05
```

### Recurring Habit
```yaml
type: [habit, health]
status: in-progress
recurs: daily
domain: [health]
last_recurred: 2026-03-01
```

### Person
```yaml
type: [person]
domain: [relationship]
last_contact: 2026-02-28
```
