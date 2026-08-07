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
is a real, live-HTTP script that reproduces forty-nine of the concrete lazy shortcuts below
(one hundred total assertions when a `devsystem.assistant` is reachable locally -- the newest check
gracefully skips, not fails, if one isn't deployed, a genuinely optional dependency; since several
checks prove both the failing case and the genuinely-clear
case in one go -- duplicate `run_id` clobbering, an unbounded/zero `AbortCriteria`, whitespace-only fields, the
"shallow" SHALL-substring bug, an unbounded `price_ceiling` going unflagged (including a later,
bounded re-proposal for the same stage correctly clearing that flag -- the exact mechanism that had
two real regressions earlier this session), cross-account access, a "deleted" run not actually
being gone, `devsystem.assistant`'s own requirement-verification evidentiary gate, a role-filler
forging fake markdown structure in the real requirements export, a proposed GitHub issue targeting
an arbitrary repo outside the real allowlist, a succeeded iteration whose own feedback admits a
known defect, empty/whitespace-only iteration feedback, a run genuinely refusing further iterations
once it hits its own configured bound, the Runs list's own `pending_reviews` count missing two of
five real proposal queues, an empty/whitespace-only `holder_label` when directly accepting a
bid, an absurdly large or zero `units` value at all three real `StageProposal` entry points,
empty/oversized text or an unknown draft id at any of the three real next-step-draft endpoints, a
draft next-step option becoming invisible/orphaned the moment its run is resumed, a requirement
with several simultaneously-bad acceptance criteria only ever reporting the first one instead of
all of them in one response, an iteration's own embedded `proposals` batch only ever naming the
first bad proposal instead of every one of them, an iteration's own `requirement_indices` batch
only ever naming the first out-of-range index instead of every one of them, a custom panel
accepting genuinely empty/whitespace-only HTML at all four real entry points that write it,
backlog item text/milestone descriptions having no real length cap at all, `repo_url` having
the identical gap, a careless re-proposal silently un-bounding an already-set real `price_ceiling`
by omission, a review that once satisfied `no_review_for_succeeded_work` covering unlimited later
succeeded work forever instead of only until the next real change, one early `devsystem.test`
covering unlimited later `devsystem.implement` rounds instead of only the next one, and a real
security-sensitive iteration's own risk flag silently vanishing the moment any unrelated iteration
follows it, a fired but unacknowledged check-in cadence silently resetting to "not due" the instant
it fires instead of staying a real, persistent signal until a human explicitly acknowledges it, and
that same signal never reaching the Open Points panel -- the one endpoint whose entire purpose is
"every real item this run is actually waiting on a human to decide", and a stale Docker build-cache
silently serving a binary that doesn't match this repo's actual real, current source)
against a real running deployment, creating and cleaning up its own real scratch run every
time via the actual `DELETE /api/runs/{id}` endpoint. It's now wired into this project's own real CI
(`pipeline-ci.yml`'s `web` job, confirmed green against a real GitHub Actions run, not just
locally), run against the exact Docker image that gets deployed -- a PR that reintroduces one of
these forty-nine fails CI instead of waiting for the next manual stress-test firing to notice.
Honestly scoped, and
self-correcting: the evidentiary-gate check above was originally left out on the wrong assumption it
needed a real LLM call to test -- a later firing caught that it's actually pure header-based server
logic (`X-Actor: devsystem.assistant`, no LLM involved) and added it for real. What's still
genuinely excluded: prompt-injection resistance and the assistant's own milestone-pause disclosure,
which need a real, non-deterministic LLM reply a human judges, not a fast boolean check. Not every
check here traces back to a live-found gap, either, and this page says so plainly rather than
implying otherwise: the next-step-draft check was added the same day its own feature shipped,
guarding validation that was correct from the start -- a coverage addition, not a fix, and the
**real track record** below counts only the latter.

## Does the harness actually catch a regression, or just pass?

Every check above was written once, against a real live-confirmed bug, and trusted ever since --
never re-verified that it would still genuinely *fail* if that exact gate broke again, as opposed to
passing vacuously for an unrelated reason. [A real engineering piece on harness design](https://www.anthropic.com/engineering/harness-design-long-running-apps)
names this directly: *"every component in a harness encodes an assumption... those assumptions are
worth stress testing."*

Put to a real test, not just read and agreed with: check `[37]` (the byte-identical-resubmission
`409` guard) was mutation-tested by building a throwaway Docker image from a deliberately neutered
`duplicate_of_last_iteration` (hardcoded to always report "not a duplicate"), running it as a fully
isolated container -- a different host port, its own scratch Docker volume, the real production
`devsystem-web` never touched -- and running the real harness against *that*. Result: check `[37]`'s
middle assertion correctly failed (`expected 409, got 200`) while all 71 other checks stayed green,
real proof the check is genuinely load-bearing rather than historically true. The mutated image,
container, and volume were torn down afterward; the source mutation itself was never committed. A
real, reusable technique now, worth applying to more of these checks over time rather than a
one-off.

**A second round, applying the same technique to check `[36]` (the paused-run gate), found a real
gap in the technique's own tooling before it found anything about the check itself.** The first
attempt gave a confusing result: neutering only the paused-check also made the *unrelated* `[37]`
fail. Investigated rather than trusted -- `strings`-checking the deployed binary showed it was
missing `duplicate_of_last_iteration` entirely, a real feature merged days earlier. The real cause:
`web/Dockerfile`'s BuildKit cache mounts are shared by *every* real `docker build -f web/Dockerfile`
on this host, not just real deploys -- an ad-hoc scratch/mutation-test build shares the exact same
cache a real deploy would use, completely outside `deploy-devsystem-web.sh`'s own `flock` (which only
serializes concurrent invocations of *itself*, not unrelated ad-hoc builds). Two plain, sequential
scratch builds -- no concurrency needed -- served a silently stale binary. Fixed at the process
level: a real, prominent comment right at the cache mount declaration stating the actual rule going
forward, any non-deploy build of this Dockerfile must pass `--no-cache`.

**That mitigation turned out to be real but incomplete, found live 2026-08-07**: a later, *regular*
(non-scratch) run of `deploy-devsystem-web.sh` -- not an ad-hoc build -- served a binary that passed
the existing behavioral smoke test (`duplicate_of_last_iteration`) clean while a completely
different, unrelated feature (`checkin_cadence_effectively_disabled`) was silently missing, only
caught by the full stress harness afterward, not the deploy script itself. Chasing individual
behavioral proxies one at a time doesn't scale to every future feature. General fix instead: [`GET
/api/version`]({{ '/reference/rest-api/' | relative_url }}) reports the running binary's own
build-time `git rev-parse HEAD`, baked into the image via a new `web/Dockerfile` build-arg --
`deploy-devsystem-web.sh` now compares it against the real, current source immediately after every
deploy and fails loudly on any mismatch, catching staleness in *any* feature rather than just
whichever one a smoke test happens to exercise. Live-verified twice: once against the pre-commit
source, again after committing -- both times the script printed the running container's real,
correct build SHA.

Rebuilt properly and re-ran the harness: `[37]` now correctly passed (the earlier failure was
conclusively a tooling artifact, not a real gap), and `[36]` failed exactly as intended -- with an
instructive twist. Its own second assertion ("the identical submission succeeds once resumed") also
failed, but for the right reason: the neutered paused-check let the while-paused submission through
for real, so the still-intact idempotency guard correctly caught the "resumed" resubmission as a
genuine duplicate of what had just wrongly landed. Both gates working exactly as designed,
cross-confirming each other in a way the test hadn't originally anticipated.

**The mutation-testing technique, applied to a full batch at once, not just one check at a time**:
the same day four new checks (`[42]`-`[45]`) were added -- one per real staleness-bug fix, see
[How real risk annotations work]({{ '/explanation/risk-annotations/' | relative_url }}) for what
each fix actually was -- every one of the four was mutation-tested individually: temporarily
reverted to its own literal pre-fix code (never a synthetic mutation), confirmed the hermetic unit
test genuinely panics against it, rebuilt and redeployed the mutated binary, confirmed the live
check fails *precisely* on its own claimed regression while every sibling check stays green, then
cleanly reverted and redeployed the real fix. All four passed on both counts -- real teeth, and
precise teeth, not a broader collateral break of the whole check. This closed out the first
complete mutation-verified batch: every check added that day now has real, live proof it would
actually catch its own regression coming back, not just a plausible-sounding assertion.

**The technique kept getting applied as new checks shipped, not treated as a one-time exercise**:
`[46]` (the check-in-pending signal) and `[47]` (its own Open Points entry) each passed the same
real revert-confirm-rebuild-redeploy-reconfirm cycle the day they were added. Applying it to an
*older* check later found something worth naming honestly: neutering `[41]`'s own price-ceiling
enforcement made **two** checks fail, not one -- `[42]`'s final assertion genuinely reuses that
same enforcement line for its own real proof, so both correctly failed together. Not a bug in
either check, just evidence that "one check, one independent regression" isn't always literally
true, worth knowing before trusting a green run to mean every individual check is testing something
wholly separate from its neighbors. One near-miss during this whole sweep, corrected before it cost
anything real: `git checkout --` (meant to revert only the temporary mutation) instead reverted a
file back to its last real *commit*, discarding a just-written, not-yet-committed real fix along
with the mutation -- caught immediately by reading the file's actual content rather than trusting
the command, restored from a backup taken before mutating. Every mutation-test round since has
restored from a pre-mutation backup file instead.

**Three more rounds followed, deliberately picked as safe, non-escalating work while a real
production-deploy decision sat open elsewhere**: check `[39]` (delete-run proposal safety --
neutered the actual `fs::remove_dir_all` call to a realistic "approval reports success but never
deletes" bug), `[40]` (`approve_destroys_panel_title` -- neutered the removal-proposal branch to
"forgot to wire the structured field," the same shape several other real gaps this session found),
and `[38]` (`checkin_cadence_effectively_disabled`, in `pipeline/src/preflight.rs` this time rather
than `web/src/main.rs`, reverted to unconditionally return no finding -- literally "this check never
existed," matching its own doc comment). All three: real teeth confirmed at the hermetic layer, live
harness failed on exactly the targeted check while every sibling stayed green, cleanly reverted from
a pre-mutation backup, real fix redeployed, full suite reconfirmed clean. A growing, not one-off,
list of checks with real, live proof they'd catch their own regression coming back, spanning both
crates this project's stress harness exercises.

**The Docker-cache-staleness incident that motivated all of the above got a real, general fix, not
another one-off behavioral patch**: both `devsystem-web` and `devsystem_assistant` -- two genuinely
separate binaries with two separate real deploy paths -- now bake in their own build-time
`git rev-parse HEAD` and expose it via a real `GET /version`; each deploy script compares the running
process's reported SHA against the actual current source immediately after startup and fails loudly
on any mismatch. Checks `[48]`/`[49]` prove the real, currently-deployed processes report a genuine
40-hex-character SHA, not the honest `"unknown"` fallback that would mean the wiring silently didn't
reach them -- `[49]` gracefully skips, not fails, when no assistant is reachable locally, since it's a
real, optional dependency. And finally, mutation-testing reached all the way back to check `[1]`
(`create_run`'s own duplicate-`run_id` rejection, this harness's literal first check, foundational
but never verified this way before): neutering it to the literal "silently clobbers" pre-fix behavior
failed the hermetic test and the live harness precisely, with every other check unaffected --
confirming even the oldest, most-trusted gate in this file still has real teeth, not just a plausible
green checkmark inherited from the day it was written. The adjacent check `[2]`
(`update_criteria`'s own upper-bound rejection) got the same treatment next: neutering the
`max_iterations`/`max_consecutive_failures`/`checkin_every` cap failed precisely the check's own
second assertion while its sibling lower-bound assertion, a genuinely separate check in the same
source block, correctly stayed green throughout.

Checks `[48]`/`[49]` themselves eventually got the same scrutiny applied to the mechanism that
*proves* every other check's binary is real, not just the checks that mechanism protects. `[48]`
passed: neutering `version()` to always report `"unknown"`, ignoring `DEVSYSTEM_GIT_SHA` entirely,
failed exactly `[48]` on the live harness with every other assertion unaffected. No hermetic unit
test applies to this one by design -- the handler's own doc comment already explains why: mutating a
process-global env var in a multi-threaded test binary would race unpredictably, so the "real SHA is
correctly reported" case is deliberately proven live only, by the deploy script's own post-deploy
check. That deploy-script check turned out to be a second, independent witness to the same mutation:
`deploy-devsystem-web.sh` caught the exact same mismatch and refused to call the deploy verified,
without being told to -- real proof the git-SHA safety net has teeth at two separate layers, not
just the harness. `[49]` (the separate `devsystem_assistant` binary and deploy path) is the one
check in this whole list still without a live mutation-test round.

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

As of this writing, the stress test has run **sixty-four** real rounds against the actual
deployment, finding and closing fifty-one real gaps -- not simulated, not hypothetical. A
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
- **A feature audited for a real gap the same day it shipped, not months later**: a draft
  next-step option (the guided "stack mode" feature's own third slice) rendered nested under the
  Open Points panel's paused-checkpoint card -- but resuming the run removes that entry from the
  queue entirely, and nothing else ever surfaced a leftover draft. Live-confirmed before touching
  anything: a draft added while paused stayed genuinely real in the run's own state after a resume,
  with zero remaining GUI path to see, edit, or delete it -- the same "declared but not accessible"
  bug class this methodology keeps finding, applied here to code barely a day old. Not fixed by
  quietly deleting the draft on resume -- the operator's own explicit ask was that a draft is
  something a human can "delete, change and manipulate," never that resuming should discard one
  without being asked. Fixed at the data source: a leftover draft now surfaces as its own real open
  point once the run isn't paused, alongside the existing nested display while it still is.
- **The same "stops at the first match" bug class, one severity notch down, found by sweeping the
  GUI's own `.find(` call sites once the risk-annotation sweep above was closed out**: `add_requirement`'s
  acceptance-criteria validation (the trivial-content and over-length checks documented above) each
  rejected on the *first* bad criterion in a request and stopped there. Not the "a real risk silently
  persists forever" severity of the risk-annotation bugs -- the caller does eventually learn about
  every bad criterion -- but a real, avoidable retry cost: submitting several simultaneously-bad
  criteria meant fixing and resubmitting once per additional mistake to discover them all. Fixed to
  collect and report every bad criterion from the one request that has them, in a single response.
- **The same bug class, found twice more by continuing the sweep past the GUI into the pipeline
  crate itself, then back into the GUI's own remaining batch fields**: `validate_proposals` (the
  shared gate for a role-filler's own embedded stage proposals, applied immediately with no human
  review) took a real batch but stopped at the first bad proposal in it -- live-confirmed: one
  proposal missing its text fields AND a second with `units: 0` only ever named the first. Fixed
  the identical way. Sweeping back through the GUI's own remaining batch-shaped fields for the
  same shape found one more: `iterate_run`'s own `requirement_indices` (which requirements this
  iteration claims to address) is also a real batch, and its bounds check had the identical
  first-only bug -- live-confirmed `[99, 150]` against a run with zero requirements only ever named
  99. Both fixed and re-verified live the same way as the acceptance-criteria fix above.
- **A real gap found by simply reading the flagship run's own current state, not another sweep**:
  `webconference-android` itself, right now, has `paused: true` with `pause_reason: null` --
  genuinely old data (every real code path that sets `paused = true` today correctly sets a reason
  too, confirmed by reading each), but the GUI's three real renderings of that field all silently
  omitted the reason clause entirely when it's missing, giving zero indication anything was
  missing at all. Fixed to say so honestly ("no reason recorded") instead of quietly showing
  nothing, matching a fallback `open_points()` already had server-side for the identical case.
  Deliberately not backfilled with a guessed historical reason -- an unverified claim, even a
  plausible one, is worse than an honest "we don't know." Pure frontend fix with no automated test
  harness for it in this repo; verified with a real headless-browser screenshot before and after
  against the actual flagship run instead.
- **Found by auditing a feature not yet individually checked, not another `.find()` sweep**: Custom
  Panels -- a real, mutable-content feature -- had the same "every other real free-text field already
  rejects whitespace-only content" gap already found and closed elsewhere in this project, this time
  in its own `html` field. Live-confirmed: `{"title":"x","html":""}` got a real `200`, creating a
  genuinely blank, useless panel with nothing telling you it was empty. The same gap existed at all
  four real entry points that write panel HTML (add, edit, and both of `devsystem.assistant`'s
  gated proposal paths) -- one of those four's own doc comment already claimed its validation
  "mirrors `add_custom_panel` exactly," confirming this was an unintentional gap, not a deliberate
  omission. Fixed at all four with the identical check every other field already uses.
- **A real bound genuinely missing server-side, not just a client-side early-warning gap for one
  that already existed**: checking whether the requirements length-cap fix's own reasoning ("every
  real free-text field has a length cap") actually held everywhere found it didn't -- backlog item
  text and milestone descriptions had no real cap at all, bounded only by the server's generic
  whole-request body limit. Live-confirmed: a real 500,000-character backlog item text got a real
  `200`; only a genuinely oversized (2MB+) request hit the generic limit. Fixed with the same real
  cap (2,000 characters) every sibling free-text field already uses, at both real entry points.
  Continuing the same check found the identical gap in one more real field: `repo_url` also had no
  length cap, live-confirmed with a real 500,000-character URL getting a real `200`. Fixed the same
  way, reusing the same constant -- this appears to close out the "no length bound at all" class:
  every real free-text field in this API now has one.

Real, live, currently-true data as of this writing (2026-08-07; this section has already been
refreshed twice before -- first from an earlier two-finding snapshot once `touches auth/security`
and `no review stage for real, succeeded work` got their staleness fixes, then again after three of
the findings below were genuinely closed) -- the actual `webconference-android` run's own risks,
fetched fresh:

```
$ curl .../api/runs/webconference-android
"risks": [
  {"label": "touches auth/security", "evidence": "iteration 1's feedback mentions \"credential\""},
  {"label": "touches auth/security", "evidence": "iteration 2's feedback mentions \"crypto\""},
  {"label": "touches auth/security", "evidence": "iteration 3's feedback mentions \"auth\""},
  {"label": "touches auth/security", "evidence": "iteration 7's feedback mentions \"session\""},
  {"label": "touches auth/security", "evidence": "iteration 11's feedback mentions \"session\""},
  {"label": "touches auth/security", "evidence": "iteration 12's feedback mentions \"session\""},
  {"label": "touches auth/security", "evidence": "iteration 13's feedback mentions \"auth\""},
  {"label": "no review stage for real, succeeded work", "evidence": "this run has at least one succeeded:true iteration with no substantive devsystem.review iteration since it ..."}
]
```

Eight honest, currently-open findings on the actual flagship run, not a synthetic example -- proof
this methodology's own checks fire against real, in-progress work, not just scratch test runs built
to demonstrate them. Down from eleven: the three `no price ceiling set` findings that used to be
here (`devsystem.document_extraction`, `devsystem.android_emulator_test`, `devsystem.review`) were
genuinely closed the same day via the pipeline's own real "re-propose the same stage with a real
ceiling" fix, not edited out of this example -- proof this list moves in both directions as real
work actually lands, not just upward as new checks get added. See
[How real risk annotations work]({{ '/explanation/risk-annotations/' | relative_url }})
for the real story behind why this list was so much longer for a while: `touches auth/security`
and `no review stage for real, succeeded work` both used to silently lose real findings once anything
unrelated happened afterward -- fixed the same day, and this run's own history is the proof.

## The DAU lens, applied to the GUI

A sample of real, shipped fixes, each verified live with a real Playwright browser against the
actual deployment, not just described:

- Rejecting a pending stage/panel/issue proposal is exactly as permanent as removing a custom panel
  -- but only the removal button asked for confirmation. All three reject buttons now do.
- Removing an indexed RAG document, and marking a memory-log entry "reviewed" (an attestation with no
  undo), had the same gap. Both now confirm.
- **The other direction of the same gap, and the more consequential one**: the Architecture panel's
  own "Approve & post to GitHub" button (an approved `devsystem.assistant` issue proposal) posted a
  real, public GitHub issue to an external repo the instant it was clicked -- zero confirmation,
  even though approving a stage proposal (purely additive to this run's own live spec) correctly
  needs none while this reaches outside the pipeline entirely and isn't meaningfully undoable. Fixed
  with a real `confirm()` naming the actual target repo; live-verified by seeding a real proposal,
  clicking Approve, capturing the real dialog, and confirming a cancel genuinely leaves the proposal
  pending -- proof the old click would have posted for real with zero warning.
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
- **The DAU lens extended to keyboard/screen-reader users, not just careless clicks**: the New
  Project dialog -- the very first control a new user encounters -- turned out to have a genuinely
  severe double gap once actually checked with a keyboard instead of a mouse. Its `autofocus`
  attribute never actually took effect (a real browser quirk: markup inserted via `innerHTML` into
  an already-connected node doesn't reliably trigger the HTML autofocus algorithm), and with no
  focus trap at all, `Tab` from the trigger button walked straight through the *entire page hidden
  behind the overlay* -- live-confirmed reaching requirement input fields a keyboard-only user could
  never see. Fixed with the standard accessible-modal pattern: explicit focus-in, a real
  `Tab`/`Shift+Tab` trap confined to the dialog, and focus restored to the trigger on every close
  path. Checking the same dialog's real accessibility tree (Playwright's `ariaSnapshot`, not a guess
  from the markup) found two more: no `role="dialog"`/`aria-modal` grouping at all, and its own
  `Creating…`/error status line updated purely visually with no `aria-live` region -- a screen
  reader got zero indication anything had happened, success or failure. All three fixed and
  live-verified against the real redeployed production container. See the New Project section of
  [Set up your first run]({{ '/tutorials/first-run/' | relative_url }}) for the real screenshot.
- **The same `aria-live` gap, found sitewide once checked past the one dialog**: every one of this
  app's 14 real status-line elements (Requirements, Backlog, Milestones, Custom Panels, RAG uploads,
  quick-offer bids, fill-mode, criteria, and more) had the identical silent-to-screen-readers gap.
  Fixed uniformly with the same `role="status"`/`aria-live="polite"` pattern. Worth naming plainly:
  the number was first miscounted as 84 by a naive text-occurrence grep that also caught the CSS
  rule and every later class-name reassignment to those same 14 elements -- corrected before
  building anything on top of a wrong number, the same discipline this page's other entries apply to
  the codebase now applied to a mistake in the investigation itself. And mid-fix, an explanatory
  code comment containing literal backtick characters inside a JS template literal threw a real
  `SyntaxError` that blanked the entire GUI -- caught by the same routine post-fix Playwright
  verification pass this methodology always runs, before it ever reached a commit.
- The Flow panel exists for one reason: a fast "where are we" glance, distinct on purpose from the
  dedicated Milestones/Backlog panels, which correctly show full text. But it rendered milestone
  descriptions and backlog item text with zero truncation, unlike `renderProcessPanel`'s own history
  entries just below it in the same file, which already reuse the same `truncate()` helper for
  exactly this reason. Not a hypothetical: a real Playwright screenshot against the actual flagship
  `webconference-android` run showed a wall of untruncated text pushing the panel's own "what
  happened" section fully off-screen, since this project's own real backlog items run to several
  hundred characters. Fixed by reusing the existing 220-character budget `renderProcessPanel`'s own
  feedback preview already uses -- live-verified with a second screenshot against the redeployed
  container, confirming every entry now fits, cleanly truncated with an ellipsis, and "what
  happened" is visible again.

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
- **Found by `devsystem.assistant` itself, not a human sweep**: a received `TextMessage`'s
  `sender_pubkey` was taken verbatim from the wire -- entirely self-reported by whoever sent it,
  never checked against anything the receiving side actually authenticated during the real
  `Noise_IK` handshake. Harmless today only because nothing in the GUI currently trusts this field
  (message direction alone drives rendering), but a forged value would have been silently accepted
  the moment that changes. Fixed on the dialer/initiator side, which already has the peer's
  handshake-pinned key in scope: a wrong key fails the handshake outright, so a live session means
  it's real, not just claimed. The listener/responder side is a real, honestly-named residual gap,
  not silently left unfixed -- closing it needs an upstream change to a separate, pinned dependency
  (`ct_common::a2a::a2a_respond` learns the initiator's key during the handshake but doesn't return
  it), correctly scoped as its own separate increment rather than worked around locally.
- **A real leak, reverted once for host disk-space reasons, shipped for real once that constraint
  was re-examined properly**: `MessageStore` (a real `SQLiteOpenHelper`) kept its real
  `SQLiteDatabase` handle open once first touched, and nothing ever closed it -- every Activity
  recreation (e.g. a rotation) opened a fresh handle to the same on-disk file without releasing the
  previous one. A first attempt at the fix was written, then reverted rather than risk a disk-full
  incident pulling a multi-GB Android SDK image just to test it locally. Re-examined instead of
  left standing: this repo's Kotlin side has never had a *local* hermetic test path at all -- its
  own real GitHub Actions CI is the established verification gate for every one of these fixes, not
  a fallback. Shipped `MainActivity.onDestroy()` calling the real `close()`, with a Robolectric test
  driving the actual production path end to end (captures the live handle, confirms it's open,
  moves the scenario through real teardown, confirms it's closed afterward) -- not declared done
  until the real, deployed GitHub Actions run for that exact commit confirmed green.

All six are real commits in the app's own repository, each with its own real test proving the fix
(five Robolectric, one hermetic `cargo test` -- no Android SDK/NDK needed for pure Rust logic),
verified green in that repo's real CI on every push -- the same discipline, applied to a different
codebase this pipeline is building, not just the pipeline's own tooling.

## Why this is a page, not just commit messages

Every fix above already has its own detailed writeup somewhere -- a commit message, a section in
another how-to or explanation page, an entry in the project's own internal goal document. This page
exists because the *pattern* connecting them is worth seeing on its own: this project doesn't treat
"someone used it wrong" as a support question. Per the governing principle at the top of this page,
a bad outcome is treated as a real, fixable defect in the process -- found the same honest way each
time, by actually trying to break it, against the real thing, not a mockup.
