---
title: "Why did my run pause itself?"
description: What happens when a milestone is achieved, and how to get going again.
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

## Getting going again

Click **Resume run** in the Health & Criteria panel, or call `POST /api/runs/{id}/resume`
directly -- there's no other gate on it; achieving a milestone doesn't require any additional
approval step beyond your own decision to resume. While paused, submitting a new iteration
(`POST /api/runs/{id}/iterate`) is refused with a real `409`, and the New Iteration panel's own
submit button disables itself with a `resume the run first` tooltip -- so this can't be missed
and silently worked around from the GUI.

## A real example

This page's own screenshot was captured the moment `webconference-android`'s **M1: 1:1
Text-Messaging end-to-end** milestone was confirmed for real (two actual Android emulator
instances exchanging a message over a live Noise_IK channel session, screenshots cross-checked
against each other before the milestone was toggled -- see
[CADS-devsystem issue #13](https://github.com/scimbe/CADS-devsystem/issues/13)) -- not staged
for this doc.
