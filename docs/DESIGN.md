# Life Garden Plugin Design

## 1. Goals and Non-Goals

### Goals
- Build a fully local Obsidian plugin that continuously computes graph-based life-balance insights from vault notes.
- Keep insights fresh without manual triggers.
- Make behavior inspectable: users can understand why each alert/ranking appears.
- Scale to medium/large vaults (1,000+ markdown notes) with responsive UI.

### Non-Goals (Phase 1)
- No cloud sync/remote processing.
- No ML/NLP classification of notes beyond explicit metadata and links.
- No auto-editing of user notes.

## 2. Domain Model

### 2.1 Node Model
Each markdown note maps to one graph node.

```ts
interface GardenNode {
  id: string; // canonical = file.path
  path: string;
  basename: string;
  mtimeMs: number;
  ctimeMs: number;
  type?: 'task' | 'project' | 'goal' | 'person' | 'habit' | string;
  status?: 'ready' | 'blocked' | 'in-progress' | 'done' | string;
  priority?: 'P0' | 'P1' | 'P2' | 'P3' | string;
  domain?: 'social' | 'health' | 'work' | 'relationship' | 'growth' | string;
  tags: string[];
}
```

Rules:
- `id = TFile.path` to avoid ambiguity when note titles collide.
- Missing frontmatter is allowed; the node still participates via wikilinks.
- Unknown enum values are preserved as strings (inspectable, no hard fail).

### 2.2 Edge Model
Multiple edge kinds are supported; each edge stores provenance.

```ts
type EdgeKind = 'wikilink' | 'blocks' | 'blocked-by' | 'implicit-dependency';

interface GardenEdge {
  from: string; // source node id
  to: string;   // target node id
  kind: EdgeKind;
  directed: boolean;
  sourcePath: string; // note where edge originated
  evidence: string;   // raw token ([[X]] / frontmatter field)
}
```

Direction policy:
- `wikilink`: `source -> destination`.
- `blocks: [[B]]` on A means `A -> B` (A gates B).
- `blocked-by: [[B]]` on A means `B -> A`.
- If both fields exist and conflict, both edges are retained and flagged in diagnostics.

Edge weighting (for algorithms that support custom distance/weight):
- Default weight = `1`.
- `blocks`/`blocked-by` can optionally be weighted `2` for priority in critical-path mode (configurable).

### 2.3 Entity Scope
Default inclusion filter:
- Include all markdown notes.
- Add optional setting `includeTypes` to restrict to specific frontmatter `type` values.
- Always keep cross-type links if destination exists (to preserve graph structure).

## 3. Vault -> Graph Construction

## 3.1 Parsing Inputs
Sources used per file:
- `metadataCache.getFileCache(file)` for frontmatter + resolved links.
- `file.stat.mtime` for decay.
- `vault.cachedRead(file)` fallback only if cache is unavailable.

Extraction per file:
1. Create/update `GardenNode` from file metadata.
2. Parse frontmatter arrays/strings:
   - `blocks`
   - `blocked-by`
3. Read wikilinks from `CachedMetadata.links` and `CachedMetadata.embeds`.
4. Resolve linkpaths through `metadataCache.getFirstLinkpathDest(link, sourcePath)`.
5. Keep unresolved links in diagnostics but do not create node stubs by default.

### 3.2 Canonicalization Rules
- Trim whitespace from frontmatter link tokens.
- Accept formats: `[[Page]]`, `Page`, and arrays.
- Convert link targets to file paths via Obsidian resolver.
- Deduplicate edges by `(from,to,kind)`.

### 3.3 Graph Store Representation
Maintain two synchronized structures:
1. `ngraph.graph` runtime graph for algorithm execution.
2. Index maps for fast incremental updates:

```ts
interface GraphIndex {
  nodesByPath: Map<string, GardenNode>;
  outgoingByPath: Map<string, GardenEdge[]>;
  incomingByPath: Map<string, GardenEdge[]>;
  unresolvedLinksByPath: Map<string, string[]>;
  fileFingerprintByPath: Map<string, string>; // hash of relevant fields
}
```

`fileFingerprint` includes:
- `mtime`, frontmatter dependency fields, resolved wikilinks, and status/domain/type.
- Skip recomputation if fingerprint unchanged.

## 4. Incremental Update Architecture

## 4.1 Event Sources
Register Obsidian events in `onload()`:
- `metadataCache.on('changed', ...)`
- `metadataCache.on('deleted', ...)`
- `vault.on('rename', ...)`
- `workspace.onLayoutReady(...)` for first full index

### 4.2 Update Strategy
Use a debounced job queue (e.g., 300-800ms configurable):
- Coalesce many rapid edits into one batch.
- Prioritize changed files only.
- Run full rebuild only when:
  - plugin first loads,
  - settings affecting inclusion/direction change,
  - explicit "Rebuild Index" command is invoked.

Incremental flow for changed file:
1. Recompute file fingerprint.
2. If unchanged: skip.
3. Remove prior edges originating from that file.
4. Re-parse and re-add node+edges.
5. If rename: migrate node id/path and rewrite incident edges.
6. Mark impacted connected component as "dirty" for algorithm refresh.

Algorithm recomputation policy:
- PageRank + betweenness: whole active graph (global metric).
- Decay: changed nodes only, then merge.
- Critical path: recompute only for goals in dirty component.

## 4.3 Concurrency and Staleness Guarantees
- Use monotonic `analysisVersion` token.
- Each async pipeline run captures start version and aborts publish if superseded.
- UI always renders latest committed version only.

This satisfies "if you see results, they are current" within the debounce window.

## 5. Algorithm Pipeline

Execution order per analysis tick:
1. `Build/Update Graph`
2. `Compute Decay`
3. `Compute PageRank`
4. `Compute Betweenness Centrality`
5. `Compute Critical Paths`
6. `Derive Insight Cards`
7. `Persist Snapshot + Diagnostics`

### 5.1 PageRank
Library: `ngraph.pagerank(graph, damping, precision)`

Defaults:
- damping: `0.85`
- precision: `0.0005` (tunable for large vaults)

Output normalization:
- Keep raw score.
- Also compute percentile rank for UI labels (`Top 5% load-bearing`).

Interpretation:
- Higher score = structurally important note where many paths/links converge.

### 5.2 Betweenness Centrality
Library: `ngraph.centrality.betweenness(graph, directed)`

Defaults:
- `directed = true` to preserve dependency semantics.
- Optional setting to use undirected for purely associative vaults.

Interpretation:
- High score = bottleneck/chokepoint.
- Surface with dependent count and nearby blocked items.

Complexity note:
- `O(n * e)`, so gate with performance settings for very large graphs (see section 8).

### 5.3 Decay Detection
Definition:
- Node is stale if:
  - `status != done` (configurable list of terminal statuses)
  - `now - mtime >= staleThresholdDays`

Defaults:
- `staleThresholdDays = 14`

Derived fields:
- `daysStale`
- `staleBucket`: 2w+, 1m+, 2m+, 6m+
- `domainDebtScore`: weighted stale count by domain and priority.

### 5.4 Critical Path Analysis
Goal: For each `type=goal`, find dependency chain controlling completion.

Graph for critical path:
- Use directed dependency edges only (`blocks`, `blocked-by`, optionally `wikilink` if setting enabled).

Algorithm approach:
1. Extract reachable dependency subgraph from each goal.
2. Detect SCCs (strongly connected components).
3. Condense SCC graph into DAG.
4. Run longest-path dynamic programming on DAG:
   - edge weight default 1 or configurable by priority/status.
5. Expand SCC nodes in output as cycle groups.

Why this approach:
- Longest path is NP-hard on general cyclic graphs; DAG condensation gives deterministic practical result for real vaults with cycles.

Output:
```ts
interface CriticalPathResult {
  goalId: string;
  pathNodeIds: string[]; // ordered bottleneck chain
  totalCost: number;
  cycleGroups: string[][]; // SCCs with >1 node
  blockedCount: number;
}
```

## 6. Insights and Scoring Layer

Convert raw algorithm results into user-facing insights.

### 6.1 Insight Types
- `loadBearing`: top PageRank nodes.
- `bottleneck`: top betweenness nodes.
- `decay`: stale non-done nodes.
- `criticalPath`: per-goal path summaries.
- `domainImbalance`: domains with highest debt score.

### 6.2 Prioritization Formula
Compute composite urgency score for sorting card feed:

```text
urgency =
  w_rank * rankPercentile +
  w_betweenness * bottleneckPercentile +
  w_decay * decaySeverity +
  w_priority * frontmatterPriorityWeight
```

Default weights:
- `w_rank=0.25`, `w_betweenness=0.30`, `w_decay=0.30`, `w_priority=0.15`

All weights configurable.

## 7. UI/UX Design

## 7.1 Surfaces
1. **Life Garden View (sidebar ItemView)**
   - Primary dashboard with tabs: `Now`, `Bottlenecks`, `Decay`, `Goals`, `Diagnostics`.
2. **Command palette actions**
   - `Life Garden: Open dashboard`
   - `Life Garden: Rebuild index`
   - `Life Garden: Copy diagnostics`
3. **Status bar indicator**
   - `Garden: Healthy | Warning | Critical` + last update timestamp.
4. **Optional inline notices**
   - Warn on large lag or parsing errors.

### 7.2 Card Format (Inspectable by Design)
Each insight card displays:
- note title/path
- score and rank
- reason lines (e.g., "Betweenness 98th percentile", "Stale 31 days")
- evidence links:
  - `View dependencies`
  - `Open note`
  - `Why this?` expandable raw metrics JSON

### 7.3 Diagnostics Panel
Show:
- indexed files count
- edge counts by kind
- unresolved links
- frontmatter parse warnings
- last pipeline duration per stage
- last successful analysis timestamp

## 8. Performance Strategy

### 8.1 Baseline Targets
For 1,000 markdown notes / 8,000 edges:
- Incremental update publish: < 500 ms typical
- Full rebuild: < 5 s on modern desktop
- UI frame blocking: avoid > 16 ms chunks where possible

### 8.2 Optimizations
- Fingerprint-based file skip.
- Debounced batch processing.
- Separate quick metrics (decay) from heavy metrics (betweenness).
- Adaptive heavy-metric throttling:
  - If graph exceeds `betweennessNodeLimit` (default 2,500), run betweenness at reduced cadence (e.g., every Nth batch) or scoped to active domains.
- Cache previous algorithm outputs and diff changed top-N before rerender.

### 8.3 Memory Management
- Keep only needed metadata in index.
- Release stale per-file caches on delete/rename.
- Persist compact snapshot to plugin data for fast startup render, then refresh in background.

## 9. Error Handling and Degradation

### 9.1 Parse-Level Errors
Examples:
- malformed frontmatter
- invalid field type
- unresolved dependency links

Behavior:
- log warning in diagnostics
- continue processing remaining files
- never crash global pipeline

### 9.2 Algorithm-Level Errors
Examples:
- numeric instability / unexpected library error

Behavior:
- keep last good result for that metric
- mark metric as stale with timestamp
- show non-intrusive notice and diagnostics entry

### 9.3 UX Behavior on Failure
- Dashboard remains usable with partial results.
- `Diagnostics` tab provides explicit explanation and affected files.

## 10. Settings and Configuration

```ts
interface LifeGardenSettings {
  staleThresholdDays: number; // default 14
  includeTypes: string[]; // default [] => all
  terminalStatuses: string[]; // default ['done']

  pagerankDamping: number; // default 0.85
  pagerankPrecision: number; // default 0.0005
  betweennessDirected: boolean; // default true

  dependencyEdgeWeight: number; // default 2
  includeWikiLinksInCriticalPath: boolean; // default false

  analysisDebounceMs: number; // default 500
  autoRebuildOnStartup: boolean; // default true

  betweennessNodeLimit: number; // default 2500
  heavyMetricIntervalBatches: number; // default 2

  showStatusBar: boolean; // default true
  maxInsightCards: number; // default 50
}
```

Extensibility:
- Future `customScoringRules` array for user-defined formulas.
- Future per-domain stale thresholds.

## 11. File and Module Organization

```text
src/
  main.ts                         # Plugin entry, command registration
  settings.ts                     # Settings schema/defaults/tab UI

  core/
    types.ts                      # Domain interfaces
    graph-builder.ts              # Vault file -> nodes/edges
    graph-store.ts                # ngraph + indexes + incremental mutation
    analyzer.ts                   # Pipeline orchestrator + versioning

  algorithms/
    pagerank.ts                   # Wrapper + normalization
    betweenness.ts                # Wrapper + throttling helpers
    decay.ts                      # Staleness/domain debt logic
    critical-path.ts              # SCC condensation + longest DAG path

  insights/
    insight-engine.ts             # Convert metrics to insight cards
    explain.ts                    # Human-readable reason generation

  ui/
    life-garden-view.ts           # ItemView dashboard
    components/                   # Tab renderers/cards
    status-bar.ts                 # status item

  obsidian/
    event-bus.ts                  # metadata/vault event wiring
    file-resolver.ts              # link resolution helpers

  infra/
    logger.ts                     # structured logging
    perf.ts                       # timers/metrics
    storage.ts                    # save/load snapshots

  tests/
    fixtures/
      small-vault/
      cyclic-deps/
      large-synthetic/
    unit/
    integration/
```

## 12. Testing Strategy

## 12.1 Unit Tests
Targets:
- frontmatter parsing normalization (`blocks`, `blocked-by` variations)
- link resolution behavior and dedupe
- decay calculations around threshold boundaries
- critical-path SCC condensation and longest-path correctness
- urgency score sorting determinism

### 12.2 Integration Tests
Approach:
- Mock minimal Obsidian APIs (`Vault`, `MetadataCache`, `TFile`) with deterministic fixtures.
- Build graph from fixture vault and assert:
  - node/edge counts
  - top-N PageRank/betweenness stable ordering
  - decay list correctness
  - critical path output for known goal chains

### 12.3 Performance Tests
Fixture: `large-synthetic` generator (1k/5k/10k nodes).
Assertions:
- full rebuild and incremental update time budgets.
- betweenness throttling triggers as expected.

### 12.4 Regression Harness
- Snapshot JSON for derived insight feed.
- On algorithm/config changes, diff snapshots intentionally.

## 13. Edge Cases and Expected Behavior

1. **Large vaults (1000+ notes)**
- Use incremental updates, throttled heavy metrics, capped UI cards.

2. **Circular dependencies**
- Detect via SCC; show cycle group in critical path diagnostics.
- Do not infinite-loop; longest path runs on condensed DAG.

3. **Missing frontmatter**
- Note still included with defaults.
- No dependency edges from missing fields.

4. **Invalid wikilinks / unresolved note names**
- Keep as unresolved diagnostic entry.
- Excluded from graph edges unless target resolves.

5. **Renamed files**
- Remap node id and incident edges using path migration; preserve metrics on next run.

6. **Deleted files**
- Remove node/edges and re-run affected analyses.

7. **Conflicting dependency fields**
- Keep both edges, emit warning for inspectability.

## 14. Observability and Inspectability

- Structured logs with stage timings.
- Diagnostics tab export (JSON) for issue reports.
- Per-card explanation payload with source edges and scores.
- Optional developer setting: `debugMode` to show raw metric tables.

## 15. Implementation Plan (Phase 2)

1. Scaffold plugin from Obsidian sample structure.
2. Implement `graph-builder` + `graph-store` with full rebuild.
3. Add event-driven incremental updates.
4. Integrate algorithms with wrappers.
5. Build insight engine + sidebar UI.
6. Add diagnostics + status bar.
7. Add tests (unit/integration/perf).
8. Tune defaults against large synthetic fixtures.

## 16. Key Decisions

- Node identity uses file path, not title.
- Unresolved links are diagnosable but non-fatal.
- Global metrics recompute globally; critical paths scoped by dirty goals.
- Critical path on SCC-condensed DAG for cycle-safe determinism.
- Inspectability is first-class via diagnostics and per-card "Why" explanations.

## 17. Open Questions for Review

1. Should wikilinks be included in critical-path analysis by default, or dependency fields only?
2. Should betweenness run on every batch for small vaults but scheduled (e.g., every 2-5 batches) for large vaults?
3. Should unresolved links create optional placeholder nodes for "planned" items?
4. Should domain imbalance score prioritize stale `P0/P1` more aggressively (default currently moderate)?
5. Should there be a hard file/type allowlist by default to reduce noise in mixed vaults?
