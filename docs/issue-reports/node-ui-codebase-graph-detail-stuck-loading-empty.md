# Issue report: DKG v10 Codebase graph detail page stays empty/loading even though the graph is present in the UI and backend

## Title
Context graph detail page stays on `Loading context graph...` for `DKG v10 Codebase Graphify rc17` even though the graph is listed and backend queries show data

## Summary
The Node UI lists `DKG v10 Codebase Graphify rc17` under **My Context Graphs**, but opening that graph does not render a usable detail view. Instead, the page remains on `Loading context graph...`, which makes the context graph appear effectively empty from the UI.

Daemon logs from the same interaction show the backend receiving multiple graph-specific queries for `dkg-v10-codebase-graphify-rc17`. Several of those return `0 bindings`, while broader graph queries for the same graph later return non-zero results (including `20000 bindings` and a grouped count query returning `463 bindings`).

This suggests the graph itself is present, but the detail page is blocked on missing/empty profile-style queries and never transitions out of the loading state.

## Environment
- Product: DKG Node UI
- Runtime observed: `@origintrail-official/dkg@10.0.0-rc.17`
- Install mode: npm-global
- Node/API: local node at `http://127.0.0.1:9200`
- Store backend: `oxigraph-server`
- Context graph: `dkg-v10-codebase-graphify-rc17`
- Graph display name: `DKG v10 Codebase Graphify rc17`

## What is visible in the UI
From the live UI during investigation:

### Dashboard/list state
The graph is visibly present in the UI:
- Sidebar shows `DKG v10 Codebase Graphify rc17` under **MY CONTEXT GRAPHS**
- Dashboard summary shows:
  - `MY CONTEXT GRAPHS: 1`
  - `CONTEXT GRAPH SIZE: 2,452 entities / 20,000 triples`
  - row label: `DKG v10 Codebase Graphify rc17`
  - role shown as `JOINED`

### Detail-page state
After opening the graph, the page shows:
- graph title: `DKG v10 Codebase Graphify rc17`
- body text: `Loading context graph...`

No usable detail content appears, so the graph looks empty/broken from the UI even though it is present in the graph list.

## API evidence
Browser-authenticated API inspection during the same session showed the graph is present in `/api/context-graph/list`:
- `id`: `dkg-v10-codebase-graphify-rc17`
- `name`: `DKG v10 Codebase Graphify rc17`
- `subscribed`: `true`
- `synced`: `true`
- `creator`: `did:dkg:agent:12D3KooWMqs7vspB9N3BRvdUxMEoH1pRxgLBzvepPmkp4TPao7oE`
- `callerInvolved`: `false`

The current agent DID in the browser matched that creator peer identity, so the graph is not absent from the backend registry/list response.

## Daemon log evidence
Relevant `daemon.log` lines from the graph-open attempt:

### The graph detail page triggers multiple graph-specific queries
At `2026-06-11 20:05:35` onward, the daemon receives several queries on:
- `contextGraph="dkg-v10-codebase-graphify-rc17"`

Examples include profile/summary-style queries beginning with prefixes like:
- `PREFIX prof: <http://dkg.io/ontology/profile/>`
- `PREFIX schema: <http://schema.org>`
- `PREFIX ag: <http://dkg.io/ontology/agent/>`

### Several graph-detail queries return zero results
The following graph-specific queries returned `0 bindings`:
- `query d728ab5e... returned 0 bindings`
- `query 80310704... returned 0 bindings`
- `query db0a339b... returned 0 bindings`
- `query c56b11c3... returned 0 bindings`
- `query e080e3e5... returned 0 bindings`
- `query 863a2845... returned 0 bindings`
- `query 7aa34158... returned 0 bindings`
- `query e2a0cd51... returned 0 bindings`
- `query 3da0717b... returned 0 bindings`
- `query bf7acdd1... returned 0 bindings`

### Broader graph queries later return real data
In the same interaction window:
- `query 35b3eac7... returned 20000 bindings`
- `query 9a46aa73...` performed a grouped count query over named graphs in `dkg-v10-codebase-graphify-rc17`
- `query 9a46aa73... returned 463 bindings`

This strongly suggests the graph is not actually empty. Instead, specific metadata/detail queries expected by the detail page are returning empty results, while broader data-extraction queries still find substantial graph content.

## Why this is a bug
The UI presents a context graph as available in the list view, with non-zero size, but the detail page does not degrade gracefully when its detail/profile queries return empty results. Instead, it remains in a perpetual loading state, which looks like an empty or broken graph.

This is misleading because:
- the graph exists,
- the graph has non-zero size in the list view,
- backend queries show data is present,
- but the detail page gives the impression that nothing is there.

## Expected behavior
When a context graph exists but detail/profile metadata queries return empty results:
- the page should stop loading,
- show a partial/limited detail view, or
- show an explicit empty-state explanation such as:
  - `Graph data exists, but no project/profile metadata was found`
  - `Showing raw graph counts only`

The UI should not remain indefinitely on `Loading context graph...`.

## Actual behavior
- Graph appears in **My Context Graphs** with non-zero counts.
- Opening the graph leads to a page that stays at `Loading context graph...`.
- Logs show graph-specific queries are running, but many return `0 bindings`.
- At least some broader queries against the same graph return substantial data, so the graph is not actually empty.

## Steps to reproduce
1. Open Node UI.
2. Confirm `DKG v10 Codebase Graphify rc17` appears under **My Context Graphs**.
3. Open that context graph.
4. Observe that the detail page remains on `Loading context graph...`.
5. Inspect daemon logs during the page load.
6. Observe multiple graph-detail queries returning `0 bindings`, alongside broader graph queries for the same context graph returning non-zero results.

## Suspected root cause
Likely one of the following:
1. The detail page assumes project/profile metadata exists in the graph and never handles the `0 bindings` case.
2. The graph was imported in a Graphify/raw-data shape that is sufficient for list/count queries but does not satisfy the detail page's expected profile schema.
3. The frontend waits on one or more metadata queries that resolve successfully-but-empty, but never exits the loading state when those responses are empty.

## Impact
- Makes a real graph appear empty/broken in the UI.
- Prevents inspection of imported codebase graphs from the detail page.
- Confuses operators because list view says the graph exists and has size, while detail view looks blank.
- Undermines confidence in whether the graph import actually worked.

## Suggested fixes
1. **Handle empty detail/profile query results explicitly**
   - If graph metadata/profile queries return zero rows, render a fallback detail view instead of indefinite loading.

2. **Separate graph existence from profile completeness**
   - A graph with data but missing profile metadata should still render raw stats, subgraphs, sample entities, or named-graph counts.

3. **Add defensive loading-state termination**
   - If all required detail requests have completed, even with zero bindings, leave loading state and render a degraded UI.

4. **Expose which detail section is empty**
   - Example: `No project profile found`, `No summary chips found`, `No agent metadata found`.

5. **Consider a Graphify/raw-import fallback renderer**
   - Especially for codebase/context graphs that are not shaped like hand-authored project profiles.

## Severity
Medium-High.

This does not necessarily mean the graph data is lost, but it blocks practical UI inspection of a real graph and makes the graph look empty from the user-facing surface.
