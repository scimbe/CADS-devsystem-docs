---
title: See a requirement's real decision basis
description: Trace why a requirement is where it is, without piecing it together from separate panels by hand.
order: 4
---

# See a requirement's real decision basis

Every requirement's card in the Requirements panel already shows which real iterations claimed to
address it ("addressed by iteration(s) N"). As of 2026-08-05 it goes further: a real, expandable
**decision basis** section underneath shows the actual feedback and real constraints from every one
of those iterations — the same data that used to require checking the separate History or Memory
Log panel and matching iteration numbers by hand.

## A real, live example

A real requirement, addressed by one real iteration that also proposed a new role:

```
$ curl -X POST .../api/runs/{id}/requirements \
    -d '{"statement": "WHEN a user taps send with an empty message, THE SYSTEM SHALL not attempt to send anything and SHALL leave the input focused for retry", "acceptance_criteria": ["empty/whitespace-only input never calls sendText", "the input field keeps focus after a no-op tap"]}'

$ curl -X POST .../api/runs/{id}/iterate \
    -d '{"stage": "devsystem.android_native_bridge", "feedback": "Found the real gap: onSendClicked only checked .isEmpty(), so whitespace-only input would have been sent as real content. Fixed with .isBlank(), checked before the session-readiness guard.", "succeeded": true, "requirement_indices": [0], "proposals": [{"proposed_by": "devsystem.android_native_bridge", "stage_id": "devsystem.review", "tag": "review", "rationale": "a UX fix touching real user input handling should get a second pair of eyes before the next release", "use_existing_service": "review-role", "units": 1, "price_ceiling": null}]}'
```

Opening that requirement's **decision basis** in the GUI shows exactly this, rendered from the same
real history data `GET /api/runs/{id}` already returns:

- **iteration 1** (`devsystem.android_native_bridge`, ok) — *"Found the real gap: onSendClicked
  only checked `.isEmpty()`, so whitespace-only input would have been sent as real content. Fixed
  with `.isBlank()`, checked before the session-readiness guard."*
- constraints for what comes next: *"devsystem.review: a UX fix touching real user input handling
  should get a second pair of eyes before the next release"*

That second line — **constraints** — is the same real derivation
[`envelope.rs`](https://github.com/scimbe/CADS-devsystem/blob/main/pipeline/src/envelope.rs) already
computes server-side for the Memory Log panel (`proposals.map(p => "stage_id: rationale")`),
mirrored client-side here so this view doesn't need a second fetch to see it.

## What's still not in this view

The assistant's own chat exchanges about a requirement aren't pulled in yet — only real iteration
history. That's the "chat/docs" half of what a full unified decision-basis view would show; today's
version covers the iteration-history half.
