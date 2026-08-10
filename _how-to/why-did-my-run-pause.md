---
title: "Why did my run pause itself?"
description: What happens when a milestone is achieved, a run hits its own bound, or a mandatory check-in fires -- and how to get going again.
order: 5
---

# Why did my run pause itself?

If a run you're watching suddenly stops proposing new iterations and a banner appears saying
**"paused -- no new iterations are accepted until resumed"**, this is expected behavior, not a
bug -- and it's the exact same mechanism [pausing to correct something]({{ '/reference/rest-api/' | relative_url }})
uses. Real, live capture of the `webconference-android` run's own **Health & Criteria** panel,
right after this happened for real:

![Paused-run banner in the Health & Criteria panel, with a Resume run button]({{ '/assets/img/howto-milestone-pause/01-paused-banner.png' | relative_url }})

## Why this happens

Every [`Milestone`]({{ '/reference/rest-api/' | relative_url }}) you (or the assistant, per your
own approval) add to a run is a real checkpoint you defined yourself -- something like *"1:1
text messaging end-to-end"*. Toggling one to `achieved` -- via the Milestones panel, or the real
`POST /api/runs/{id}/milestones/{index}/toggle` route directly -- **auto-pauses the run** as a
deliberate side effect, the same real code path either way. This is the "periodic check-ins with
the human owner" half of this pipeline's own governing design (`docs/development-system-goal.md`,
"a bounded super loop, not something that runs fully unsupervised") made concrete: a milestone is
exactly the kind of moment where the system should stop and let you look, not silently keep
proposing more work toward whatever comes next.

Because that side effect is real and easy to miss on what otherwise looks like a plain checkbox,
checking a milestone as achieved in the GUI now asks you to confirm first -- *"Mark this milestone
achieved? This pauses the run for real -- no new iterations are accepted until you resume it."*
Cancelling leaves the checkbox and the run exactly as they were. Un-checking an already-achieved
milestone has no such warning: it never auto-resumes the run, so there's nothing surprising about
it either way.

**Update, 2026-08-07**: a milestone achieved while the run is *already* paused for one of the real
bounds below no longer overwrites that reason -- a real, precisely-evidenced evaluator finding
([issue #50](https://github.com/scimbe/CADS-devsystem/issues/50)). Until this fix, achieving a
milestone always set `pause_reason` to its own free text, even on a run genuinely halted for
`max_consecutive_failures` or `max_iterations` -- silently replacing *"3 consecutive failed
iterations (limit 2)"* with *"milestone achieved: ..."* and permanently losing the real reason
(un-checking the milestone afterward did not restore it). Since a milestone's own description and
`achieved` flag are entirely self-serve -- any signed-in account can set both, on any unowned run
(see [Delete a run]({{ '/how-to/delete-a-run/' | relative_url }})'s own coverage of that gap) -- an
operator reading the banner had no way to tell they were about to `Resume` a run that had actually
hit its real failure budget. Fixed: the first real reason a run halts for now wins and stays visible
-- a milestone reached while already correctly paused for a real bound doesn't get to relabel why.

## The other real trigger: hitting the run's own bound

Milestones aren't the only real reason a run pauses. Every run has a real, operator-set
[`AbortCriteria`]({{ '/reference/rest-api/' | relative_url }}) -- `max_iterations` and
`max_consecutive_failures` -- and the "bounded super loop" this project's own governing design
describes only means something if hitting that bound actually stops the loop. Until 2026-08-06 it
didn't: the server correctly reported `"outcome": "Abort"` in its own response the moment a run hit
its bound, but nothing else happened -- a role-filler (or a careless script) could keep submitting
iterations past the configured ceiling forever, and the run's own real history would keep growing
past the number the operator had actually set.

Real, live-confirmed proof of the fix, current behavior: a run with `max_iterations: 1` accepts
exactly one real iteration --

```
$ curl -X POST .../api/runs/docs-run/iterate -d '{"stage":"devsystem.implement", ...}'
{"outcome": "Abort", "iteration": 1, ...}
```

-- and the run's own real state confirms `"paused": true` immediately after, the same real
mechanism the milestone case above uses: the next iteration attempt gets the identical real `409`
a milestone-paused run already gives, not silently accepted past the bound.

**Update, 2026-08-06**: the paused banner used to look identical no matter which of these caused
it -- a real, separate gap this page itself named as still open, closed the same day. The banner
now shows the actual real reason, live-confirmed for all three real triggers:

```
⏸ paused -- reached the 1-iteration limit -- no new iterations are accepted until resumed
⏸ paused -- milestone achieved: 1:1 messaging works end to end -- no new iterations are accepted until resumed
⏸ paused -- paused manually -- no new iterations are accepted until resumed
```

A consecutive-failure abort says so specifically too (*"3 consecutive failed iterations (limit
3)"*), distinct from an iteration-count abort -- not just "the run aborted."

**Update, 2026-08-07**: that same real, distinguishing reason now reaches the `/iterate` response
itself, not just a follow-up `GET` -- a real, cheap gap named in both [issue
#46](https://github.com/scimbe/CADS-devsystem/issues/46) and [issue
#47](https://github.com/scimbe/CADS-devsystem/issues/47): until this fix, the response to the exact
submission that triggered an abort only ever said `"outcome": "Abort"`, and the GUI's own status
line fell back to a generic *"too many consecutive failures, or the iteration ceiling was
reached"* -- unable to tell you which one actually happened, even though the server already knew.
Live-confirmed current behavior, the same two real submissions from above:

```
$ curl -X POST .../api/runs/docs-run/iterate -d '{"stage":"devsystem.implement", ..., "succeeded": true}'
{"outcome": "Abort", "iteration": 1, "pause_reason": "reached the 1-iteration limit", ...}
$ curl -X POST .../api/runs/docs-run/iterate -d '{"stage":"devsystem.implement", ..., "succeeded": false}'
{"outcome": "Abort", "iteration": 1, "pause_reason": "1 consecutive failed iterations (limit 1)", ...}
```

The GUI's own status line (right after **Submit iteration**) now shows this real, specific reason
too, not the old generic sentence.

**What if the reason itself is missing?** Every real code path that pauses a run today sets a real
reason alongside it, but `pause_reason` predates the other pieces of `RunState` by a while, so a run
whose own `state.json` was written before this field existed can genuinely have `paused: true` with
no recorded reason -- confirmed live on `webconference-android` itself, whose paused checkpoint
predates the field. Until 2026-08-06 this rendered as a bare `"paused"` with zero indication anything
was missing, which is exactly the kind of silent gap this whole project's documentation exists to
call out rather than paper over. Fixed to say so honestly instead:

```
⏸ paused -- no reason recorded -- no new iterations are accepted until resumed
```

Deliberately not backfilled with a guess (it's very likely the M1 milestone pause, given the run's
own history, but that's an inference, not something the data actually proves) -- an honest "we don't
know" beats a plausible-sounding but unverified claim. Real capture of `webconference-android`'s own
Health & Criteria panel showing exactly this:

![Paused banner honestly showing "no reason recorded" for a pause that predates reason-tracking]({{ '/assets/img/howto-milestone-pause/02-paused-banner-no-reason.png' | relative_url }})

## A third real trigger: a mandatory check-in fired

A run's `AbortCriteria` has a third field beyond the two bounds above: `checkin_every`, the "pause
at least this often for a human to look, even if nothing is failing" cadence. Its own doc comment
has always promised the run must pause here -- but until 2026-08-07 nothing did (real evaluator
finding, [issue #48](https://github.com/scimbe/CADS-devsystem/issues/48)): a fired check-in was
correctly reported (`"outcome": "CheckinDue"`), and correctly accepted and recorded, but the run
kept going regardless of how many boundaries it crossed unacknowledged. Fixed: a fired check-in
now pauses the run for real, the identical mechanism the other two triggers already use --

```
$ curl -X POST .../api/runs/docs-run/iterate -d '{"stage":"devsystem.implement", ...}'
{"outcome": "CheckinDue", "iteration": 1, ...}
$ curl .../api/runs/docs-run
"paused": true, "pause_reason": "check-in due -- iteration 1 crossed the every-1-iteration cadence"
```

**This one has a real, deliberate difference from the other two.** Acknowledging it (`POST
/api/runs/{id}/checkin/acknowledge`, or the **Acknowledge check-in** button in the Health &
Criteria/Check-in panels) both resumes the run *and* records that a human actually reviewed that
boundary -- a cadence check-in is a review checkpoint, not a stop, so acknowledging it is the real
decision to continue. Plain **Resume** also unblocks the run (unlike a ceiling pause, which refuses
even after resuming -- above), but it does *not* record the review: live-confirmed, resuming
five separate check-in pauses without ever acknowledging leaves the run genuinely running
(`"paused": false`) while `health.checkin_pending` stays `true` and the run keeps showing up as
needing attention in the Runs list and Open Points. Acknowledge, not Resume, is the button that
actually closes the loop.

## A fourth real trigger: a real decision left unanswered on the final slot

This one is not a `paused` pause at all -- it's a real `409` refusal on the submission itself,
same shape as the `max_iterations`/`max_consecutive_failures` ceiling above, but for a different
reason: [issue #39](https://github.com/scimbe/CADS-devsystem/issues/39)'s own "a run should not be
allowed to burn its final iteration with a blocking question outstanding." If a role-filler raised
a real open question through the [Decisions channel]({{ '/reference/rest-api/#decisions' | relative_url }})
and it's still unanswered, the ONE submission that would consume the run's last remaining
`max_iterations` slot is refused -- naming the real question, not just a generic count:

```
$ curl -X POST .../api/runs/docs-run/decisions -d '{"question":"should CADS-webconference ever support offline delivery?","options":["yes","no"]}'
$ curl -X POST .../api/runs/docs-run/iterate -d '{"stage":"devsystem.implement","succeeded":true,...}'
this would be the run's final iteration (1 of 1 max_iterations) while 1 real decision(s) remain
unanswered -- answer them first (POST /api/runs/{id}/decisions/{decision_id}/answer), or raise
max_iterations before submitting: should CADS-webconference ever support offline delivery?
HTTP 409
```

Deliberately narrow, matching the issue's own wording: only the submission that would consume the
*final* slot is refused -- an ordinary decision raised mid-run never blocks an ordinary iteration,
only the point where the run is about to lose its own ability to act on the answer at all. Answer
the decision (the same [Open Points]({{ '/how-to/work-through-open-points/#answering-an-escalated-question' | relative_url }})
input+Answer button, or `POST /api/runs/{id}/decisions/{decision_id}/answer` directly), and the
identical submission is let through:

```
$ curl -X POST .../api/runs/docs-run/decisions/<id>/answer -d '{"answer":"no, forward-only for this run scope"}'
$ curl -X POST .../api/runs/docs-run/iterate -d '{"stage":"devsystem.implement","succeeded":true,...}'
{"outcome":"Abort","iteration":1, ...}
HTTP 200
```

The real `Abort` outcome above is the run's own ordinary ceiling firing as expected once the
iteration lands -- this gate only ever blocks the one submission that would otherwise have skipped
past a real open question with no way back, not the ceiling itself.

## Getting going again

Click **Resume run** in the Health & Criteria panel, or call `POST /api/runs/{id}/resume`
directly -- there's no other gate on it; achieving a milestone doesn't require any additional
approval step beyond your own decision to resume. While paused, submitting a new iteration
(`POST /api/runs/{id}/iterate`) is refused with a real `409`, and the New Iteration panel's own
submit button disables itself with a `resume the run first` tooltip -- so this can't be missed
and silently worked around from the GUI.

**Until 2026-08-06, this held only for the GUI and the HTTP API -- not for `devsystem_iterate`'s
own local mode.** `devsystem_iterate` (no `--remote`, see [Bid for a role and submit a real
iteration]({{ '/how-to/submit-an-iteration/' | relative_url }})) reads and writes `runs/<run_id>/`
directly, with no HTTP layer in between at all -- and until this fix, nothing on that path checked
`paused` before applying an iteration. Live-confirmed before fixing: a scratch run with
`paused: true` accepted a real iteration through this binary, appended it to the run's history, and
left `paused` untouched -- the resume-first gate was only ever a GUI/HTTP-level courtesy, never
actually enforced for anyone submitting locally:

```
$ devsystem_iterate a-paused-run record.json
rejected: run is paused -- resume it first (devsystem-web POST /api/runs/{id}/resume, or clear
state.paused directly) before submitting another local iteration
```

Same real check, same message shape as the two `devsystem_iterate`-only gaps documented in
[Bid for a role and submit a real iteration]({{ '/how-to/submit-an-iteration/' | relative_url }})
(the `run_id` path-traversal guard and the `requirement_indices` bounds check) -- a validation that
existed only in `devsystem-web`'s HTTP handler, with the local CLI path calling `run_iteration`
directly and never going through it
([CADS-devsystem@7f09ae3](https://github.com/scimbe/CADS-devsystem/commit/7f09ae3)).

**If the run paused because it hit its own bound, resuming alone does not raise that bound, and
does not buy you another iteration either.** Until 2026-08-07 it did: `Resume` cleared `paused`
without re-checking whether the ceiling was still true, so the very next submission was accepted
and durably recorded one past the declared bound -- resume, submit, re-pause, repeat, one real
iteration of grace per click, forever (real evaluator finding, [issue
#46](https://github.com/scimbe/CADS-devsystem/issues/46)). Fixed: a run already at its
`max_iterations` or `max_consecutive_failures` ceiling is refused with a real `409` regardless of
whether `paused` was just cleared, naming the actual current count --

```
$ curl -X POST .../api/runs/docs-run/resume
{"paused": false, "pause_reason": null}
$ curl -X POST .../api/runs/docs-run/iterate -d '{"stage":"devsystem.implement", ...}'
already at 1 of 1 max iterations -- raise max_iterations for this run, or close it out;
resuming a run that already reached its ceiling does not raise the ceiling
```

To actually raise the ceiling, update the run's real criteria first (Health & Criteria panel, or
`POST /api/runs/{id}/criteria`), then resume -- `Resume` alone is only ever correct for pause
reasons where the underlying condition genuinely isn't still true the moment the run comes back: a
milestone, a manual pause, or (see below) a check-in you're prepared to acknowledge properly rather
than just wave past.

**`max_consecutive_failures` has one real escape `max_iterations` does not: a genuine success.**
This was a real, hard deadlock until 2026-08-07 -- the fix above checked the run's state *before*
the incoming submission, with no visibility into what was actually being submitted. Once
`consecutive_failures` reached the bound, **every** further call got the same `409`, including a
real `succeeded: true` submission that would have reset the streak to `0` -- exactly the remedy the
error message itself named, but the one submission that could never actually land (real evaluator
finding, [issue #47](https://github.com/scimbe/CADS-devsystem/issues/47), reported as a follow-up
comment on the same issue rather than filed fresh, since it was the same control degrading). The
only working way out was editing `max_consecutive_failures` -- a door the message never said was the
*only* one that opened.

Fixed the same day: the gate now looks at the incoming submission's own `succeeded` flag and lets a
real success through specifically because applying it is what clears the streak -- landing it is the
resolution, not a bypass. A further failure at the same bound is still refused outright, so this
stays a real ceiling, not a blanket unlock the instant anyone resumes. Real, live proof:

```
$ curl -X POST .../api/runs/docs-run/iterate -d '{"stage":"devsystem.implement","succeeded":false,...}'
already at 1 consecutive failed iteration(s), at or past the configured limit of 1 -- raise
max_consecutive_failures for this run, or submit a real, succeeded iteration to reset the streak
(a succeeded:true submission is let through specifically to clear it)
HTTP 409

$ curl -X POST .../api/runs/docs-run/iterate -d '{"stage":"devsystem.implement","succeeded":true,...}'
{"outcome":"Continue","iteration":2, ...}
HTTP 200
```

`max_iterations` has no equivalent escape -- iteration count only ever grows, so no incoming
`succeeded` flag changes anything there; raising the ceiling itself really is the only way past it.

**Raising it too far gets caught immediately, not after a round-trip.** All three fields share the
same real, generous-but-finite cap the server itself enforces (10,000 -- see
[the REST API reference]({{ '/reference/rest-api/' | relative_url }})'s `POST .../criteria` row for
why). Real, live capture -- typing an absurdly large value and clicking **Save criteria**:

![The Health & Criteria panel's edit form showing "All three fields must be at most 10,000." immediately after typing 999999 into max iterations and clicking Save -- no round-trip to the server needed]({{ '/assets/img/howto-criteria-cap/01-immediate-feedback.png' | relative_url }})

A real gap found and closed 2026-08-06: this used to only get caught server-side, after a real
round-trip (still a clear, specific error once it got there -- but one avoidable retry to discover
it). Simply adding a `max="10000"` attribute to the input wouldn't have been enough on its own: this
form's own **Save criteria** button is a plain click handler, not a real `<form>` submit, so the
browser never runs native HTML5 validation against it regardless of the attribute. The fix checks
the real bound explicitly before ever sending the request.

## A real example

This page's own screenshot was captured the moment `webconference-android`'s **M1: 1:1
Text-Messaging end-to-end** milestone was confirmed for real (two actual Android emulator
instances exchanging a message over a live Noise_IK channel session, screenshots cross-checked
against each other before the milestone was toggled -- see
[CADS-devsystem issue #13](https://github.com/scimbe/CADS-devsystem/issues/13)) -- not staged
for this doc.
