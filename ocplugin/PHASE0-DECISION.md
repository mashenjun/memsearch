---
title: Phase 0 Decision Record - Trigger Policy
status: completed
created: 2026-02-20
completed: 2026-02-20
---

## Observations

Probe plugin: `ocplugin/.opencode/plugins/idle-probe.ts`
Output: `/tmp/idle-probe.log` (appendFileSync)
OpenCode SDK: `@opencode-ai/plugin@1.1.60`

### S1: Single-turn Q&A

Ask "What is 2+2?", wait for response, do nothing for 30s.

| Metric | Value |
|--------|-------|
| `session.idle` event count | 1 |
| `session.status type=idle` event count | 1 |
| Time from last `message.updated` to first `session.idle` | 7ms |
| Time from last `message.updated` to first `session.status type=idle` | 6ms |
| Does `session.status` show busy->idle transition? | Yes: busy(x3) -> idle(x1) |

### S2: Rapid multi-turn

Ask 3 questions sequentially (wait for each response before sending next).

| Metric | Value |
|--------|-------|
| `session.idle` events between turns | 1 per turn |
| `session.idle` events after final turn | 1 |
| `session.status type=idle` events between turns | 1 per turn |
| `session.status type=idle` events after final turn | 1 |
| Does idle fire between turns or only after the burst? | Between every turn |

### S3: Long idle (120s)

After S2 final response, waited 120s without typing.

| Metric | Value |
|--------|-------|
| Total `session.idle` events | 0 additional (beyond the per-response one) |
| Total `session.status type=idle` events | 0 additional |
| Does idle fire multiple times? At what intervals? | No. Fires exactly once per response. |
| Any `session.status type=busy` without a prompt? | No |

## Chosen trigger mode

- [x] **Per-response** -- `session.idle` fires once per assistant response
- [ ] **Timeout-based** -- `session.idle` fires after inactivity period
- [ ] **Mixed/unpredictable** -- use `session.status` busy->idle as primary

## Rationale

Both `session.idle` and `session.status type=idle` fire exactly once
after each assistant response completes, with 0-1ms between them.
Neither fires on a timer or repeats during inactivity. This is
pure per-response semantics -- the simplest and most predictable model.

## Implementation impact

- Primary event: `session.idle` (dedicated event, cleaner API)
- Fallback event: none needed (`session.status type=idle` fires identically)
- SessionState fields: no changes needed vs proposal baseline
- Debounce: not needed (events are already 1:1 with responses)
- The proposal's `session.idle` handler design is correct as-is

## Key finding: `session.idle` vs `session.status type=idle`

| Question | Answer |
|----------|--------|
| Do both events fire? | Yes, always |
| Which fires first? | `session.status type=idle` fires 0-1ms before `session.idle` |
| Are they 1:1 or does one fire more often? | Exactly 1:1 |
| Which one does the existing entire.ts plugin rely on? | `session.status type=idle` |

Both are equivalent. We use `session.idle` as primary because it's a
dedicated event with a simpler property shape (`{ sessionID }` vs
`{ sessionID, status: { type } }`). The existing `entire.ts` uses
`session.status` but that's a stylistic choice, not a functional one.

**This decision is FROZEN. All Phase 1+ implementation uses the
handler code specified in the corresponding design outcome in the
proposal.**

## Raw log summary

```
Total events captured: 4 response cycles
session.idle count: 4 (one per response)
session.status type=idle count: 4 (one per response)
session.status type=busy count: varied (multiple per cycle, includes retransmits)
No idle events during 120s inactivity window
```
