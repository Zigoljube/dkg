# Issue report: Node UI mixes demo fallback data with live-looking status and integrations

## Title
Node UI mixes demo fallback data with live-looking status and integrations

## Summary
When the Node UI falls back to demo data because the node is unavailable or timing out, the page still presents realistic operational state in the same visual language as live node data. The warning banner says the node is not responding and that the UI is showing example data, but the rest of the screen still shows believable node identity, sync/peer state, integrations marked as ready, named context graphs, and wallet/spending metrics.

This makes it hard for an operator to tell what is real vs mocked and can lead to incorrect debugging conclusions.

## Environment
- Product: DKG Node UI
- Source branch at time of report: `origin/main` synced local checkout
- Runtime observed during investigation: `@origintrail-official/dkg@10.0.0-rc.17`
- Evidence source: screenshot captured from local Node UI

## Screenshot evidence
Screenshot path used for this report:
- `/Users/clawdnode/.hermes/image_cache/img_7ac542a31dc1.jpg`

Visible on screen:

### 1. Explicit fallback/demo warning
The top banner says:

> Demo data — the node isn't responding (it may be down, timing out, or returning errors), so the UI is showing example data, not live node state. Check the daemon, then reload.

### 2. Header still presents live-looking operational state
Despite the fallback warning, the header shows:
- `mock-node-agent`
- `0x1111...1111`
- `synced`
- `12 peers`

These read like normal live node health/status indicators rather than an isolated mock state.

### 3. Dashboard is populated with believable project data
The page shows:
- `MY CONTEXT GRAPHS: 3`
- `COLLABORATING AGENTS: 5`
- Named context graphs:
  - `Pharma Drug Interactions`
  - `Climate Science`
  - `EU Supply Chain`

These are plausible, domain-specific example names and do not look obviously synthetic.

### 4. Integrations appear operational during fallback
The left sidebar shows:
- `OpenClaw — Chat ready`
- `Hermes — Chat ready`

The right-side agent panel also renders normally. In a fallback state caused by node unavailability, these readiness signals are confusing unless explicitly marked as independently verified from the node state.

### 5. Wallet and spending cards show believable numeric data
The page displays realistic wallet balances and spending values rather than obviously mock/sample placeholders. This increases the risk that the operator mistakes the page for partially live state.

## Why this is a bug
The UI is mixing two incompatible stories at once:
1. **The node is unavailable, and this page is using demo/example data**.
2. **The node appears synced, peered, configured, and actively serving meaningful operational/project metrics**.

Even though the warning banner is present, the rest of the screen is convincing enough that users may trust the mock content.

## Expected behavior
When the UI enters demo/example fallback mode:
- The fallback state should be unmistakable.
- Live-looking node health signals should be hidden, disabled, or clearly labeled as mock/sample.
- Integrations should not appear generically `ready` unless that health is verified independently and labeled as such.
- Context-graph names, balances, peer counts, and spending values should either be hidden or marked as sample/demo.
- The UI should avoid presenting fallback content in the same visual language as normal live data.

## Actual behavior
The UI shows a demo-data warning banner while also rendering:
- sync status
- peer count
- node identity
- integration readiness
- populated context graphs
- wallet balances
- spending metrics

All of these appear in the same normal dashboard shell, making mock state look live.

## Steps to reproduce
Likely reproduction flow:
1. Open Node UI.
2. Put the node/API into an unavailable, timing-out, or failing state.
3. Allow the UI to enter demo/example fallback mode.
4. Observe that the dashboard still renders realistic mock operational data alongside the warning banner.

## Impact
- Misleads operators during debugging.
- Reduces trust in Node UI observability.
- Makes it harder to distinguish true runtime state from fallback/demo state.
- Can produce false conclusions about context-graph availability, integration health, or wallet/node status.

## Suggested fixes
1. **Make fallback mode visually distinct**
   - Strong global `Demo mode` / `Fallback mode` treatment in header and dashboard shell.

2. **Suppress live-looking health indicators in fallback mode**
   - Hide or relabel sync state, peer count, node identity, and similar indicators.

3. **Separate integration health from node health**
   - If integrations are actually reachable while node data is not, label them explicitly, e.g. `Integration reachable; node unavailable`.

4. **Use obviously synthetic example data**
   - Replace realistic project names and plausible numeric values with clearly fake/sample values, or omit them entirely.

5. **Prefer a clearer degraded-state UX**
   - Consider separate modes for:
     - node unavailable
     - partial data
     - demo mode
   instead of one mixed presentation.

## Severity
Medium-High for operator trust and debugging correctness.

Not necessarily data-loss-critical, but it undermines the UI as a trustworthy operational surface.
