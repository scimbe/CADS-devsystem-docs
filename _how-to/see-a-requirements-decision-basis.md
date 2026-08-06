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

## A real gap this view surfaced, found and fixed live (2026-08-06)

Long feedback text in the decision basis is shown truncated (a `"…"` preview, not the full text) --
and until this fix, the truncation itself could corrupt an emoji or any other real character
outside the Basic Multilingual Plane if it happened to straddle the cut point. JavaScript strings
index by UTF-16 code unit, not real character, and a plain `slice(0, n)` doesn't know the
difference -- reproduced directly before fixing:
`truncate("x".repeat(219) + "😀" + "y".repeat(50), 220)` returned a string ending in a bare,
unpaired high surrogate with no matching low half.

Fixed ([CADS-devsystem@9d3dcf0](https://github.com/scimbe/CADS-devsystem/commit/9d3dcf0)) with
`Array.from()`, which iterates by real Unicode code point instead of raw code unit, so a
surrogate pair is never split. Real, live capture after the fix -- a requirement addressed by an
iteration whose feedback contains an emoji positioned exactly at the truncation boundary:

<figure>
<img src="{{ '/assets/img/howto-decision-basis/01-emoji-intact.png' | relative_url }}" alt="The Requirements panel's decision basis section showing 'emoji test 😀 more t…' -- the emoji rendered whole and intact right at the truncation cut point, not split into a broken character">
<figcaption>The emoji survives the cut whole -- no broken glyph, no replacement character.</figcaption>
</figure>

## What's still not in this view

The assistant's own chat exchanges about a requirement aren't pulled in yet — only real iteration
history. That's the "chat/docs" half of what a full unified decision-basis view would show; today's
version covers the iteration-history half.
