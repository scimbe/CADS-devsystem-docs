---
title: How real risk annotations work
description: Mechanical checks over a run's own history and spec -- not an LLM guess, real patterns a human reviewer would otherwise have to spot by hand.
order: 5
---

# How real risk annotations work

`GET /api/runs/{id}`'s `risks` field, and `GET /api/runs`'s `risk_count`, aren't an LLM's opinion —
every entry is a real, mechanical check over a run's own actual history (and, for one dimension,
its own actual spec), traceable back to concrete evidence a human can verify in seconds. This
project's own convention, stated directly in the source: simple, explainable checks, never fake
LLM-judgment-in-disguise.

## Two dimensions, for a real reason

- **History-only checks** (`preflight_annotations`) — a security-relevant keyword in the latest
  iteration's feedback, `devsystem.implement` running before any *substantive* `devsystem.test`
  iteration, a new service proposal with no `price_ceiling`, a `succeeded: true` iteration whose
  own feedback admits a known defect, real succeeded work with no substantive `devsystem.review`
  iteration anywhere in history, and a mandatory check-in cadence that's effectively disabled.
  These only need a run's `RunState` — its iteration history — so they're usable everywhere a run's
  history is available, including `devsystem_checkin`'s own binary (which never loads the run's spec
  at all).

  **"Substantive" is load-bearing, not decorative** — found live by this project's own
  incompetent-agent stress test, 2026-08-05: the test-before-implement check originally only asked
  *whether* a `devsystem.test` record existed, not whether it had any real content. A rubber-stamp
  `feedback: "tests pass"` iteration silently satisfied it exactly as well as real testing would
  have, making the risk annotation vanish even though nothing real had actually been tested. Fixed
  with the same two mechanical bars the review gate uses (see
  [Requirements, verification, and automode]({{ '/explanation/requirements-and-automode/' | relative_url }})) —
  25+ characters and 8+ distinct words. A `devsystem.test` record that doesn't clear both doesn't
  count as real evidence testing happened, and the check falls through to flagging the risk as if
  no test iteration existed at all.

  **Declaring `review` isn't the same as it ever actually happening** — [the goal document](https://github.com/scimbe/CADS-devsystem/blob/main/docs/development-system-goal.md)'s
  own §5 names this gap directly: "a role-filler can mark an iteration `succeeded: true` without
  passing through `review`... at all." `no review stage for real, succeeded work` flags a run with
  at least one real `succeeded: true` iteration but no `devsystem.review` iteration anywhere in
  history substantive enough to count (the identical 25+ character / 8+ distinct word bar as the
  test-stage check above, same rubber-stamp-proofing). Deliberately advisory, not a block — a
  separate, narrower hard `409` already exists for marking a *requirement* verified without real
  review evidence (see [Requirements, verification, and automode]({{ '/explanation/requirements-and-automode/' | relative_url }})),
  but nothing yet stops an *iteration* itself from counting as done the same way. Complementary to,
  not the same as, the process-level "review role declared" check below: that one asks whether
  `devsystem.review` exists in the run's own spec at all; this one asks whether real review
  substance ever actually showed up in history, regardless of whether the role was ever declared —
  a run can pass one and still fail the other.

  **Contradicting yourself doesn't go unnoticed either** — a different flavor of the same
  discipline, this one going after a gap [the goal document's §5 quality-bar table](https://github.com/scimbe/CADS-devsystem/blob/main/docs/development-system-goal.md)
  already named directly rather than one found by simulating a lazy agent: nothing stopped an
  iteration from claiming `succeeded: true` while its own feedback admitted a real, open defect.
  `DEFECT_ADMISSION_PHRASES` -- six specific multi-word phrases ("known issue", "known bug", "not
  fixed", "not implemented", "workaround needed", "still broken") -- flags exactly that
  contradiction. Deliberately narrow: single common words like "broken" alone would false-positive
  on something like "fixes the previously broken X", so the phrases are specific enough to mean what
  they say. Only fires on `succeeded: true` — a FAILED iteration honestly admitting it's broken is
  the behavior this check wants to encourage, not flag.

  **This one stays flagged, deliberately, once it's found** — until 2026-08-06 it only looked at the
  LATEST iteration, the same bug shape as `no_price_ceiling` below. Live-verified: a real, unfixed
  defect admission got correctly flagged, then vanished the moment one unrelated iteration followed
  it, even though nothing was ever fixed. Unlike a price ceiling (a real, checkable field that either
  got set or didn't), there's no structural "was this defect actually fixed" signal in free-text
  feedback -- so instead of scanning for a resolution signal that doesn't exist, this now scans all
  of history and keeps flagging as long as ANY successful iteration ever admitted a defect. The real
  evidence text says so directly: *"No later iteration signals it was ever fixed, so this stays
  flagged."* Named honestly: a defect that genuinely got fixed later, without anyone saying so in
  words this check recognizes, still nags -- a real cost, but a smaller one than silently hiding a
  defect nobody ever said was fixed.

  **A second real gap under that very fix, found 2026-08-06 applying the same lens to
  `no_price_ceiling`'s own "stops at the first match" bug below**: the "stays flagged" fix above
  solved the defect vanishing over time, but the check still only ever returned ONE finding -- the
  single most recent defect-admitting iteration. Live-confirmed: two real iterations, one admitting
  an unfixed session-expiry security gap, the other an unfixed search crash, produced exactly one
  risk finding. The security defect was completely invisible, not because it was ever resolved, but
  because a genuinely different, more recent defect happened to get admitted afterward. Fixed to
  collect every real defect-admitting succeeded iteration, not just the latest.

  **A risk doesn't get to expire just because the conversation moved on** — `no price ceiling set`
  fires when a role needing a brand-new service (`use_existing_service: null`) landed in the run's
  own live spec with no `price_ceiling`, so nothing bounds what filling it could actually cost. Until
  2026-08-06 this only ever looked at the LATEST iteration's own proposals -- found live, the same
  bug shape already fixed for other checks above: propose an unbounded role, get correctly flagged;
  submit one completely unrelated iteration after it, and the exact same still-live, still-unbounded
  role (confirmed still in `state.added_stages`) silently stopped being flagged, even though nothing
  about the real exposure had changed. A human doing periodic check-ins would see a real cost risk
  come and go based on what the most recent iteration happened to talk about, not on whether the
  run's actual state had changed at all. Fixed by scanning all of history for an unbounded proposal
  whose role is still live in `added_stages` -- the real, checkable "is this risk still real" signal.
  A proposal that was rejected, or simply never approved, never entered `added_stages` and is
  correctly never flagged either way -- this doesn't turn into permanent noise for something that was
  actually resolved or discarded.

  **A real `0` isn't a real ceiling either** — `price_ceiling` is never actually enforced against a
  real bid's price anywhere in this codebase (confirmed by reading every real call site, not
  assumed) -- it's stored and shown, never compared against anything, which is exactly why this
  risk exists at all. That makes `price_ceiling: 0` exactly as meaningless as leaving it unset, not
  safer -- but until 2026-08-06 the check only matched `is_none()`, so a role proposed with a real
  `0` ceiling silently produced zero risk findings, a false "this is bounded" signal for a role
  that's exactly as unbounded as one with no ceiling at all. Fixed to treat a real `0` the same as
  `None`.

  **A structurally lost record, found investigating the `0` case** — the fix above still didn't
  fire against a proposal approved through `devsystem.assistant`'s own pending-queue path
  (`POST .../stages/proposals/{id}/approve`). Traced to something more significant than a check bug:
  this risk only ever scanned `history.proposals`, which only a role-filler's own iteration-embedded
  proposals land in — the assistant-relayed approval path never touches `history` at all, so an
  approved proposal's real `price_ceiling` became permanently unrecoverable the moment it was
  approved, not just invisible to this one check. The same two-real-entry-points shape this site's
  own [self-optimizing pipeline]({{ '/explanation/self-optimizing-pipeline/' | relative_url }}) page
  already documents for other checks. Fixed with a new, honest, complete record
  (`RunState.approved_stage_proposals`) both real approval paths now write to, regardless of which
  one a given proposal took.

  **A real regression in that very fix, found the same day** — switching this check to scan *only*
  the new field, instead of adding it, silently dropped every real risk approved before that field
  existed. Caught live, not in a scratch test: a routine read-only health check against the actual
  deployed `webconference-android` run found its own real `devsystem.document_extraction` risk --
  correctly flagged all session -- had vanished. `history.proposals` was still the only real record
  of everything approved before `approved_stage_proposals` existed; this check now scans the union
  of both, not one replacing the other. Named plainly because it's a real, useful lesson about this
  very page's own subject: fixing a real gap can introduce a real regression if the fix narrows a
  check's data source instead of widening it.

  **The natural "fix" didn't actually fix anything, found live-testing the edge cases of the fixes
  above** — a human trying to resolve an unbounded role the obvious way (re-propose the exact same
  role with a real `price_ceiling` this time) got a genuine `200`, since `apply_proposal` correctly
  reports `AlreadyPresent` for a role whose service/tag hasn't changed -- but the check itself kept
  citing the *original* proposal, because it always took the *first* matching record for a given
  role, never the latest. The risk stayed flagged with stale evidence forever, with no way to
  actually clear it through the real proposal mechanism. Fixed on both ends: every real approval
  attempt is recorded now, not just ones that changed the live spec, and the check reads the *last*
  matching record per role instead of the first -- so a later, better proposal genuinely supersedes
  an earlier bad one. Verified this cuts both ways, not just the direction that was found broken: a
  later proposal that *drops* an existing ceiling correctly re-flags the risk too, matching the same
  "current, live state wins" discipline this check already used for staleness elsewhere on this page.

  **A real gap found inside this very check, 2026-08-06**: `no_price_ceiling` used to return at most
  *one* finding -- `Iterator::find` over every real unbounded role in `added_stages`, stopping at the
  first match. Live-confirmed against the actual `webconference-android` run: `devsystem.review` had
  the exact same unbounded shape as the already-flagged `devsystem.document_extraction` (`use_existing_service: null`,
  no `price_ceiling`), but was never surfaced -- the check simply never looked past the first one it
  found. Fixed to collect every real unbounded role, not just the first: re-checking the same real
  run afterward found not two but **three** simultaneously unbounded roles --
  `devsystem.document_extraction`, `devsystem.android_emulator_test`, and `devsystem.review` --
  two of which had been silently invisible the entire time. The exact "a real risk exists but
  nothing surfaces it" pattern this whole page documents, found this time inside one of the checks
  meant to catch it.

  **Not every check could get the same fix, and that's stated plainly, not hidden**:
  `touches auth/security` has the identical "only checks the latest iteration" shape, but a keyword
  mention in some past feedback text has no equivalent checkable "is this still live" entity the way
  a role in `added_stages` does -- there's no honest way to know a security concern was actually
  *resolved* versus just not mentioned again. Left as the latest-iteration-only check it's always
  been, named here as a real, remaining limitation rather than papered over with an invented
  resolution signal.

  **A disabled cadence isn't the same as no risk at all** — `AbortCriteria.checkin_every`'s whole
  documented purpose is a mandatory human check-in that "fires at least this often, even when every
  iteration is succeeding". Found live: `checkin_every: 0` had zero validation anywhere (unlike
  `max_iterations`/`max_consecutive_failures`, which already reject `0`), and it doesn't just make
  check-ins sparse — the underlying `should_checkin` logic falls back to *only* the hard
  `max_iterations` ceiling, so the cadence never fires on its own at all. Worse, the real bug this
  surfaced while investigating: `health.iterations_until_checkin` used to hardcode `0` for this
  case, actively claiming a check-in was **due right now** instead of **disabled** — and since the
  run list's own `needs_attention` flag treats `iterations_until_checkin <= 1` as urgent, this
  permanently false-flagged the run for a reason that was never real. Fixed both: a real risk
  annotation (`checkin_every == 0 || checkin_every >= max_iterations`, since the latter can also
  never fire before the ceiling does), and the health field itself now reports the real distance to
  the actual next check-in event.

  **The opposite extreme, found later the same investigation thread, 2026-08-06**: `0` wasn't the
  only unvalidated edge -- there was no *upper* bound on any of the three `AbortCriteria` fields
  either. A live test proved `{"max_iterations": 4294967295, ...}` (`u32::MAX`) got a real `200`,
  which doesn't just make check-ins sparse the way a large-but-sane value would -- it makes the
  entire "bounded super loop" this project's own architecture is built around unbounded for any
  practical purpose. Fixed with a real, generous ceiling (10,000) on all three fields -- real runs
  here use single- or low-double-digit values, nowhere close to it.

  **A criterion clearing the add-time length gate isn't the same as being specific** — the [goal
  document](https://github.com/scimbe/CADS-devsystem/blob/main/docs/development-system-goal.md)'s
  own §1 commits to "acceptance criteria specific enough to leave no real decision to the LLM."
  `add_requirement`'s own minimum-content gate (5 real alphanumeric characters) already rejects
  the worst cases ("ok", ".", an invisible character) -- but a criterion like `"works"` or `"is
  fast"` clears that bar while still leaving the actual behavior entirely up to the role-filler's
  own judgment. `vague_acceptance_criteria` flags any requirement whose criterion has fewer than 3
  distinct words -- the same crude-but-honestly-scoped proxy discipline as the defect-admission
  phrases above: a genuinely specific-but-terse criterion (`"file exists"`) can still
  false-positive, and a genuinely vague-but-wordy one can still slip through. Not claimed
  comprehensive, same as everywhere else on this page.

  **Only the first one, found 2026-08-06 applying the same lens as the two fixes above** — this
  check had the identical shape: an early `return` inside a nested loop over every requirement's
  every criterion, so it stopped at the first vague one it found. Live-confirmed: two separate
  requirements, each with its own genuinely vague criterion (`"works"`, `"is fast"`), only ever
  produced one finding. Fixed to collect every real vague criterion across every requirement, not
  just the first.

  **`stored text contains a Unicode bidi control character`, added 2026-08-06** — defense-in-depth
  for the [Trojan Source (CVE-2021-42574) bidi-control-character
  fixes]({{ '/explanation/requirements-and-automode/' | relative_url }}) closed at every real
  write-time entry point this same day: requirement statement/criteria, milestones, backlog items,
  custom-panel title, and stage-proposal rationale (both pending and approved). Those fixes only
  guard *new* writes — they can't retroactively clean data already persisted before they shipped,
  and say nothing about a future field that adds free text without remembering the check. A real
  audit of every production `state.json` this repo actually has (110 files) found zero
  contamination, but "audited once and found clean" isn't the same guarantee as "structurally can't
  happen again" — this check scans the same seven fields the write-time fixes cover and surfaces
  any hit here automatically, the same mechanical-check discipline every other finding on this page
  already follows. Verifying it live meant simulating genuinely pre-existing contaminated data,
  since the write-time gates now correctly block new bidi text through the normal API: a bidi-laced
  milestone description written directly into a scratch run's `state.json` (bypassing the API
  entirely) showed up in that run's real `GET /api/runs/{id}` response as this exact finding.
- **Process-level checks** (`process_annotations`, added 2026-08-05) — need the run's own live
  `PipelineSpec` too, since they're about *which roles are declared*, not just what already
  happened. The first one: a run with 3+ real successful iterations that has never declared a
  `devsystem.review` role. Kept as a genuinely separate function rather than folded into
  `preflight_annotations` — extending that one's signature would have broken every existing caller
  that only ever has a bare `RunState`.

## Real, live example — two findings, one run

A real test run: a single `devsystem.implement` iteration, marked `succeeded: true`, whose own
feedback admits an unfixed defect, with no `devsystem.test` iteration ever run and no
`devsystem.review` role ever declared:

```
$ curl .../api/runs/{id}/iterate -d '{"stage":"devsystem.implement","feedback":"Shipped the retry-on-failure feature. Known issue: it crashes on a null message id, not fixed yet, workaround needed before real use.","succeeded":true}'

$ curl .../api/runs/{id}
{
  "risks": [
    {
      "label": "no test stage before implement",
      "evidence": "devsystem.implement first ran at iteration 1, with no devsystem.test iteration before it that's substantive enough to count as real evidence testing happened (25+ characters and 8+ distinct words of feedback, not a rubber-stamp)"
    },
    {
      "label": "succeeded iteration admits a known defect",
      "evidence": "iteration 1's own feedback contains \"known issue\" while marked succeeded:true -- goal doc §5's Vertragsgemäße/Sachmangelfreie row names this exact gap: nothing else blocks marking work \"done\" with open, known defects"
    }
  ]
}
```

(The process-level dimension -- `"no review role declared despite real progress"` -- needs 3+ real
successful iterations to fire, so a single-iteration example like this one never triggers it; see
the comparison right below for what its *absence* looks like on a run that has since declared
`review`.)

**Compare against the real `webconference-android` run, re-checked live, 2026-08-06** — this
example has already gone stale twice, both times corrected here rather than left wrong: first an
earlier version claimed the run shows `"risks": []`; that was fixed once the real two long-standing
risks (`touches auth/security`, `no price ceiling set`) and a genuinely new one were found live. Then
the multi-unbounded-role fix above found the real `no price ceiling set` count itself had been
undercounted the whole time -- the real run currently shows **five** risks, not three:

```
$ curl .../api/runs/webconference-android
{
  "risks": [
    {"label": "touches auth/security", "evidence": "iteration 11's feedback mentions \"session\""},
    {"label": "no review stage for real, succeeded work", "evidence": "this run has at least one succeeded:true iteration, but no devsystem.review iteration anywhere in its history that's substantive enough to count as real evidence review happened (25+ characters and 8+ distinct words of feedback, not a rubber-stamp) -- advisory today, not a block (goal doc §5)"},
    {"label": "no price ceiling set", "evidence": "role `devsystem.document_extraction` is live in this run's own spec, ..."},
    {"label": "no price ceiling set", "evidence": "role `devsystem.android_emulator_test` is live in this run's own spec, ..."},
    {"label": "no price ceiling set", "evidence": "role `devsystem.review` is live in this run's own spec, ..."}
  ]
}
```

(Evidence text truncated above for length -- each real entry is the identical shape, naming its own
real role.) The middle risk is the cleanest real illustration of this page's own "declared isn't the
same as happened" distinction: `devsystem.review` genuinely IS declared in this run's own spec (the
process-level check stays correctly silent), but no substantive `devsystem.review` iteration has
ever actually run in its history -- the history-level check fires anyway, on a completely different
real signal. And `devsystem.review` shows up a SECOND time too, in the last entry -- it's both
declared-but-never-executed AND genuinely cost-unbounded, two different real facts about the same
role, not a contradiction.

## One risk kind now leads you straight to fixing it, not just naming it

Every real risk finding above was, until 2026-08-07, rendered in the GUI as plain text — a real,
mechanical finding, but the human still had to know on their own which panel to open and which
field to change. Real DAU-lens gap, found live the same day it was closed: the Risks & Stalled
panel's own sibling section (stalled stages) already gives a one-click fix (a button that fills in
the iteration form with the stalled stage), but risk findings never did.

Not generalized to all twelve risk kinds — eleven of them genuinely need human judgment (a vague
acceptance criterion, an admitted defect, a change touching auth/security) and an automatic "fix
it" button for those would be dishonest, not helpful. `mandatory check-in cadence effectively
disabled` is the one exception: a run-level setting with a single, unambiguous, always-safe fix, no
per-role or per-stage targeting needed. That one now gets a real **Fix it →** button right next to
its finding:

<figure>
<img src="{{ '/assets/img/explanation-risk-annotations/01-risk-before.png' | relative_url }}" alt="The Risks panel showing the check-in-cadence risk, with a real 'Fix it →' button next to it">
<figcaption>A real scratch run with <code>checkin_every: 0</code> — the exact trigger condition above — showing the finding with its new button.</figcaption>
</figure>

<figure>
<img src="{{ '/assets/img/explanation-risk-annotations/02-fix-it-clicked.png' | relative_url }}" alt="After clicking Fix it: the Health & Criteria panel is open, its criteria details expanded, and the check-in-every field focused and selected">
<figcaption>One click later: the Health & Criteria panel is genuinely open, its collapsed details expanded, and <code>check-in every N iterations</code> focused and selected — ready to type a real value into, not auto-filled for you.</figcaption>
</figure>

It never auto-submits a value on your behalf — same "flag it, then let the human act" restraint
`saveCriteria`'s own client-side bounds-check already applies in the other direction (see [Ask
devsystem.assistant about your run]({{ '/how-to/ask-the-assistant/' | relative_url }}) for that
side of the same discipline). Live-verified via Playwright against the real deployed container,
not assumed from source: `document.activeElement.id` was checked directly after the click and
confirmed to be the real `cr-checkin-every` input
([CADS-devsystem@e9e075c](https://github.com/scimbe/CADS-devsystem/commit/e9e075c)).

## Why "3+ successful iterations", not a specific stage name

The process-level check deliberately doesn't look for `devsystem.implement` by name. This
pipeline's own roles are custom-named per project — `webconference-android` itself never has a
literal `devsystem.implement` stage, only project-specific ones like
`devsystem.android_native_bridge`. Counting real *successful* iterations (`succeeded: true`) is the
honest, general signal available across every run, regardless of what its stages happen to be
called — and only successful ones count, since a string of failed attempts isn't the "real
progress" this check is actually about.
