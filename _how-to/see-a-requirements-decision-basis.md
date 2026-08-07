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

## Who confirmed a criterion, and when

Until 2026-08-07, a confirmed acceptance criterion was a bare `true` — the pipeline's own
highest-stakes verdict signal (a human saying "yes, this is actually met") carried no record of who
confirmed it or when, on any run, ever. Fixed: a confirmed criterion is now a real object, shown
right under the criterion's own strikethrough text, and a requirement now shows who created it too:

<figure>
<img src="{{ '/assets/img/howto-decision-basis/02-real-confirmation-provenance.png' | relative_url }}" alt="A requirement card showing 'created by scimbe@gmail.com', with two confirmed acceptance criteria -- one reading 'confirmed by scimbe@gmail.com, 8/7/2026, 6:45:15 PM' and the other reading 'confirmed 8/7/2026, 6:45:15 PM (no account on the session)'">
<figcaption>Two real, live-captured confirmations: one from a real gate-verified browser session, one from a header-less request -- honestly recording no actor rather than fabricating one.</figcaption>
</figure>

Both actors here are real, not staged text: `created_by` and `confirmed_by` are always stamped
server-side from the same real `X-Gate-Email` session header every other owner-scoped write in this
API already trusts — never a client-claimed value in the request body. A request with no session
(the local `devsystem_iterate` CLI, an M2M `--remote` submission) still records the *timestamp*
honestly, just with `confirmed_by: null` — "confirmed, no account on the session" is a real,
distinct state from "confirmed by someone," not a gap papered over.

**This does not repurpose `proposed_by`.** That field already had a different, correct meaning
before this fix — human-authored (`null`) vs. LLM-proposed (`Some(stage_tag)`) — and still does.
`created_by` is a new, separate field for "which real account."

### The honest fallback for history older than this fix

Every run's confirmed criteria that predate 2026-08-07 migrated automatically on load — a legacy
`true` becomes a real record, just one that honestly admits it doesn't know who or when:

<figure>
<img src="{{ '/assets/img/howto-decision-basis/03-legacy-migrated-fallback.png' | relative_url }}" alt="A requirement card on the real webconference-android run, with 'created by: not recorded (local CLI, M2M, or predates this field)' under the statement, and a confirmed criterion reading 'confirmed -- predates provenance tracking, who/when unknown'"
/>
<figcaption>The real flagship webconference-android run's own one confirmed criterion, migrated live -- the fact ("yes, confirmed") is preserved, honestly paired with "who/when unknown" instead of an invented actor or timestamp.</figcaption>
</figure>

Nothing is lost and nothing is invented: the migration was verified live against this exact run's
`state.json` before and after deploying, comparing all eighteen of its real requirements one by one.
See the [REST API reference]({{ '/reference/rest-api/' | relative_url }}#backlog-milestones-requirements)
for the wire shape.

## Correcting a wrong requirement

A requirement that turns out wrong or unsatisfiable as written isn't permanently stuck any more --
see [Correct a wrong requirement]({{ '/how-to/edit-a-requirement/' | relative_url }}) for the real
Edit control this panel now has, and why saving a correction honestly resets the requirement's own
confirmation state.

## What's still not in this view

The assistant's own chat exchanges about a requirement aren't pulled in yet — only real iteration
history. That's the "chat/docs" half of what a full unified decision-basis view would show; today's
version covers the iteration-history half.
