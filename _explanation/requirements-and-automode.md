---
title: Requirements, verification, and automode
description: Why acceptance-criteria judgment stays human by default, and what opting in actually authorizes.
order: 1
---

# Requirements, verification, and automode

A CADS-devsystem [`Requirement`](https://github.com/scimbe/CADS-devsystem/blob/main/pipeline/src/runner.rs)
is modeled on [EARS notation](https://en.wikipedia.org/wiki/Easy_Approach_to_Requirements_Syntax):
a real trigger, a real system behavior, and a list of concrete `acceptance_criteria` — distinct from
a `Milestone` (a checkpoint) or a `BacklogItem` (a task). It gives both a human reviewer and any
role-filler agent something checkable to verify against, instead of a vague wish.

## What actually counts as a real requirement -- and a real acceptance criterion

**The `statement` field itself, first.** Until 2026-08-05 this endpoint accepted literally any
non-empty string as a "requirement" — a live test proved `{"statement":"asdf",...}` got a real
`200`, despite this whole feature being built around EARS notation. Fixed as a hard gate: a
statement must contain **"SHALL"** (case-insensitive) — the one universal, defining keyword across
every real EARS requirement type. Deliberately doesn't also require `"WHEN"`: a real *ubiquitous*
EARS requirement (no trigger clause at all, e.g. "THE SYSTEM SHALL always encrypt messages at
rest") is legitimate and would be wrongly rejected by a stricter check.

**A real self-correction, 2026-08-06**: the check above was originally a plain substring search
(`.contains("shall")`), which has the exact false-positive shape any raw substring search does — it
matches inside completely unrelated words, not just the real keyword. A live test proved
`"Do a shallow implementation of the login flow for now"` got a real `200` even though it has zero
trigger/behavior clause and isn't remotely EARS-shaped — "shallow" contains "shall" as a substring.
"Marshall"-style false positives have the same shape. Worth stating plainly: this isn't even
necessarily adversarial — an agent genuinely describing scope as "shallow" would accidentally clear
a gate meant to catch exactly this class of non-attempt. Fixed by requiring "shall" as a real,
whole word (split on non-alphanumeric boundaries, case-insensitive) rather than any substring match
— `"SHALL,"`, `"shall."`, and `"shall/could"` still correctly match; `"shallow"`/`"Marshall"` no
longer do.

`POST /api/runs/{id}/requirements` doesn't just check that a criterion is non-empty, either. That
alone turned out to be a real gap: a live test of this exact endpoint (simulating the least
competent realistic role-filler on purpose — see [the goal document](https://github.com/scimbe/CADS-devsystem/blob/main/docs/development-system-goal.md)'s
incompetent-agent stress test) proved `"ok"`, `"."`, `"done"`, and — found along the way, not
planned — a criterion that was **only a zero-width space** (U+200B) all sailed through as real,
checkable criteria. The zero-width case is the more interesting one: `.trim()` only strips Unicode
`White_Space` characters, and U+200B's Unicode category is *Format* (Cf), not *White_Space* — so it
rendered as an apparently blank checkbox line in the GUI while technically passing "not empty".

Every acceptance criterion now needs a minimum count of **alphanumeric** characters, not just
non-empty content — one rule that catches both problems, since an invisible-character-only string
has zero alphanumeric characters under this count:

```
$ curl -X POST .../api/runs/{id}/requirements -d '{"statement": "WHEN a user submits an empty message, THE SYSTEM SHALL reject it", "acceptance_criteria": ["ok"]}'
acceptance criteria: "ok" doesn't have enough real content to be checkable (minimum 5
letters/digits) -- "ok", ".", or an invisible character aren't real acceptance criteria
HTTP 400
```

The bar is deliberately low (5 alphanumeric characters) — this filters non-answers, not real short
criteria. `"no crash"` (7 alphanumeric characters) is accepted without issue.

**A real gap in this same check, found and closed 2026-08-06**: the trivial-content check and the
500-character length cap each used to reject on the *first* bad criterion it found and stop there —
so submitting several simultaneously-bad criteria in one request meant discovering them one at a
time, fixing, resubmitting, and repeating once per remaining mistake. Fixed to report every bad
criterion from the one request that has them, in a single response:

```
$ curl -X POST .../api/runs/{id}/requirements -d '{"statement": "WHEN a user submits an empty message, THE SYSTEM SHALL reject it", "acceptance_criteria": ["ok", "."]}'
acceptance criteria: "ok" doesn't have enough real content to be checkable (minimum 5
letters/digits) -- "ok", ".", or an invisible character aren't real acceptance criteria;
"." doesn't have enough real content to be checkable (minimum 5 letters/digits) --
"ok", ".", or an invisible character aren't real acceptance criteria
HTTP 400
```

Both criteria are named in the one response body — no retry needed to learn about the second.

**A much more consequential member of the same Unicode category, found live 2026-08-06**: the
zero-width space above hides content by being invisible. A **bidi control character** (the
[Trojan Source](https://trojansource.codes/), CVE-2021-42574 attack class) does something worse —
it makes real, visible content *lie about its own order*. Live-confirmed with a real headless
browser before fixing, not assumed: typing a criterion containing a single U+202E (RIGHT-TO-LEFT
OVERRIDE) already displays scrambled in the plain textarea, before ever submitting anything —

<figure>
<img src="{{ '/assets/img/howto-requirements-bidi/01-typed-scrambled.png' | relative_url }}" alt="The Requirements panel's acceptance-criteria textarea showing the literal text 'approvedThis is not test noitcudorp rof' -- the real stored value is 'approved' followed by a right-to-left override character and reversed text">
<figcaption>What's actually typed: "approved" + U+202E + " for production tset ton si sihT". What's displayed: "approvedThis is not test noitcudorp rof".</figcaption>
</figure>

— and the criterion has plenty of real alphanumeric content on both sides of the control
character, so it cleared every other check above untouched. A human reviewer relies on reading a
criterion correctly to decide whether to mark it verified; text whose on-screen order doesn't match
its real content leads a good-faith reviewer to the wrong result through no fault of their own
judgment. Fixed the same way as every other gate on this page — reject it outright, in both the
statement and each criterion
([CADS-devsystem@ed58299](https://github.com/scimbe/CADS-devsystem/commit/ed58299)), then a
same-day follow-up added the matching immediate client-side warning
([CADS-devsystem@9241642](https://github.com/scimbe/CADS-devsystem/commit/9241642)) so this, too,
never needs a server round-trip to discover:

<figure>
<img src="{{ '/assets/img/howto-requirements-bidi/02-rejected-legible.png' | relative_url }}" alt="The same Requirements panel showing a real rejection message: 'approved[bidi-control-char] for production tset ton si sihT contains a Unicode bidi control character -- its displayed text may not match what would actually be stored.'" />
<figcaption>Rejected immediately on submit -- and the control character itself is swapped for a visible placeholder in the message, so the explanation stays legible instead of getting scrambled by the very thing it's warning about.</figcaption>
</figure>

That last detail was itself a real, live find while capturing this exact screenshot: the first
version of the client-side message echoed the offending criterion back verbatim, which meant the
unterminated override character scrambled the rest of the sentence too. Deliberately scoped to just
the two fields a human actually reads and trusts for a review decision (the statement, each
criterion) — milestones, backlog items, and other free text are plausible candidates for the same
class but weren't what this live test touched.

## Two independent, explicit signals

`verified` (the whole requirement) and `verified_criteria` (each individual criterion) are
deliberately kept separate rather than auto-derived from one another. A human confirming "yes, this
whole requirement is done" and a human ticking off individual criteria are two different judgment
calls — coupling them silently would be a real design decision made *for* the operator, not asked
of them.

## `auto_judge`: a real flag that (still) does nothing -- and why the checkbox says so now

The natural next question, once acceptance criteria exist at all, is: can `devsystem.assistant`
judge them itself, instead of a human clicking checkboxes? A requirement's owner can flip a
per-requirement `auto_judge` bit in the API/GUI that looks like it should answer that question.

**A real, live-verified correction, 2026-08-06**: `auto_judge` is not read anywhere in
`devsystem_assistant.rs` -- confirmed directly, not assumed. Three live tests against the real
deployment, identical requirement/evidence shape each time, only the flag's value and the chat
wording varied: asked to "judge and verify if it passes" with `auto_judge: true`, the assistant
genuinely verified the requirement from nothing but the implementer's own feedback text; asked the
same way with `auto_judge` at its default `false`, it declined, citing the same missing evidence.
The flag's value did not predict the outcome -- the LLM's own read of the instruction and the
available evidence did. **The checkbox never gated or unlocked anything.** It used to be labeled
"let the assistant judge this one," implying otherwise; the GUI now says plainly that checking it
changes nothing about what the assistant can already do
([CADS-devsystem@2159a9b](https://github.com/scimbe/CADS-devsystem/commit/2159a9b)) -- the assistant
could always be asked, in a plain chat message, to judge and verify any requirement, flag or no
flag. Deliberately not deleted: it stays a real, honest, persisted placeholder for whatever future
opt-in judgment logic eventually gets built, without pretending that logic already exists.

## The real, mandatory review gate

Every signal above (`verified`, `verified_criteria`, `auto_judge`, `proposed_by`) was, until
2026-08-05, purely advisory: nothing actually stopped anyone (human or role-filler) from marking a
requirement `verified` with zero real review behind it. This project's own
[goal document](https://github.com/scimbe/CADS-devsystem/blob/main/docs/development-system-goal.md)
states the governing principle directly: *"it is the fault of the pipeline, not the user of the
pipeline, if the process leads him not to the perfect result."* A soft annotation a role-filler can
freely ignore doesn't satisfy that — so this is now a real, hard block, not another advisory signal.

**The rule**: if a run's own `PipelineSpec` declares a `review` role (tag `"review"`, service
`Custom("devsystem.review")`), marking one of its requirements verified is rejected with a real
`409 Conflict` unless a `devsystem.review` iteration that `succeeded` and named that exact
requirement (via `requirement_indices`) already exists in the run's history. Un-verifying is always
allowed unconditionally — loosening a claim never needs a review to justify it.

**Scoping, deliberately**: a run that never declares `review` (every new run's default —
`plan_only_spec` has no such role) is never gated at all. There's nothing to hold a run accountable
to a role it never opted into; the gate only bites once a run's own spec says review is part of its
process.

**A qualifying review has to clear two real, mechanical bars, not just exist.** Found live, via this
project's own incompetent-agent stress test, not designed in up front: an early version of this gate
checked only that *a* successful `devsystem.review` iteration named the requirement, and a
one-line rubber-stamp (`"looks fine to me"`) satisfied it just as well as real scrutiny would have.
Fixed with a minimum feedback length (25 characters) -- then, once that shipped, the *next*
realistic lazy move (padded filler well past the length bar: `"looks good looks good looks good
looks good"`) satisfied that too. Both real gaps are closed the same honest way: simple, explainable,
mechanical proxies, not fake LLM-judgment-in-disguise -- **25 characters AND 8 distinct words**
(case-insensitive, punctuation-collapsing, so `"Good! good? GOOD."` still counts as one distinct
word). Neither bar claims to verify the review is actually *good* -- only that it isn't trivially
empty or trivially repetitive. A generic-but-varied review ("looks good, works fine, nothing to
flag, all clear here") would still clear both bars without being real scrutiny either -- a known,
honestly-named, still-open gap, not claimed solved.

**A real self-correction, 2026-08-06**: the two bars above were flat constants, but a single review
iteration can name an arbitrary number of requirements at once via `requirement_indices` -- a live
test proved one generic review, *"Reviewed all of these carefully, checked the real implementation
against each one, everything looks correct and matches expectations on device testing today"* (22
distinct words, comfortably past the flat 8-word bar), satisfied the gate for **five** completely
unrelated requirements at once. Fixed by scaling both bars by how many requirements a given review
claims to cover -- **25 characters × N AND 8 distinct words × N**, where N is that iteration's own
`requirement_indices.len()`. A review naming 3 requirements now needs 75 characters and 24 distinct
words to qualify, real live proof:

```
$ curl -X POST .../api/runs/{id}/iterate -d '{"stage":"devsystem.review","feedback":"Reviewed all of these carefully today","succeeded":true,"requirement_indices":[0,1,2]}'
HTTP 200

$ curl -X POST .../api/runs/{id}/requirements/2/toggle
requirement 2 cannot be marked verified yet -- every devsystem.review iteration addressing
it is too short or too repetitive to plausibly be real scrutiny (best is iteration 2, 37
character(s) and 6 distinct word(s); minimum 75 characters AND 24 distinct words (this
iteration names 3 requirements at once via requirement_indices, so the real bar for it is
3x the usual minimum -- the same real per-requirement bar applies that many times over,
not once for the whole batch)). A rubber-stamp, padded filler, or generic shotgun review
doesn't satisfy this gate.
HTTP 409
```

Single-requirement reviews -- the common case -- are completely unaffected (N=1, same bar as
always). A genuinely thorough multi-requirement review naturally clears the scaled bar, since real
per-requirement observations accumulate real distinct content; a generic "reviewed everything, LGTM"
does not.

Real, live proof against the actual deployment (single-requirement case, unaffected by the above):

```
$ curl -X POST .../api/runs/{id}/requirements/0/toggle   # no review iteration exists yet
requirement 0 cannot be marked verified yet -- no successful devsystem.review iteration
addressing requirement 0 (via its requirement_indices) exists yet. Submit one first.
HTTP 409

$ curl -X POST .../api/runs/{id}/iterate -d '{"stage":"devsystem.review","feedback":"reviewed, approved","succeeded":true,"requirement_indices":[0]}'
HTTP 200

$ curl -X POST .../api/runs/{id}/requirements/0/toggle   # a review exists, but it's too short/repetitive
requirement 0 cannot be marked verified yet -- every devsystem.review iteration addressing
it is too short or too repetitive to plausibly be real scrutiny (best is iteration 1, 18
character(s) and 2 distinct word(s); minimum 25 characters AND 8 distinct words). A
rubber-stamp or padded filler review doesn't satisfy this gate.
HTTP 409

$ curl -X POST .../api/runs/{id}/iterate -d '{"stage":"devsystem.review","feedback":"Checked the actual EditText handling directly against the acceptance criteria, confirmed the real behavior matches.","succeeded":true,"requirement_indices":[0]}'
HTTP 200

$ curl -X POST .../api/runs/{id}/requirements/0/toggle   # now a real, substantive review exists
{"requirements":[{...,"verified":true,...}]}
HTTP 200
```

The GUI surfaces the block as a real error message next to the Requirements panel, and — a related
gap found and fixed alongside the gate itself — reverts the checkbox's own visual state rather than
leaving it looking checked when the toggle was actually rejected.

**This whole gate is only as strong as the integrity of the `stage` string it reads** — until
[issue #49](https://github.com/scimbe/CADS-devsystem/issues/49), that field had no validation at
all, so a case/whitespace near-miss like `"  DEVSYSTEM.REVIEW  "` got a real `200` and a history
entry that *reads* as a completed review while this gate's own exact-match comparison silently
never counted it. See [`stage` itself is validated now
too]({{ '/how-to/submit-an-iteration/' | relative_url }}#stage-itself-is-validated-now-too-2026-08-07)
for the fix and live proof.

## The gate above only ever protected the human-click path -- until 2026-08-06

The mandatory review gate's own scoping is deliberate: a run that never declares `review` is never
gated at all (see above). That was always meant to apply to a *human's* own direct click -- but
`POST /api/runs/{id}/requirements/{index}/toggle` is the exact same endpoint `devsystem.assistant`'s
own `apply_action` calls when it decides, from a plain chat message, to mark a requirement verified.
Nothing server-side could tell the two callers apart, so on the (very common) case of a run that
never declared `review`, the assistant could be talked into verifying a requirement from nothing but
the implementer's own self-report -- exactly the "soft, ignorable, no real review" pattern the gate
above exists to close, just for a caller the gate never covered.

Fixed for real, not just documented as a known gap: `devsystem.assistant` now sends a real
`X-Actor: devsystem.assistant` header on every request it makes, and the toggle endpoint requires
the identical real evidence the mandatory review gate already enforces -- **unconditionally**, when
that header is present and the call would mark a requirement verified, regardless of whether this
run declares `review` at all
([CADS-devsystem@76facaf](https://github.com/scimbe/CADS-devsystem/commit/76facaf)). Un-verifying,
and a human's own direct click (no `X-Actor` header), are both completely unaffected in every case.

Real, live proof against the actual deployment, on a run that never declared `review` at all --
exactly the case the original gate never covered:

```
$ curl -X POST .../api/runs/{id}/requirements/0/toggle           # human click, no review declared
{"requirements":[{...,"verified":true,...}]}
HTTP 200                                                          # succeeds -- by design, unaffected

$ curl -X POST .../api/runs/{id}/requirements/0/toggle           # toggle back off, same human path
{"requirements":[{...,"verified":false,...}]}
HTTP 200

$ curl -X POST .../api/runs/{id}/requirements/0/toggle \
    -H 'X-Actor: devsystem.assistant'                             # the assistant-relayed path
requirement 0 cannot be marked verified yet -- no successful devsystem.review iteration
addressing requirement 0 (via its requirement_indices) exists yet. Submit one first. This
check applies unconditionally to devsystem.assistant-driven verification, regardless of
whether this run declares a review role -- a human's own direct click in the Requirements
panel is not affected.
HTTP 409
```

Same run, same requirement, same missing evidence -- the human's own click succeeds unconditionally
(that scoping decision is untouched), the assistant-relayed call is blocked. Toggling individual
acceptance criteria is deliberately unaffected either way -- that's always been routine bookkeeping,
not the headline `verified` status this gate governs.

## Which requirement is "requirement 0"?

Requirement prose and iteration feedback refer to each other by **ordinal** — a requirement's own
`statement` might read "WHEN an iteration is submitted for requirement 0 (M2 broker-mediated channel
discovery), THE SYSTEM SHALL be validated by...". Until 2026-08-07 that ordinal was never actually
*shown* anywhere in the GUI — it existed only as a `data-index` attribute in the DOM, so resolving a
cross-reference meant opening devtools or counting cards and assuming 0-based numbering
([issue #32](https://github.com/scimbe/CADS-devsystem/issues/32), a real evaluator finding).

Fixed: each Requirements panel card now shows its own real `#N` badge, the identical ordinal the
prose already uses:

<figure>
<img src="{{ '/assets/img/explanation-requirement-ordinals/01-requirements-badges.png' | relative_url }}" alt="A Requirements panel card with a real '#0' badge next to its statement, reading 'WHEN the Android client starts and no manual peer address/public-key has been entered...'">
<figcaption>Real data from the live webconference-android run -- this is requirement #0.</figcaption>
</figure>

The same index is prefixed onto each entry in the New Iteration form's "Addresses" traceability
list, where a truncated statement alone can be ambiguous -- this run has two entries that both
truncate to "WHEN a participant sends a 1:1 message, THE SYSTEM SHALL...":

<figure>
<img src="{{ '/assets/img/explanation-requirement-ordinals/02-newiter-addresses.png' | relative_url }}" alt="The New Iteration form's Addresses checklist, each entry prefixed with a real #N badge, disambiguating two entries that otherwise truncate to the same text">
<figcaption>#3 and #4 are otherwise indistinguishable once truncated -- the badge is the only way to tell them apart.</figcaption>
</figure>

**A real, honestly-named residual gap, not fixed here**: the ordinal is positional. If requirements
are ever reordered or one is deleted, every prose reference to "requirement N" silently retargets —
with the index now visible, that drift is at least *visible* too, but a stable requirement id that
survives reorder/delete would close the gap fully. Left open as a real, separate, bigger question.

## Who wrote this requirement: `proposed_by`

A requirement can come from two real places: a human typing directly into the GUI's Requirements
panel, or `devsystem.assistant` proposing one from a chat exchange (its real `AddRequirement`
action). Until 2026-08-05 there was no way to tell which was which once it landed — both looked
identical in the panel. `proposed_by` closes that gap: `null` means a human wrote it directly;
a stage tag (today, always `"devsystem.assistant"`) means an LLM proposed it and it's still that
LLM's first draft, not yet reviewed. The GUI shows a real **LLM-proposed** badge on any requirement
carrying the tag.

Real, live proof against the actual deployment (not just the hermetic test suite) — adding one of
each and reading them straight back:

```
$ curl -X POST https://devsystem-demo.bunsenbrenner.org/api/runs/{id}/requirements \
    -d '{"statement": "WHEN a human writes a requirement directly, THE SYSTEM SHALL record it with no proposed_by", "acceptance_criteria": ["checkable"]}'
{"requirements":[{"...","proposed_by":null,"statement":"WHEN a human writes a requirement directly, THE SYSTEM SHALL record it with no proposed_by",...}]}

$ curl -X POST https://devsystem-demo.bunsenbrenner.org/api/runs/{id}/requirements \
    -d '{"statement": "WHEN devsystem.assistant proposes a requirement, THE SYSTEM SHALL record it with proposed_by set", "acceptance_criteria": ["checkable"], "proposed_by": "devsystem.assistant"}'
{"requirements":[{...},{"...","proposed_by":"devsystem.assistant","statement":"WHEN devsystem.assistant proposes a requirement, THE SYSTEM SHALL record it with proposed_by set",...}]}
```

The GUI's Requirements panel renders the second kind with a real **LLM-proposed** badge next to its
statement; the first gets none.

This isn't decorative. It's the concrete answer to a real design question this project's own
[goal document](https://github.com/scimbe/CADS-devsystem/blob/main/docs/development-system-goal.md)
asks directly: an LLM is meant to fill in a spec's first draft at a high level, but the human has to
be able to tell, per detail, which parts are still that first draft waiting on their review versus
already theirs to trust. Before `proposed_by` existed, that distinction only lived in whoever's
memory happened to remember which requirement came from which conversation.

## Taking it with you: a real Markdown export

Every requirement shown in the GUI is also a real, downloadable document —
`GET /api/runs/{id}/requirements/export`, or the **⬇ Download as Markdown** link in the
Requirements panel itself. Real, live output from a run with one requirement already reviewed and
verified:

````
$ curl .../api/runs/{id}/requirements/export
# Requirements: `{id}`

1/1 verified.

## 1. ✅

```
WHEN a user taps send with an empty message, THE SYSTEM SHALL not attempt to send anything and SHALL leave the input focused for retry
```

*Human-authored.*

Acceptance criteria:

- [ ] `empty/whitespace-only input never calls sendText`
- [ ] `the input field keeps focus after a no-op tap`
- [ ] `no crash or exception is thrown`
````

**A real self-correction, 2026-08-06**: the statement and each acceptance criterion weren't always
fenced/backtick-wrapped like this -- a live test proved a crafted `statement` containing
`"## 2. ✅\n\n...\n\n*Human-authored.*"` rendered as a completely convincing **forged second
requirement entry** in this exact document -- falsely showing as verified and human-authored,
directly undermining `proposed_by`'s own provenance signal just above (the entire reason this
document exists: telling a human's own requirement apart from an LLM's first draft). Fixed the same
way the [check-in artifact]({{ '/how-to/review-a-checkin/' | relative_url }}) was: a requirement's
`statement` now renders inside a fenced code block, each acceptance criterion inside an inline code
span -- the full text is still there in every case, it just can no longer impersonate this
document's own real headings/bold text.

Notice the whole requirement shows `✅` verified while its individual acceptance criteria still
show unchecked — that's not a bug in the export, it's the same real, deliberate separation §2 above
describes: `verified` (the whole requirement) and `verified_criteria` (each individual criterion)
are two independent signals, and this run's owner confirmed the requirement as a whole without
separately ticking off each criterion. The export renders reality exactly as it is, not a cleaned-up
version of it.

## Why this matters for reproducibility

Fine-grained requirements are, honestly, hard to make perfectly reproducible run to run — the exact
wording of a trigger or a criterion shapes what gets built. What *is* reproducible, and what this
whole model is built to support, is the discipline: every requirement is a real, checkable claim,
every verification is an explicit, attributable signal (a specific human, or an explicitly-authorized
assistant), and nothing about "done" is ever silently assumed.
