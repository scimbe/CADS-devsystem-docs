---
title: How auction selection policies work
description: LowestFloor, RoundRobin, LeastCalls -- what each actually does when a role has more than one qualifying bid.
order: 2
---

# How auction selection policies work

The Pipeline panel shows a `default selection policy` next to every run's pipeline id --
`LowestFloor` for every run created through devsystem-web today (see the honest caveat at the
bottom). This picks the winner whenever a role's auction has more than one real, qualifying bid.

## The three real policies

Defined in `ct_common::pipeline::SelectionPolicy` (CADS-Tunnel core, not devsystem-specific):

- **`LowestFloor`** (the default) -- cheapest offer wins, ties broken by holder key for
  determinism. Stateless: this is byte-identical to the original, pre-policy `convene()` behavior.
  Gives real priority-failover when a preferred provider publishes a lower floor than its standby.
- **`RoundRobin`** -- rotates across the qualifying providers: each auction picks the next
  provider in a deterministic ring (sorted by holder key), after whoever won last time. Over N
  stable providers this cycles 1 → 2 → … → N → 1, spreading load evenly.
- **`LeastCalls`** -- routes to whichever qualifying provider has served the fewest jobs so far
  (ties broken by floor, then holder key). Self-balancing: a freshly added copy of a provider
  starts at zero served jobs and is preferred until it catches up, draining backlog off the busy
  ones with no manual reconfiguration.

`RoundRobin` and `LeastCalls` are explicitly **stateful** -- each needs to remember something
(the last winner; how many jobs each provider has served) across separate auction calls to behave
as designed. `LowestFloor` reads and writes none of that state.

## A real bug this deployment had until 2026-08-05

Worth being honest about, not glossed over: `view_auction` (the handler behind
`GET /api/runs/{id}/auction`) used to construct a **fresh** `SelectionState` on every single call.
`LowestFloor` was unaffected -- it never reads that state anyway. But `RoundRobin` and
`LeastCalls` could never actually rotate or load-balance in this deployment: every request started
from a blank cursor, so `RoundRobin` always re-picked whichever provider sorts first, and
`LeastCalls` saw every candidate permanently tied at zero served jobs. Neither policy had ever
worked as designed here.

Fixed in [CADS-devsystem@5b51fb9](https://github.com/scimbe/CADS-devsystem/commit/5b51fb9):
devsystem-web now holds a real, persisted `SelectionState` per run, so the state `auction_view`
mutates on one request is genuinely still there on the next. A new hermetic test drives three real,
separate HTTP requests against a `RoundRobin`-configured run and proves the winner actually
rotates (and wraps back around with exactly two candidates) -- not just that the code compiles.

## Honest current limitation: no live way to choose a non-default policy

`create_run` always calls `plan_only_spec`, which hardcodes `SelectionPolicy::LowestFloor` --
there is currently no API route or GUI control to set a different pipeline-wide policy on a real
run. The fix above makes `RoundRobin`/`LeastCalls` *correct* the moment a spec actually declares
one, but reaching that state today means hand-editing a run's `spec.json` directly, not a
supported GUI flow. A real, separate, not-yet-scoped increment.
