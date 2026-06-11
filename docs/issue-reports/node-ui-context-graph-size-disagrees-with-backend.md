# Issue report: Node UI shows empty/unknown context graph size while backend reports a non-empty graph

## Title
Node UI shows `No context graphs yet` / `—` size while backend reports `dkg-v10-codebase-graphify-rc17` with ~120k triples

## Summary
The live Node UI can show an empty/fallback state for context graphs at the same time that the backend still reports a real, non-empty context graph.

In the observed case, the UI at `http://127.0.0.1:9200/ui` showed:
- `My Context Graphs: 0`
- `Context Graph Size: —`
- `No context graphs yet`
- `Loading context graphs...`

But an authenticated backend call to:
- `GET /api/sub-graph/list?contextGraphId=dkg-v10-codebase-graphify-rc17`

returned a populated graph with these subgraphs:
- `code`: `13,742` entities / `107,500` triples
- `indexes`: `1,044` entities / `12,489` triples
- `reports`: `2` entities / `10` triples
- `architecture`: `0` entities / `0` triples

Subgraph total at the API layer:
- `14,788` entities
- `119,999` triples

So the user-visible UI currently implies “no graph / unknown size”, while the backend reports a substantial graph.

## Environment
- Product: DKG Node UI
- Runtime observed: `@origintrail-official/dkg@10.0.0-rc.17`
- Install mode: `npm-global`
- Local node URL: `http://127.0.0.1:9200`
- Store backend: `oxigraph-server`
- Context graph ID: `dkg-v10-codebase-graphify-rc17`
- Context graph name: `DKG v10 Codebase Graphify rc17`

## UI evidence
During the check, browser-visible text on the live UI showed:
- `No context graphs yet`
- `MY CONTEXT GRAPHS loading…`
- `CONTEXT GRAPH SIZE —`
- `Loading context graphs...`
- `My Context Graphs 0`

This is not just a smaller count or a rounding mismatch. The UI is effectively presenting the graph inventory as empty/unknown.

## Backend evidence
### Runtime status
`GET /api/status` was healthy and reported:
- version: `10.0.0-rc.17`
- commit: `3aae056b`
- store backend: `oxigraph-server`
- store URL: `http://127.0.0.1:7878/query`

### Graph size data
Authenticated request:
- `GET /api/sub-graph/list?contextGraphId=dkg-v10-codebase-graphify-rc17`

returned:

```json
{
  "subGraphs": [
    {
      "name": "code",
      "entityCount": 13742,
      "tripleCount": 107500
    },
    {
      "name": "indexes",
      "entityCount": 1044,
      "tripleCount": 12489
    },
    {
      "name": "architecture",
      "entityCount": 0,
      "tripleCount": 0
    },
    {
      "name": "reports",
      "entityCount": 2,
      "tripleCount": 10
    }
  ]
}
```

Summed API totals:
- entities: `14,788`
- triples: `119,999`

## Why this is a bug
The UI is not merely showing a different counting semantic. It is currently showing an empty/unknown state while the backend still has a real context graph with non-zero data.

That creates two user-facing problems:
1. the graph can look missing when it is not;
2. size/inventory debugging becomes much harder because the user cannot tell whether the problem is data loss, loading failure, auth/session state, or rendering fallback.

## Important note on counting semantics
The backend `entityCount` values above come from the daemon’s subgraph aggregation semantics and may not equal a globally deduped whole-graph entity count. Repeated subjects across named graphs can inflate summed entity totals.

However, that caveat does **not** explain the current UI state, because the UI is showing:
- `0` graphs / `—` size / `No context graphs yet`

while the backend clearly reports a non-empty graph.

## Expected behavior
If the backend can still resolve the graph and return non-zero subgraph counts, the UI should do one of the following:
- show the context graph with its available counts;
- show a partial-data state with explicit explanation;
- or show a targeted error message such as `Context graph list failed to load, but graph data is still available`.

It should **not** collapse into a generic empty-state message that implies there are no graphs.

## Actual behavior
- UI shows empty-state / unknown-size messaging.
- Backend subgraph-list endpoint returns a non-empty graph with ~120k triples.

## Steps to reproduce
1. Open the local Node UI.
2. Observe the dashboard/graph list state.
3. If the UI shows the empty/fallback state (`No context graphs yet`, `0`, `—`), query the backend directly with:
   - `GET /api/status`
   - `GET /api/sub-graph/list?contextGraphId=dkg-v10-codebase-graphify-rc17`
4. Compare the visible UI state with the API response.

## Suspected causes
Likely one of:
1. graph-list loading path and graph-detail/subgraph-count path are failing independently;
2. stale/invalid frontend state causes fallback rendering even though authenticated API access still works;
3. the UI treats context-graph-list failure as true emptiness instead of error/partial state;
4. inconsistent handling of auth/session/bootstrap across dashboard widgets and graph-specific endpoints.

## Suggested fixes
1. Distinguish **empty** from **failed to load** in the dashboard and graph list UI.
2. If graph-list fetch fails but graph-specific endpoints succeed, show a degraded/error state rather than `No context graphs yet`.
3. Add explicit diagnostics in the UI for which fetch failed:
   - context graph list
   - current agent identity
   - subgraph counts
4. Log or surface a correlation between frontend fallback state and live API success/failure.
5. Consider showing backend-resolved graph count/size whenever graph-specific APIs are still available, even if the broader list fetch is degraded.

## Severity
Medium-High.

This misrepresents the live state of the graph inventory and can make a non-empty context graph appear absent.
