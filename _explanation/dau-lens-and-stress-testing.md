---
title: The DAU lens and the incompetent-agent stress test
description: The methodology behind this project's confirm() dialogs and mechanical gates -- what it is, why it exists, and its real track record.
order: 6
---

# The DAU lens and the incompetent-agent stress test

Almost every gate, confirmation dialog, and validation check documented elsewhere on this site
traces back to one governing principle, stated directly by this project's own operator and quoted
verbatim in its [goal document](https://github.com/scimbe/CADS-devsystem/blob/main/docs/development-system-goal.md):

> It is the fault of the pipeline, not the user of the pipeline, if the process leads him not to the
> perfect result.

That's not a slogan -- it's a concrete methodology this project actually runs, repeatedly, against
its own real, live deployment. This page collects what that methodology is and what it's actually
found, so the reasoning behind any one fix doesn't have to be pieced together from scattered
mentions across other pages.

## Two lenses, one principle

**The incompetent-agent stress test** simulates the least competent realistic role-filler on
purpose -- not malicious, just lazy, careless, or genuinely mistaken -- and drives it against a real
run's real API. Not a thought experiment: an actual `POST` against the actual deployment, with the
actual response checked, before anything is called a gap. If a lazy shortcut gets a real `200` it
shouldn't, that's a real, fixable hole in a mechanical check, found the same way a bad actor
eventually would.

**The DAU lens** ("dumbest available user") asks the same question from the human side: a person
with poor judgment, who mostly listens to `devsystem.assistant` but sometimes chooses wrong, clicks
around the GUI. Every destructive or hard-to-undo action gets checked for whether a careless click
can trigger it with zero warning.

Both share the same discipline this project applies everywhere: **crude, mechanical, explainable
checks -- never fake LLM-judgment-in-disguise.** A length bar, a word-boundary check, a distinct-word
count. Nothing here claims to verify that a review is actually *good* -- only that it isn't trivially
empty, repetitive, or misapplied. See [How real risk annotations work]({{ '/explanation/risk-annotations/' | relative_url }})
for what these checks look like in the code.

## Now a real, reusable regression guard, not just a narrative

Thirty-four rounds in, this whole methodology was still one-off manual investigation every single
time -- nothing stopped a later change from silently reintroducing a gap already found and fixed.
[`scripts/incompetent-agent-stress-test.sh`](https://github.com/scimbe/CADS-devsystem/blob/main/scripts/incompetent-agent-stress-test.sh)
is a real, live-HTTP script that reproduces twenty-three of the concrete lazy shortcuts below
(duplicate `run_id` clobbering, an unbounded/zero `AbortCriteria`, whitespace-only fields, the
"shallow" SHALL-substring bug, an unbounded `price_ceiling` going unflagged (including a later,
bounded re-proposal for the same stage correctly clearing that flag -- the exact mechanism that had
two real regressions earlier this session), cross-account access, a "deleted" run not actually
being gone, `devsystem.assistant`'s own requirement-verification evidentiary gate, a role-filler
forging fake markdown structure in the real requirements export, a proposed GitHub issue targeting
an arbitrary repo outside the real allowlist, a succeeded iteration whose own feedback admits a
known defect, empty/whitespace-only iteration feedback, a run genuinely refusing further iterations
once it hits its own configured bound, the Runs list's own `pending_reviews` count missing two of
five real proposal queues, an empty/whitespace-only `holder_label` when directly accepting a
bid, and an absurdly large or zero `units` value at all three real `StageProposal` entry points)
against a real running deployment, creating and cleaning up its own real scratch run every
time via the actual `DELETE /api/runs/{id}` endpoint. It's now wired into this project's own real CI
(`pipeline-ci.yml`'s `web` job, confirmed green against a real GitHub Actions run, not just
locally), run against the exact Docker image that gets deployed -- a PR that reintroduces one of
these twenty-three fails CI instead of waiting for the next manual stress-test firing to notice.
Honestly scoped, and
self-correcting: the evidentiary-gate check above was originally left out on the wrong assumption it
needed a real LLM call to test -- a later firing caught that it's actually pure header-based server
logic (`X-Actor: devsystem.assistant`, no LLM involved) and added it for real. What's still
genuinely excluded: prompt-injection resistance and the assistant's own milestone-pause disclosure,
which need a real, non-deterministic LLM reply a human judges, not a fast boolean check.

The same investigation that produced the harness also found a real gap in this project's own §5
quality-bar table: it named `check-no-secrets.sh` as a real secrets-scanning gate, but that script
had never actually existed in this repo at all -- a different project's convention, referenced in
prose but never built here. A real, adapted version now exists and runs in its own CI job.

Later still, observing the actual live GitHub Actions state (not simulating anything) found a real
CI hygiene gap: this project's own frequent pushes had no `concurrency` group, so four separate CI
runs were genuinely stacked in-progress at once, none of them cancelled, even though only the last
push's result actually matters for `main`'s current health. Fixed with a standard concurrency group
-- and live-verified it actually took effect, not just reasoned about: the very next push after the
fix landed showed a real `cancelled` conclusion on the run it superseded.

Every per-run list in this codebase already had a real defensive cap -- `create_run` itself didn't,
on the total NUMBER of runs. With 110 real runs already on a host at 91% disk, and `list_runs`
doing a full filesystem read for every run on every dashboard refresh, unbounded growth would have
degraded the whole GUI for every real user, not just eaten disk. A real cap now exists, verified via
a hermetic test rather than the live harness above -- testing it live would need either 2000 real
scratch runs (worsening the exact clutter problem the delete-run feature exists to fix) or direct
filesystem access a remote script doesn't have.

Every other real free-text field already rejected whitespace-only content -- an iteration's own
`feedback` was the one exception, silently accepting a `succeeded: true` iteration with zero real
account of what happened.

Two more real gaps, both about the Runs list silently hiding something a human needs to see. The
Pipeline panel's own pending-proposal chip badge was already fixed once for undercounting (missing
panel-removal/edit proposals) -- the Runs list's own separate `pending_reviews` count had the exact
same bug, never fixed at this call site: a real pending panel-removal proposal showed `0`. And
`paused` was already in the Runs list's own real API response and never once checked by the GUI's
badge logic -- a fully paused run could show zero badge at all, indistinguishable from a healthy
one at a glance. Both fixed the same day; the paused badge now shows the real reason (see below),
confirmed with a real Playwright screenshot of the actual rendered GUI, not just the API payload.

While checking whether that same undercounting bug had a third instance anywhere (it didn't), found
something bigger: the Runs list sorted purely alphabetically, no priority at all. Live-confirmed
against the actual deployment, not a synthetic scenario: the real flagship `webconference-android`
run -- genuinely paused, needing an actual human decision -- sat at position 105 of 110, behind well
over a hundred alphabetically-earlier scratch runs with nothing outstanding. Fixed to sort by the
same real urgency order the badge already uses (paused first, then pending review, then needs
attention, then stalled, then risk, alphabetical only as the tie-break within a tier) -- confirmed
live afterward that the real flagship run moved to position 0.

One more instance of a pattern repeated all session: directly accepting a bid (skipping the
auction) already validated its own `label` as non-empty, but the nested `accepted_bid.holder_label`
-- a real identity record of who actually won the bid -- had zero validation. Live-confirmed before
touching anything: both a byte-empty and a whitespace-only `holder_label` got a real `200`. Fixed
the same way every other real free-text field in this codebase already is.

## The most significant finding this methodology has produced

Every gate above assumes the pipeline's own "bounded super loop" -- the central architectural claim
repeated throughout this project's own design docs -- actually means something. It didn't.
`RunOutcome::Abort` was purely advisory: the server correctly reported `"outcome": "Abort"` the
moment a run hit its own configured `max_iterations`/`max_consecutive_failures`, but nothing else
happened. Live-confirmed before anything was touched: a run capped at `max_iterations: 2` accepted
a real third and fourth iteration anyway, its own history growing to double the configured,
operator-set bound.

Fixed at the root, not patched at one call site: `run_iteration` itself now pauses the run the
moment it aborts, reusing the exact same mechanism the milestone-pause case above already
established -- see [Why did my run pause itself?]({{ '/how-to/why-did-my-run-pause/' | relative_url }})
for the second real trigger this added. Every real entry point (the GUI, the REST API, and the
local `devsystem_iterate` CLI) shares the identical fix automatically, since they all funnel
through the one function that now enforces it.

The fix itself named a real, honest gap: a milestone achieved, an abort ceiling reached, and a
human's own manual pause all set the identical flag with no way to tell them apart. Closed the same
day -- the paused banner now shows the actual real reason for all three, not a generic message.

## The real track record

As of this writing, the stress test has run **fifty-six** real rounds against the actual
deployment, finding and closing forty-three real gaps -- not simulated, not hypothetical. A
representative sample, each with its own real live before/after proof:

- A one-line rubber-stamp review (`"looks fine to me"`) satisfied the mandatory review gate just as
  well as real scrutiny -- closed with a minimum length bar, then a **distinct-word** bar once padded
  filler (`"looks good looks good..."`) beat the length bar alone, then a same-text-reused-on-a-
  different-requirement check, then a **scaled-by-requirements-claimed** bar once a single review
  turned out to be able to shotgun-approve five unrelated requirements at once with one generic
  paragraph. See [Requirements, verification, and automode]({{ '/explanation/requirements-and-automode/' | relative_url }}).
- A requirement `statement` needed to at least attempt EARS notation ("SHALL" required) -- and once
  that shipped, a plain substring match on "shall" turned out to match inside "shallow" and
  "Marshall," accepting non-attempts by accident. Fixed to require the real word, not a substring.
- A role-filler's own embedded stage proposal could carry a completely empty `stage_id`/`tag`/
  `rationale` and still get applied to the live `PipelineSpec` -- found missing from the HTTP entry
  point, fixed, then found *still* missing from the separate local CLI entry point that mutates the
  exact same state with no HTTP layer involved at all. The real lesson from that one: closing a bug
  at the one call site you tested it against isn't the same as closing the bug class.
- `devsystem.assistant` could be talked into marking a requirement `verified` in a plain chat
  message, based on nothing but the implementer's own self-reported feedback, on any run that hadn't
  declared a `review` role (most runs, by default) -- closed with a real evidentiary gate requiring
  the same evidence bar a human's own review needs, enforced unconditionally for the assistant's own
  calls.
- A cluster of real, live-confirmed **untrusted-content** findings, one leading to the next: role-
  filler-controlled free text (`feedback`, a proposal's `rationale`, a requirement's `statement`)
  could impersonate this project's own real markdown structure in the mandatory check-in artifact
  and the real requirements export -- a crafted statement once forged a completely convincing fake
  "verified, human-authored" entry. The same untrusted text also reaches `devsystem.assistant`'s own
  LLM context on every `/ask` call -- a live prompt-injection test found this particular model
  already resisted a crafted `"SYSTEM OVERRIDE"` payload on its own, but an explicit structural
  defense was added anyway as real defense-in-depth, since the LLM backend is documented as
  swappable.
- Two real, live-confirmed **infrastructure** findings, outside the GUI/API surface entirely: the
  local `devsystem_iterate`/`devsystem_checkin` CLI binaries built filesystem paths straight from a
  raw `run_id` argument with no validation -- `devsystem_iterate ../some-name record.json` wrote
  real files directly into the repo root, completely outside `runs/`. Separately, both binaries that
  persist a real ed25519 signing key wrote it world-readable (confirmed live: mode `664` on the
  actual deployed key) -- anything else able to read arbitrary files on the host could lift it and
  sign fraudulent auction bids under that identity.
- **A real operational gap found by observing the actual deployment, not simulating one click**:
  over a hundred real runs had accumulated -- almost all throwaway scratch/verification runs this
  very stress-test methodology creates on every firing -- with no way to ever remove one. Added
  [a real delete button]({{ '/how-to/delete-a-run/' | relative_url }}), permanent, confirmed, and
  owner-checked the same as every other real destructive action here. Stress-testing that fix's own
  edge cases the same day found the natural follow-on: a run deleted from another tab while still
  open elsewhere used to keep silently showing dead, stale content forever, since a background
  refresh treated a genuine 404 the same as a transient network blip. Fixed to tell you plainly and
  fall back to the runs list instead.
- **Two more real entry points, the same "gone run" bug class**: achieving a milestone through chat
  hits the identical endpoint the GUI checkbox does, but the assistant had zero awareness in its own
  system prompt that doing so pauses the whole run -- it would toggle it and say nothing, the same
  silent surprise run 30 closed for the checkbox. Fixed by telling the model to always disclose the
  consequence in its one-line confirmation (there's no `confirm()` equivalent for a chat action, so
  that's the only real lever). Separately, asking the assistant about a run that no longer exists at
  all used to fall through to a wasted round-trip ending in a confusing wrapped `502`, and unlike the
  background-refresh case, the chat panel never recovered -- fixed to 404 immediately and get the
  same clear alert-and-fall-back recovery. See [Ask devsystem.assistant about your
  run]({{ '/how-to/ask-the-assistant/' | relative_url }}) and [Delete a
  run]({{ '/how-to/delete-a-run/' | relative_url }}).
- `update_criteria` had no upper bound on any `AbortCriteria` field -- a real `u32::MAX` submission
  got a real `200`, turning a run's "bounded super loop" (this project's own central architectural
  claim) unbounded for any practical purpose. A DAU-lens gap, not a role-filler one: a human
  mistyping a value would silently lose their own safety net.
- The 500-item defensive cap on backlog/milestones/requirements was never actually a cap on every
  list that needed one -- `custom_panels` and all four pending-proposal queues (stage, panel-add,
  panel-removal, panel-edit, issue) had no cap at all. Live-confirmed: 510 real custom panels added
  in a row against the actual deployment, zero rejections. Closed by adding the identical check to
  all six remaining entry points; re-verified live by seeding a run to exactly 500 panels and
  confirming the 501st gets a real `400`.
- **Stress-testing a feature the same day it ships, not just the ones already live for a while**:
  real per-requirement chat attribution shipped, computing which requirement an assistant's chat
  reply touched from its own dispatched `Action`s -- but it computed that *before* the real server
  call behind each action resolved success or failure, so a genuine `404` (an out-of-range
  acceptance criterion) still attributed the exchange to a requirement nothing had actually happened
  to. A GUI upload-success message shipped the same session read "Extracted 0 element(s)." for a
  real, successful upload through a path that has no "elements" concept at all -- confusing, not
  wrong, but exactly the kind of thing a careless human reads as "something silently broke."
- **A real cost-exposure gap, and a real regression in its own fix, both found live**:
  `price_ceiling` is never actually enforced against a real bid anywhere in this codebase -- which
  is exactly why the "no price ceiling set" risk exists -- so a real `0` conveys no more protection
  than leaving it unset, but the check only matched "unset." Fixed, then investigating *that* fix
  found something more significant: an assistant-relayed proposal's approval path never touched the
  run's own history at all, so its real price ceiling became permanently unrecoverable the moment it
  was approved -- not just invisible to one check, genuinely lost. Fixing *that* then caused a real
  regression the same day: narrowing the check to only the new, complete-going-forward record
  silently dropped every risk that predated it -- caught by a routine health check against the
  actual flagship run, whose own real, months-old risk had vanished. The honest lesson, stated
  plainly rather than glossed over: fixing a real gap can itself introduce a real regression if the
  fix narrows a check's data source instead of widening it -- the same scrutiny this methodology
  applies to everything else has to apply to its own fixes too. A third finding followed directly
  from testing the fix's own edge cases: a human trying to actually *fix* an already-flagged
  unbounded role the natural way -- re-proposing the same stage with a real price ceiling this time
  -- got a real `200` back, but the fix was silently discarded, because the check always matched the
  *first* proposal recorded for a stage, never a later, better one. The natural human "fix" action
  was a no-op no one would have noticed without deliberately testing it. Fixed so the latest real
  proposal for a stage wins, then re-verified the fix was symmetric (a later proposal *dropping* a
  ceiling correctly re-flags too, not just the reverse).
- A mechanical gap in this project's own quality-bar table, not a runtime bug: "idiomatic to the
  language, not just working" had no real check at all -- `RUSTFLAGS=-D warnings` in CI catches
  compiler warnings, a narrower and different layer than idiomatic-Rust lints. Running `cargo clippy
  --all-targets -- -D warnings` hermetically against both real crates, before adding anything to CI,
  found nine small, real, mechanical findings (a misparsed doc comment, a hand-rolled reimplementation
  of a stdlib method, unnecessary clones, an overly complex type, a `drain`-then-`extend` that should
  have been `append`, three sorts that should have used `sort_by_key`) -- all fixed in the same commit
  that added the gate, watched green in the project's real GitHub Actions run for that exact push, not
  just a local Docker run standing in for it.
- **The "two/three real entry points" pattern, at its most consequential yet**: a `StageProposal`'s
  `units` field was checked for zero at two of its three real entry points (`propose_stage`,
  `quick_submit_offer`), but not for an absurdly large value at either -- live-confirmed:
  `units: 18446744073709551615` (`u64::MAX`) got a real `200`. Investigating whether the third real
  entry point, `validate_proposals` (reached by a role-filler's own embedded stage proposal, which
  applies to the live `PipelineSpec` immediately, with **no human review gate at all**), had even the
  existing zero-check found it had neither check -- the most consequential of the three, since it's
  the one path with no person in the loop to catch it. Fixed at the root: a single `MAX_ROLE_UNITS`
  constant now lives in the pipeline crate as the one source of truth, enforced at all three real
  entry points from the same shared check, not three independently-duplicated ones.

Real, live, currently-true data as of this writing -- the actual `webconference-android` run's own
risks, fetched fresh:

```
$ curl .../api/runs/webconference-android
"risks": [
  {"label": "touches auth/security", "evidence": "iteration 11's feedback mentions \"session\""},
  {"label": "no price ceiling set", "evidence": "role devsystem.document_extraction ... nothing since has bounded what filling it could cost"}
]
```

Two honest, currently-open findings on the actual flagship run, not a synthetic example -- proof this
methodology's own checks fire against real, in-progress work, not just scratch test runs built to
demonstrate them.

## The DAU lens, applied to the GUI

A sample of real, shipped fixes, each verified live with a real Playwright browser against the
actual deployment, not just described:

- Rejecting a pending stage/panel/issue proposal is exactly as permanent as removing a custom panel
  -- but only the removal button asked for confirmation. All three reject buttons now do.
- Removing an indexed RAG document, and marking a memory-log entry "reviewed" (an attestation with no
  undo), had the same gap. Both now confirm.
- The Pipeline chip's own pending-proposal badge -- meant to be the one signal that tells a human
  "something needs your decision" -- silently undercounted for a while: its formula summed three of
  the five real proposal queues, missing panel-removal and panel-edit proposals entirely until a
  stress-test firing caught a real pending removal proposal showing a badge of `0`.
- The "automode" checkbox on a requirement was labeled "let the assistant judge this one," implying
  it gated something real. It never did -- confirmed live, `auto_judge` is never read anywhere in
  `devsystem.assistant`'s own code. The label was corrected to say so plainly rather than keep
  implying a permission model that was never real. See
  [Requirements, verification, and automode]({{ '/explanation/requirements-and-automode/' | relative_url }})
  for the full live verification.
- A genuinely empty run's Requirements panel used to say just "No requirements yet." -- easy to skim
  past, no nudge toward the form sitting right below it. Now a real first-action callout, with the
  statement field auto-focused (guarded so a hidden panel can never silently steal keystrokes).
  Backlog and Milestones had the identical gap and got the identical fix. RAG's version is a banner
  plus auto-expanding its uploads section rather than an auto-focus -- there's no one obvious field
  to focus when the real next action (set a repo, or upload) lives in two different places. Custom
  Panels stayed out of this pattern on purpose: it's an opt-in, power-user feature (writing raw
  HTML), not a core per-run workflow item like the other four, so its existing lighter nudge may
  already be the right amount rather than a real gap.
- The New Iteration form's price-ceiling field only warned about leaving it *blank* -- a careless
  human reading "leave blank for none" could easily type `0` thinking it's a deliberate, conservative
  choice, the opposite of the truth (nothing in this codebase actually enforces `price_ceiling`
  against a real bid, so a real `0` is exactly as unbounded as leaving it empty, and the same
  preflight check flags both identically). Fixed with an explicit label addition and a real `title`
  tooltip stating this plainly on the input itself.
- Checking a milestone's checkbox to mark it achieved fired immediately, with zero warning -- but
  that specific transition auto-pauses the ENTIRE run server-side, blocking every further iteration
  submission until a human explicitly resumes it. A careless click on what looked like a plain
  checkbox had no indication of that real, run-wide consequence. Fixed with a confirm() guarding
  only the achieve direction (un-achieving has no such side effect, so it stays unconditional,
  mirroring the existing un-verify-a-requirement pattern). See
  [Why did my run pause itself?]({{ '/how-to/why-did-my-run-pause/' | relative_url }}).

## The same lens, applied to the flagship app itself

The DAU lens isn't only aimed at this project's own GUI -- it's applied to the real Android client
(`CADS-webconference-android`) built by this pipeline's own role-fillers, too:

- Tapping **Connect** with either field blank used to attempt a real native FFI call and surface
  whatever error the Rust side happened to produce. Now caught locally first, before any native call.
- Tapping **Send** with a blank or whitespace-only message used to silently do nothing --
  `.isEmpty()` alone doesn't catch `"   "`, so a message of just a space would have actually been
  sent as real content.
- A peer disconnecting mid-session used to leave both Connect and Start Listening permanently
  disabled, with no way to reconnect short of restarting the app.
- Tapping **Send** with a real, genuine message *before ever connecting to anyone* used to hit a
  silent early return -- no status update, no rendered message, nothing to tell a user "you're not
  connected" apart from the app looking broken.

All four are real commits in the app's own repository, each with its own Robolectric test proving
the fix, verified green in that repo's real CI on every push -- the same discipline, applied to a
different codebase this pipeline is building, not just the pipeline's own tooling.

## Why this is a page, not just commit messages

Every fix above already has its own detailed writeup somewhere -- a commit message, a section in
another how-to or explanation page, an entry in the project's own internal goal document. This page
exists because the *pattern* connecting them is worth seeing on its own: this project doesn't treat
"someone used it wrong" as a support question. Per the governing principle at the top of this page,
a bad outcome is treated as a real, fixable defect in the process -- found the same honest way each
time, by actually trying to break it, against the real thing, not a mockup.
