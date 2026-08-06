---
title: "Why did my run pause itself?"
description: What happens when a milestone is achieved or a run hits its own bound, and how to get going again.
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

**If the run paused because it hit its own bound, resuming alone doesn't raise that bound** --
live-confirmed: resuming a run that hit `max_iterations: 1` and submitting one more real iteration
gets accepted (it's genuinely below the new, post-resume gate check), but immediately re-aborts and
re-pauses on that same submission, since the run is still, correctly, at its configured ceiling.
Resuming buys you exactly one more real iteration's worth of grace, not a reset. To actually raise
the ceiling, update the run's real criteria first (Health & Criteria panel, or
`POST /api/runs/{id}/criteria`), then resume.

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
