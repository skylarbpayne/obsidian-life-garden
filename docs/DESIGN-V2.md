# Life Garden Plugin Design V2: Vault Conventions + Palmer Integration

## 1. Goals and Non-Goals

### Goals
- Provide continuous life-system insights from an Obsidian vault that is messy, incomplete, and evolving.
- Make metadata authoring and repair first-class, not optional.
- Support Palmer-driven workflows: create notes, update metadata, transition status, and consume insights programmatically.
- Keep analysis reliable with sane defaults and minimal required configuration.
- Preserve inspectability: users and Palmer can see why a note is flagged and what data is missing.

### Non-Goals (Phase 1)
- No cloud sync or remote processing.
- No autonomous destructive edits (delete/move notes without explicit command).
- No requirement for perfect metadata before analysis can run.

## 2. End-to-End Operating Loop (User -> Palmer -> Plugin -> Palmer)

1. User says: "set up date with Jacqueline."
2. Palmer calls `createNote` with `type=[task,relationship]`, `people=[[Jacqueline]]`, `domain=[relationship,social]`.
3. Plugin creates note in canonical folder with template-backed frontmatter and content scaffolding.
4. Plugin re-indexes on schedule/event and computes insights.
5. Plugin exposes structured insights via `getInsights` (stale relationship, blockers, critical path, missing metadata).
6. Palmer calls `transitionStatus`/`updateMetadata` as work progresses.
7. Weekly, Palmer runs batch metadata repair for under-specified notes.
8. Plugin records repair proposals and applies approved changes.
9. Loop repeats with progressively cleaner metadata and higher-quality insights.

## 3. Vault Structure Conventions (Phase 0.5 Foundation)

### 3.1 Canonical Folder Hierarchy

```text
~/skyvault/Palmer/
  goals/        # long-horizon outcomes
  projects/     # multi-step initiatives with bounded scope
  tasks/        # actionable units of work
  people/       # relationship records + interactions
  habits/       # recurring behaviors/routines
  journal/      # daily logs and context
  reference/    # non-actionable source material
  archive/      # inactive/completed historical items
  templates/    # note templates used by plugin + Templater
```

### 3.2 Folder Purpose and Scope Rules
- `goals/`: strategic outcomes (weeks to years). Not granular task checklists.
- `projects/`: finite efforts with multiple tasks and explicit status.
- `tasks/`: single actionable work items; can link to project/goal/person.
- `people/`: one primary note per person plus optional dated interaction notes.
- `habits/`: recurring cadence notes (daily/weekly plans, streaks, checkpoints).
- `journal/`: chronological logs; source for inference and metadata repair.
- `reference/`: background docs, not directly actionable.
- `archive/`: excluded from default scoring but retained for context links.

### 3.3 File Naming Conventions
- Task: `tasks/YYYY-MM-DD-<slug>.md` (example: `tasks/2026-03-02-schedule-date-with-jacqueline.md`)
- Project: `projects/<slug>.md`
- Goal: `goals/<area>-<slug>.md`
- Person: `people/<name>.md`
- Habit: `habits/<cadence>-<slug>.md` (example: `habits/weekly-relationship-outreach.md`)
- Journal: existing `journal/YYYY-MM-DD.md`
- Interaction note (optional): `people/<name>/YYYY-MM-DD-<slug>.md`

### 3.4 Cross-Folder Linking Conventions
- Tasks should link upward to project/goal where relevant.
- Person-related tasks include `people: [[Name]]` and a body link to `[[people/Name]]`.
- `archive/` notes can be linked for historical traceability but are excluded from urgency ranking by default.
- Prefer full path disambiguation when titles collide.

## 4. Frontmatter Schema (Minimal Required + Flexible Optional)

### 4.1 Required Core Fields

```yaml
---
type: [task]
status: ready
created: 2026-03-02
updated: 2026-03-02
---
```

Rules:
- `type` is always an array (multi-valued taxonomy).
- `status` is single-valued lifecycle state.
- `created`/`updated` are ISO dates (`YYYY-MM-DD`).
- If missing, plugin marks note `needsMetadataRepair` and can auto-propose fixes.

### 4.2 Optional Enhancement Fields

```yaml
---
priority: P1
domain: [relationship, social]
people: [[Jacqueline]]
project: [[projects/relationship-investment]]
goals: [[goals/relationship-depth]]
blocks: [[tasks/find-restaurant]]
blocked_by: [[tasks/confirm-availability]]
due: 2026-03-06
cadence: weekly
energy: medium
archive: false
---
```

### 4.3 Status Lifecycle Enum
- `idea`
- `ready`
- `in-progress`
- `blocked`
- `done`
- `dropped`

### 4.4 Type Examples
- Date planning task: `type: [task, relationship]`
- Newsletter initiative: `type: [task, habit, growth]`
- Person interaction note: `type: [relationship, person-log]`

## 5. Flexible Taxonomy and Domain Model

### 5.1 Revised Node Model

```ts
interface GardenNode {
  id: string; // canonical: file.path
  path: string;
  title: string;
  types: string[];      // formerly single type
  domains: string[];    // formerly single domain
  people: string[];     // wikilink targets or normalized names
  status: string;
  priority?: 'P0' | 'P1' | 'P2' | 'P3' | string;
  created?: string;
  updated?: string;
  needsMetadataRepair: boolean;
  repairReasons: string[];
  tags: string[];
}
```

### 5.2 Principles
- Treat `type` and `domain` as sets, not enums.
- Preserve unknown user values; normalize only where needed for filtering.
- Scoring uses tag groups (`relationship`, `health`, `work`) but never rejects unknown tags.

## 6. Note Templates (Templater-Compatible)

Templates live in `templates/` and are used by both humans and API note creation.

### 6.1 `templates/Task.md`

```markdown
---
type: [task]
status: ready
priority: P2
domain: []
people: []
project:
goals: []
blocks: []
blocked_by: []
created: <% tp.date.now("YYYY-MM-DD") %>
updated: <% tp.date.now("YYYY-MM-DD") %>
---

# <% tp.file.title %>

## Outcome
What must be true for this task to be complete?

## Next Action
Smallest concrete step.

## Notes
Context, links, decisions.
```

### 6.2 `templates/Project.md`

```markdown
---
type: [project]
status: ready
priority: P2
domain: []
goals: []
created: <% tp.date.now("YYYY-MM-DD") %>
updated: <% tp.date.now("YYYY-MM-DD") %>
---

# <% tp.file.title %>

## Why

## Success Criteria

## Active Tasks
- [[tasks/...]]
```

### 6.3 `templates/Goal.md`

```markdown
---
type: [goal]
status: in-progress
domain: []
horizon: quarter
created: <% tp.date.now("YYYY-MM-DD") %>
updated: <% tp.date.now("YYYY-MM-DD") %>
---

# <% tp.file.title %>

## Outcome

## Leading Indicators

## Linked Projects
- [[projects/...]]
```

### 6.4 `templates/Person.md`

```markdown
---
type: [person, relationship]
status: active
domain: [relationship]
last_contact:
created: <% tp.date.now("YYYY-MM-DD") %>
updated: <% tp.date.now("YYYY-MM-DD") %>
---

# <% tp.file.title %>

## Relationship Health

## Open Loops

## Next Touchpoint
```

### 6.5 `templates/Habit.md`

```markdown
---
type: [habit]
status: in-progress
domain: []
cadence: weekly
created: <% tp.date.now("YYYY-MM-DD") %>
updated: <% tp.date.now("YYYY-MM-DD") %>
---

# <% tp.file.title %>

## Trigger

## Routine

## Tracking
```

## 7. Palmer Integration API

API is local-only within plugin runtime, exposed by command handlers and internal service methods.

### 7.1 Create Note

```ts
createNote(input: {
  kind: 'task' | 'project' | 'goal' | 'person' | 'habit';
  title: string;
  metadata?: Partial<NoteFrontmatter>;
  content?: string;
  template?: string;
}): Promise<{ path: string; created: boolean }>;
```

Behavior:
- Chooses canonical folder + naming convention.
- Applies template.
- Merges provided metadata.
- Guarantees required fields and timestamps.

### 7.2 Update Metadata

```ts
updateMetadata(input: {
  path: string;
  set?: Partial<NoteFrontmatter>;
  addToArray?: Record<string, string[]>;
  removeFromArray?: Record<string, string[]>;
  touchUpdated?: boolean;
}): Promise<{ path: string; updated: boolean; frontmatter: NoteFrontmatter }>;
```

Behavior:
- Frontmatter-aware patching (no manual YAML surgery required).
- Array merge and dedupe.
- `updated` timestamp refresh by default.

### 7.3 Query Insights

```ts
getInsights(input?: {
  types?: string[];
  domains?: string[];
  statuses?: string[];
  includeArchived?: boolean;
  limit?: number;
}): Promise<{
  generatedAt: string;
  health: 'healthy' | 'warning' | 'critical';
  cards: InsightCard[];
  metadataWarnings: MetadataWarning[];
}>;
```

Behavior:
- Structured payload suitable for Palmer cron jobs and bead creation automation.
- Includes missing metadata warnings as first-class output.

### 7.4 Transition Status

```ts
transitionStatus(input: {
  path: string;
  to: 'idea' | 'ready' | 'in-progress' | 'blocked' | 'done' | 'dropped';
  reason?: string;
}): Promise<{ path: string; from: string; to: string; updated: string }>;
```

Behavior:
- Validates allowed transition edges.
- Updates `status`, `updated`, and optional transition log block.
- Triggers re-analysis scheduling.

### 7.5 Batch Metadata Repair

```ts
repairMetadata(input: {
  scope?: { paths?: string[]; folders?: string[] };
  mode: 'propose' | 'apply-safe' | 'apply-all';
  maxEdits?: number;
}): Promise<{
  scanned: number;
  proposals: RepairProposal[];
  applied: number;
  skipped: number;
}>;
```

Behavior:
- Detects missing required fields and malformed arrays.
- Infers likely `type/domain/people/status` from content, links, and location.
- Produces reviewable proposals; safe mode only applies high-confidence changes.

## 8. Lifecycle and State Management

### 8.1 Transition Rules

```text
idea -> ready -> in-progress -> done
             \-> blocked -> in-progress
ready -> dropped
in-progress -> dropped
blocked -> dropped
```

### 8.2 Trigger Ownership
- Palmer/API triggers normal transitions during execution.
- Plugin auto-suggests transitions (never forces) based on signals:
  - completed checklist items + no remaining blockers -> suggest `done`
  - explicit `blocked_by` unresolved > threshold -> suggest `blocked`
  - stale `ready` > threshold -> suggest reprioritize/drop.

### 8.3 Automation Policy
- Default: semi-automatic (proposal + explicit apply).
- Optional setting: auto-apply safe transitions for API-originated tasks only.

## 9. Dependency Authoring UX (No YAML-Only Requirement)

### 9.1 Command Palette Flow
- `Life Garden: Add Dependency`
  - Select source note.
  - Select relation: `blocks` or `blocked_by`.
  - Select target via fuzzy note picker.
  - Plugin updates frontmatter safely.

### 9.2 Inline Syntax Extraction
- Support inline markers in note body:
  - `@blocks [[tasks/find-restaurant]]`
  - `@blocked_by [[tasks/confirm-availability]]`
- Parser harvests markers and syncs them into frontmatter (idempotent).

### 9.3 Incomplete Graph Handling
- Dependencies are additive, not mandatory.
- Missing dependencies degrade confidence score but do not block analysis.
- Insight cards show confidence (`high/medium/low`) based on metadata completeness.

## 10. Metadata Repair Strategy

### 10.1 Detection
Plugin flags notes when any of these are true:
- Missing required fields (`type`, `status`, `created`, `updated`).
- Scalar where array expected (`type: task` instead of `[task]` can be auto-normalized).
- Unknown status outside configured lifecycle.
- Empty task/project title sections.

### 10.2 Weekly Palmer Workflow
1. `getInsights` with `metadataWarnings` filter.
2. `repairMetadata(mode='propose')` to generate inferred fixes.
3. Review queue: apply high confidence immediately, leave medium confidence for approval.
4. Apply with `repairMetadata(mode='apply-safe')`.
5. Re-run analysis snapshot.

### 10.3 User Review Surface
- New dashboard tab: `Metadata`.
- Each proposal card shows:
  - current vs proposed frontmatter
  - confidence score
  - evidence lines (folder, links, keywords, recent edits)
  - actions: `Apply`, `Skip`, `Never suggest this rule`.

## 11. Inclusion, Archive, and Allow/Blocklist

### 11.1 Inclusion Rules
- Default include folders: `goals`, `projects`, `tasks`, `people`, `habits`, `journal`.
- Default exclude: `archive`, `reference`, `templates`, hidden folders.

### 11.2 Allow/Blocklist Settings
- `includeFolders: string[]`
- `excludeFolders: string[]`
- `excludeTags: string[]` (example: `archive`, `ignore-garden`)

### 11.3 Archive Link Policy
- If active note links to archived note:
  - Keep edge for context graph.
  - Exclude archived node from urgency rankings by default.
  - Show warning badge: `depends on archived context`.

## 12. Analysis and Update Cadence

- Real-time recomputation is not required.
- Default cadence:
  - light parse/index: every 15 minutes or on focused user action
  - full analysis: hourly
  - guaranteed freshness SLA: within 24 hours
- Manual command `Recompute Now` remains available.

## 13. Simplified Settings (7 Essential Controls)

```ts
interface LifeGardenSettingsV2 {
  includeFolders: string[];          // default core folders
  excludeFolders: string[];          // default ['archive','reference','templates']
  staleThresholdDays: number;        // default 14
  analysisCadence: 'manual' | 'hourly' | 'daily'; // default 'hourly'
  maxInsightCards: number;           // default 25
  autoRepairMode: 'off' | 'propose' | 'safe';     // default 'propose'
  includeWikiLinksInDependencies: boolean;         // default false
}
```

Advanced algorithm tuning (damping, centrality limits, etc.) is hidden under `Advanced` and untouched by default.

## 14. Edge Cases and Expected Behavior

1. Incomplete metadata: include note, mark `needsMetadataRepair`, lower confidence.
2. Mixed scalar/array frontmatter: normalize when safe, otherwise propose repair.
3. Path/title collisions: canonical id remains `TFile.path`.
4. Circular dependencies: SCC condensation for critical path summaries.
5. Archived dependencies: preserve context edge, exclude from urgency by default.
6. Very large vault: skip expensive metrics on low cadence; always return latest successful snapshot.

## 15. Revised Implementation Plan

1. Add vault conventions + template registry support.
2. Implement frontmatter patch service (`createNote`, `updateMetadata`, `transitionStatus`).
3. Add metadata warning detector and repair proposal engine.
4. Add command palette dependency authoring flow + inline extraction.
5. Expose local API for Palmer (`getInsights`, repair endpoints).
6. Add metadata dashboard tab and review actions.
7. Integrate allow/blocklist + archive policy.
8. Tune cadence defaults and add minimal settings UI.

## 16. How V2 Addresses Critical Issues

1. Metadata authoring hard problem: solved with template-backed `createNote`, patch API, repair workflows, and metadata warnings.
2. No Palmer integration story: solved with explicit local API for note CRUD, transitions, insights, and batch repair.
3. Missing lifecycle: solved with concrete state model, transition graph, and trigger ownership.
4. Rigid taxonomy: solved via multi-valued `type`/`domain` arrays and flexible tags.
5. Dependency model oversimplified: solved with command palette authoring, inline syntax extraction, and confidence-based incomplete handling.
6. Vault structure undefined: solved with explicit folder hierarchy, naming rules, and linking conventions.
7. Settings complexity: reduced to seven essential controls with advanced settings hidden.

## 17. Open Questions (Implementation-Scoped)

1. Should `journal/` be included in default ranking or only as context edges?
2. What confidence threshold should `apply-safe` use by default (e.g., 0.85)?
3. Should transition logs be embedded in note body or stored in plugin state?
4. Should metadata repair auto-run daily when Obsidian is idle?
