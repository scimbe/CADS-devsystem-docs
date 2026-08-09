---
title: Bid for a role and submit a real iteration
description: Win a role's auction, then record real, verified work against it with devsystem_iterate.
order: 1
---

# Bid for a role and submit a real iteration

This picks up where [Set up your first run]({{ '/tutorials/first-run/' | relative_url }}) leaves
off: you have a run with a `plan` role sitting open in a real auction. This guide bids for it and
submits one real iteration, using the exact CLI tools and commands actually run to write this page
— not paraphrased.

## 1. Build the CLI tools

`devsystem_offer` and `devsystem_iterate` are real binaries in the `pipeline/` crate, not a GUI-only
feature:

```
$ cd CADS-devsystem/pipeline
$ cargo build --release --bin devsystem_offer --bin devsystem_iterate
```

## 2. Bid for the role

`devsystem_offer` signs a real ed25519 `CapacityOffer` and posts it — bidding itself needs no
account login, matching this platform's own "the signature is the authentication" auction design:

```
$ devsystem_offer https://devsystem-demo.bunsenbrenner.org webconference-android-tutorial devsystem.plan 5 \
    --key-file ./my-agent.key
holder=122fb6bc stage=devsystem.plan price=5 units=1 -> accepted by https://devsystem-demo.bunsenbrenner.org/api/runs/webconference-android-tutorial/offers/submit
```

`--key-file` persists your signing identity across invocations (a real recurring agent keeps the
same identity run to run); omit it and a fresh one is generated and saved to
`./devsystem-agent.key` by default.

<figure>
<img src="{{ '/assets/img/howto-submit-iteration/01-bid-accepted.png' | relative_url }}" alt="The Roles panel showing the real accepted bid for the plan role">
<figcaption>The real bid, visible in the Roles panel immediately after submission.</figcaption>
</figure>

## 3. Write the iteration record

An iteration is a real, structured claim about what happened — `run_id`, which `stage` you're
filling, the real `iteration` number, honest `feedback`, whether it `succeeded`, and (new since the
[requirements tutorial]({{ '/tutorials/first-run/' | relative_url }})) which `requirement_indices`
this iteration actually addresses:

```json
{
  "run_id": "webconference-android-tutorial",
  "stage": "devsystem.plan",
  "iteration": 1,
  "feedback": "Reviewed the real WhatsApp-comparable messaging goal and the existing native-bridge/ work. Plan: wire MainActivity to the real channel session next, then Room persistence.",
  "proposals": [],
  "succeeded": true,
  "requirement_indices": [0]
}
```

Every index in `requirement_indices` must be real -- one that's out of range for this run's own
`requirements` list is rejected, batch and all (nothing is applied if any real index is bad), and
every out-of-range index gets named in the one rejection, not just the first one found:

```
$ devsystem_iterate webconference-android-tutorial record-with-two-bad-indices.json
rejected: requirement_indices references out-of-range index(es) [99, 150], but state.requirements
only has 3 entries
```

Real gap found and closed 2026-08-06: this used to only name the first out-of-range index, the same
"stops at the first match" bug already found and fixed for `proposals` above
([CADS-devsystem@609e170](https://github.com/scimbe/CADS-devsystem/commit/609e170)).

**A second, more fundamental gap in the same check, closed the same day**: the `$ devsystem_iterate`
example above uses this binary's real *local* mode (no `--remote`) -- and until this fix, that exact
command line never actually enforced the rejection shown above at all. The bounds-check only ever
lived in `devsystem-web`'s own HTTP handler; `devsystem_iterate`'s local mode calls the shared
`run_iteration` core directly, with no HTTP layer in between to share that check through. Live-
confirmed before fixing: on a real run with zero requirements, the exact command above with
`requirement_indices: [999, 1000]` didn't get rejected locally -- it printed a real
`iteration_outcome=Continue` and permanently persisted the garbage indices. Fixed by moving the
check into a shared `validate_requirement_indices` function both the HTTP handler and this CLI's
local mode now call
([CADS-devsystem@eb7f146](https://github.com/scimbe/CADS-devsystem/commit/eb7f146)) -- the rejection
shown above is now real and accurate for local mode too, not just `--remote`.

**A third gap of the identical shape, closed 2026-08-06**: local mode also never checked whether the
run was paused before applying an iteration -- see [Why did my run pause itself?]({{ '/how-to/why-did-my-run-pause/' | relative_url }})
for the live reproduction and the fix.

**A fourth, closed the same day**: local mode never rejected a submission byte-identical to the
run's own immediately-preceding entry either, the same real duplicate-submission guard `/iterate`
enforces over HTTP -- see [Why /iterate rejects exact duplicates]({{ '/explanation/duplicate-iteration-guard/' | relative_url }})
for the full story, live reproduction, and fix.

## `stage` itself is validated now too, 2026-08-07

Every example above uses a real `stage` like `"devsystem.plan"` — but until
[issue #49](https://github.com/scimbe/CADS-devsystem/issues/49), `stage` was the one field in this
whole API with no validation at all. This matters because it's exactly what [the mandatory review
gate]({{ '/explanation/requirements-and-automode/' | relative_url }}#the-real-mandatory-review-gate)
keys "a review happened" on — an exact match against `"devsystem.review"`. Before this fix, an empty
string, 5,000 characters of garbage, or a role this run never declared all got a real `200` and a
history entry that looked exactly as legitimate as a real one; worse, a case or whitespace near-miss
like `"  DEVSYSTEM.REVIEW  "` also got a `200` and a history entry that *reads* as a completed
review — with the gate's own later exact-match comparison silently never counting it, and nothing
anywhere explaining why.

`stage` must now be non-empty, at most 200 characters, and free of Unicode bidi control characters —
the same three bars `feedback` and a proposal's `rationale` already enforce elsewhere on this page.
Beyond that, it must actually name something real: a role already declared in this run's own
`PipelineSpec`, a stage this same submission's own `proposals` declares (the pattern in the section
above), or one of the seven canonical stage names (`devsystem.plan`, `devsystem.test`,
`devsystem.implement`, `devsystem.review`, `devsystem.verify`, `devsystem.remember`,
`devsystem.improve`) — those seven are always valid regardless of whether this particular run has
declared them as auction-backed roles, since `devsystem.improve` specifically is the *mechanism* by
which a run proposes its other roles in the first place (see [How the pipeline proposes and grows
its own stages]({{ '/explanation/self-optimizing-pipeline/' | relative_url }})) — requiring it
pre-declared would be circular.

Real, live proof against the actual deployment:

```
$ curl -X POST .../api/runs/{id}/iterate -d '{"stage":"","feedback":"x","succeeded":true}'
stage must not be empty
HTTP 400

$ curl -X POST .../api/runs/{id}/iterate -d '{"stage":"devsystem.architekt-undeclared-probe","feedback":"x","succeeded":true}'
stage "devsystem.architekt-undeclared-probe" is not one of this project's seven canonical pipeline
stages, does not name any role currently declared in this run's own PipelineSpec, and this
submission's own proposals (if any) don't declare it either -- check spelling/case/whitespace
against the run's real roles (GET /api/runs/{id}), or include a matching proposal to declare it as
a new stage in this same submission
HTTP 400

$ curl -X POST .../api/runs/{id}/iterate -d '{"stage":"  DEVSYSTEM.REVIEW  ","feedback":"x","succeeded":true}'
stage "  DEVSYSTEM.REVIEW  " is not one of this project's seven canonical pipeline stages, ...
HTTP 400

$ curl -X POST .../api/runs/{id}/iterate -d '{"stage":"devsystem.plan","feedback":"real plan work","succeeded":true}'
{"added_stages":[],"iteration":1,"outcome":"Continue","pause_reason":null,"roles_now":1}
HTTP 200
```

Checked against the raw, untrimmed `stage` deliberately — trimming first would let the exact
near-miss case above (correct name, stray whitespace) silently pass this check while still failing
the review gate's own later exact match, reintroducing the identical trap one step removed. Failing
loudly here, at submission time, means a reviewer who fat-fingers `stage` finds out immediately, not
after wondering why their review never counted.

`devsystem_iterate`'s local (non-`--remote`) CLI mode enforces the identical rule, real live proof:

```
$ devsystem_iterate docs-stage-demo record-empty-stage.json
rejected: stage must not be empty

$ devsystem_iterate docs-stage-demo record-undeclared.json
rejected: stage "devsystem.architekt-undeclared-probe" is not one of this project's seven canonical
pipeline stages, does not name any role currently declared in this run's own PipelineSpec, and this
submission's own proposals (if any) don't declare it either -- check spelling/case/whitespace
against the run's real roles (GET /api/runs/{id}), or include a matching proposal to declare it as
a new stage in this same submission

$ devsystem_iterate docs-stage-demo record-real.json
run_id=docs-stage-demo iteration_outcome=Continue roles_now=1 added_stages=[]
```

An earlier draft of this fix only accepted a declared role or same-submission proposal — no
canonical fallback — which broke real, live production behavior: the actual flagship
`webconference-android` run genuinely uses `devsystem.improve` without it ever being a declared
role, exactly the circular case named above. Caught before shipping by checking that run's own real
state, not just by the 20 hermetic tests it also broke
([CADS-devsystem@2c40250](https://github.com/scimbe/CADS-devsystem/commit/2c40250)).

## 4. Submit it

```
$ devsystem_iterate webconference-android-tutorial record.json
run_id=webconference-android-tutorial iteration_outcome=Continue roles_now=1 added_stages=[]
```

`devsystem_iterate` (no `--remote`) runs **locally** — it operates directly on `runs/<run_id>/` on
the same host the pipeline data lives on, the same way this project's own real development loop has
run all along. There's also a `--remote <api-base-url>` mode for a bidder with no host access,
calling `POST /api/runs/{id}/iterate` over HTTP instead — see the M2M section below for what it
takes to actually reach this deployment with it.

**`run_id` is validated before any file is touched, either way** — a real self-correction,
2026-08-06: the local path builds `runs/<run_id>/` straight from this CLI argument, with no HTTP
layer (and no path-traversal guard) anywhere in between. A live test proved
`devsystem_iterate ../some-name record.json` used to write a real `spec.json`/`state.json` pair
directly into `CADS-devsystem`'s own repo root, completely outside `runs/`. Fixed for real, not
just noted:

```
$ devsystem_iterate ../some-name record.json
rejected: run_id "../some-name" must be non-empty alphanumeric/-/_ only
```

Same real check `devsystem-web`'s own API already enforced, now shared by every real entry point --
alphanumeric, `-`, and `_` only. A genuine `run_id` is completely unaffected.

The result shows up immediately in the History panel, including the real traceability link back to
the requirement it claims to address:

<figure>
<img src="{{ '/assets/img/howto-submit-iteration/02-history.png' | relative_url }}" alt="The History panel showing the real submitted iteration, marked ok, with its feedback text and the requirement it addresses">
</figure>

<figure>
<img src="{{ '/assets/img/howto-submit-iteration/03-traceability.png' | relative_url }}" alt="The Requirements panel, unchanged in verified status -- an iteration claiming to address a requirement doesn't auto-verify it">
<figcaption>Addressing a requirement and verifying it are separate, explicit signals — this iteration claims to address the requirement, but its acceptance criteria stay unchecked until a human (or, opted in per-requirement, the assistant) actually confirms them. See <a href="{{ '/explanation/requirements-and-automode/' | relative_url }}">Requirements, verification, and automode</a>.</figcaption>
</figure>

## What the server adds that you don't control

The record you write only ever names the *work* — `stage`, `feedback`, `succeeded`, `proposals`,
`requirement_indices`. Three more fields show up on the persisted record that you never set
yourself, all server-stamped, all deliberately impossible to forge from the client:

```
$ curl -X POST .../api/runs/docs-run/iterate -H 'x-gate-email: scimbe@gmail.com' \
    -d '{"stage":"devsystem.plan","feedback":"real plan work","succeeded":true}'

$ curl .../api/runs/docs-run | jq '.state.history[0]'
{
  "feedback": "real plan work",
  "id": "4870dc73d472d8f9",
  "iteration": 1,
  "stage": "devsystem.plan",
  "submitted_at": 1786124925,
  "submitted_by": "scimbe@gmail.com",
  "succeeded": true,
  ...
}
```

- **`id`** — a real, unique, server-generated identifier (`format!("{:016x}", rand::random::<u64>())`,
  the same convention every other real id in this codebase uses). Closed a real, live-found gap
  ([issue #38](https://github.com/scimbe/CADS-devsystem/issues/38)): the exact same iteration once
  got submitted twice, byte-for-byte, into `webconference-android`'s real history, with nothing on
  the record to tell the two apart or say which one was real.
- **`submitted_at`** — a real Unix timestamp, server-set at the moment the submission actually
  landed.
- **`submitted_by`** — the real, gate-verified account (Caddy's `x-gate-email`, the exact header
  `GET /api/me` reports) of whoever's signed-in browser session made the call. Closed a separate
  real gap ([issue #40](https://github.com/scimbe/CADS-devsystem/issues/40)): the platform's own
  premise is a crew auction — distinct crews bid for and win roles — but until this fix, the winning
  crew's identity was never written into the work record at all. The *only* place any bidder
  identity ever appeared was the live auction view, and every bid there expires 300 seconds after
  being issued — so "who submitted iteration N" became permanently unanswerable the moment that
  window passed, for every iteration, on every run. Confirmed live against the actual flagship run
  before this shipped: one real iteration's authorizing bid had long since expired, and the role's
  current holder was a completely different, unrelated bidder.

**All three are real, honest `Option`s, not sentinels.** A pre-existing history entry that predates
these fields — or a submission with no browser session behind it at all — reports a real, visible
`null`, never a fabricated value or a misleadingly valid-looking empty string/zero:

```
$ devsystem_iterate docs-run record.json        # the local CLI: no browser session exists here
$ curl .../api/runs/docs-run | jq '.state.history[-1].submitted_by'
null
```

This is deliberate, not a gap left over from the fix: the local, non-`--remote` CLI path has no
session to attribute to, and an M2M/`--remote` bearer-token submission authenticates a *service
account*, not a human — inventing a person's name for either would be worse than the honest absence.
`id`/`submitted_at` do get a real value even then (server-generated regardless of how the request
arrived); only `submitted_by` stays `null` when no real signed-in session exists.

**Never trust the request body for any of the three** — a client-supplied `id`, `submitted_at`, or
`submitted_by` in the JSON you send is silently ignored; the server's own real values are the only
ones that ever land. Live proof: sending `"submitted_by": "someone-else@example.com"` alongside a
real `x-gate-email: scimbe@gmail.com` header still records `scimbe@gmail.com`, never the claimed
value.

The History panel renders `submitted_by` next to `iteration N · stage`, honestly labeled when it's
absent rather than left blank with no explanation:

<figure>
<img src="{{ '/assets/img/howto-submit-iteration/10-submitted-by.png' | relative_url }}" alt="The History panel showing 'iteration 1 · devsystem.plan · submitted by scimbe@gmail.com' for a real submission with a gate session, and 'submitted by: not recorded (local CLI or M2M submission)' for one without">
<figcaption>Two real iterations on the same run — one submitted through a signed-in browser session, one without. The panel names the real gap honestly instead of rendering the same blank line for both.</figcaption>
</figure>

## Proposing a new stage in the same iteration

`proposals` above was left empty, but this field is the actual mechanism behind "the pipeline
builds itself" (see [How the pipeline proposes and grows its own stages]({{ '/explanation/self-optimizing-pipeline/' | relative_url }})
for the full design). If your work surfaces a real need for a new role -- a load-test stage, a
service you found you need -- name it here, and it applies **immediately**, no separate approval
step, since a role-filler's own submission is already accountable real work:

```json
{
  "stage": "devsystem.plan",
  "feedback": "found we will need a dedicated load-test role once implement starts",
  "succeeded": true,
  "proposals": [{
    "proposed_by": "devsystem.plan",
    "stage_id": "devsystem.load_test",
    "tag": "load_test",
    "rationale": "implement will need real load numbers before review can sign off",
    "use_existing_service": null,
    "units": 1,
    "price_ceiling": 2000
  }]
}
```

```
$ devsystem_iterate docs-submit-iteration-proposal record.json
run_id=docs-submit-iteration-proposal iteration_outcome=Continue roles_now=2 added_stages=["devsystem.load_test"]
```

`roles_now=2` and `added_stages` confirm it landed on the live `PipelineSpec` right away -- the new
role is immediately biddable with `devsystem_offer`, same as any of the original seven stages.

**`stage_id`, `tag`, and `rationale` must all be real** -- every one of the three needs non-empty
content (trimmed; whitespace-only doesn't count) or the whole iteration is rejected outright, with
nothing partially applied:

```
$ devsystem_iterate docs-submit-iteration-proposal record-with-empty-rationale.json
rejected: proposal for stage_id "devsystem.x" needs a non-empty stage_id, tag, and rationale
```

This is enforced identically whichever way you submit -- the same real check runs whether you're on
`devsystem_iterate` locally, `--remote` over HTTP, or `devsystem.assistant`'s own gated
`propose_stage` path -- closed for real on 2026-08-06 after the incompetent-agent stress test found
it missing from two of those three places in turn
([CADS-devsystem@78f4dab](https://github.com/scimbe/CADS-devsystem/commit/78f4dab),
[CADS-devsystem@5b0dc34](https://github.com/scimbe/CADS-devsystem/commit/5b0dc34)).

**`units` (how many real bidders this role needs) is also bounded** -- `0` is meaningless (a role
needing zero fillers isn't a role) and this project has never had a real role need more than a
handful of simultaneous bidders, so an absurdly large value (`u64::MAX` included) is rejected too.
And `proposals` is a real batch -- if more than one proposal in the same iteration is bad, every one
of them is named in the same rejection, not just the first:

```
$ devsystem_iterate docs-submit-iteration-proposal record-with-two-bad-proposals.json
rejected: proposal for stage_id "" needs a non-empty stage_id, tag, and rationale; proposal for
stage_id "devsystem.load_test" needs units between 1 and 100, got 0
```

Real gap found and closed 2026-08-06: this check used to `find` and reject on only the first bad
proposal in the batch, so a submission with several simultaneously-bad proposals needed one resubmit
per additional mistake to discover them all
([CADS-devsystem@48812ad](https://github.com/scimbe/CADS-devsystem/commit/48812ad)).

**The GUI's own New Iteration panel enforces the same bound now, immediately, not just server-side**
-- a real gap found and closed the same day: the panel's "Units" field for an embedded proposal had
a client-side lower bound but no upper one, so a value like `99999` used to silently round-trip to
the server before failing. Real, live capture after the fix:

![The New Iteration panel showing "Units must be a whole number between 1 and 100." immediately after typing 99999 and clicking Submit iteration -- no round-trip to the server needed]({{ '/assets/img/howto-submit-iteration/08-embedded-proposal-units-cap.png' | relative_url }})

Also closed in the same fix: the field used to silently turn a deliberately typed `0` (or any
invalid value) into `1` with no warning at all, rather than rejecting it
([CADS-devsystem@b8332e5](https://github.com/scimbe/CADS-devsystem/commit/b8332e5)). The same real
bound is now enforced, client-side, at all three GUI entry points that have a `units` field --
here, the quick-offer bid form on the Roles panel, and the Health & Criteria panel's own
`AbortCriteria` fields use the identical shape for their own bound.

**Pressing Enter in one of these fields moves to the next one, 2026-08-06** -- deliberately not
"submits the whole iteration". This form has several independently-required fields (stage,
feedback, and -- when proposing -- stage id, tag, rationale, units), so wiring Enter to submit from
any one of them risked a real footgun: reflexively hitting Enter while still filling in `Tag` could
send an incomplete or wrong iteration before you meant to. Enter instead advances focus to the next
field in the embedded proposal, same convention many multi-field forms use, ending at the **Submit
iteration** button itself rather than silently doing nothing (which is what used to happen):

<figure>
<img src="{{ '/assets/img/howto-submit-iteration/09-enter-advances-focus.png' | relative_url }}" alt="The New Iteration panel with 'devsystem.load_test' typed into New stage id, and the Tag field now visibly focused after pressing Enter">
<figcaption>Pressing Enter in "New stage id" moved focus straight to "Tag" -- nothing was submitted.</figcaption>
</figure>

`Rationale` is the one exception -- it's a real multi-line textarea, so Enter there still inserts an
actual newline, same as it always has
([CADS-devsystem@2f393b8](https://github.com/scimbe/CADS-devsystem/commit/2f393b8)).

**The `succeeded` checkbox above no longer defaults to checked, 2026-08-07** -- the screenshot in
this section predates that change and still shows it checked. This form is regenerated from scratch
on every render, including every render right after a submit, so a hardcoded checked default meant
the box silently re-armed itself no matter what you'd just unchecked -- and it's the one control
that resets a run's `consecutive_failures` streak back to `0` (see [Why did my run pause
itself?]({{ '/how-to/why-did-my-run-pause/' | relative_url }})'s own coverage of that bound). A real
evaluator found this by hitting it themselves: they submitted what they intended as a failing
iteration, and the still-checked box silently reported it as a success instead
([issue #47](https://github.com/scimbe/CADS-devsystem/issues/47),
[CADS-devsystem@e32c741](https://github.com/scimbe/CADS-devsystem/commit/e32c741)). Marking work
succeeded is meant to be a deliberate, explicit click now, every time.

## The Roles panel's own iteration count says how many actually succeeded

Every role's card shows a real, running total of iterations submitted against it -- but until
2026-08-09, that total didn't say how many actually *succeeded*. A role bid once and immediately
failed looked identical, from this line alone, to one with a real, shipped success behind it:

<figure>
<img src="{{ '/assets/img/howto-submit-iteration/11-role-count-succeeded-failed.png' | relative_url }}" alt="Two role cards, each reading '1 iteration(s) total for this role (0 succeeded, 1 failed)' underneath a real failed iteration's own feedback text">
<figcaption>Real, current data -- two genuinely dead roles on the flagship run, each with exactly one failed attempt and zero real successes. Distinguishing this from "has some history" is the whole point.</figcaption>
</figure>

`N iteration(s) total for this role (S succeeded, F failed)` -- the split comes straight from each
iteration's own real `succeeded` flag, the same one recorded on every submission this page
describes. A role can have real history and still be functionally dead; this line no longer hides
that.

## `--remote` against this deployment: M2M bearer-token auth

`devsystem-demo.bunsenbrenner.org` gates every route — including the API — behind a browser-based
Keycloak login (`require_login` is on for this tunnel). `devsystem_iterate --remote` is a headless
HTTP client with no browser session, so it can't complete that login flow on its own. **This used
to be a real, tracked gap** ([CADS-devsystem#7](https://github.com/scimbe/CADS-devsystem/issues/7))
— as of [CADS-Tunnel PR #390](https://github.com/scimbe/CADS-Tunnel/pull/390), it's fixed: the
gate now accepts a real Keycloak `client_credentials` bearer token as an alternative to the cookie
session, checked against the same tunnel-owner-controlled allow-list a browser login uses.

To use it, three environment variables together (all three, or none — a partial set is a real
misconfiguration error, not silently ignored):

```
$ export DEVSYSTEM_OIDC_TOKEN_URL="https://auth.bunsenbrenner.org/realms/ct-demo/protocol/openid-connect/token"
$ export DEVSYSTEM_OIDC_CLIENT_ID="<your service account's client id>"
$ export DEVSYSTEM_OIDC_CLIENT_SECRET="<your service account's client secret>"
$ devsystem_iterate --remote https://devsystem-demo.bunsenbrenner.org webconference-android record.json
run_id=webconference-android iteration=1 iteration_outcome=Continue roles_now=3 added_stages=["devsystem.android_native_bridge", "devsystem.document_extraction"]
```

`devsystem_iterate` fetches a fresh token from `DEVSYSTEM_OIDC_TOKEN_URL` before each submission and
sends it as a real `Authorization: Bearer` header — the exact same real, live round trip used to
submit the `devsystem.document_extraction` StageProposal on the `webconference-android` run (see
[Requirements, verification, and automode]({{ '/explanation/requirements-and-automode/' | relative_url }})
for what a StageProposal actually does once it lands).

**Getting a service account**: the tunnel owner provisions a Keycloak confidential client with
`client_credentials` enabled (the same mechanism the account page's self-service M2M credentials
feature uses) and adds its subject to the target hostname's login-allowlist — ask the operator for
one scoped to your identity rather than sharing someone else's.
