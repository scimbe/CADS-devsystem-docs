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
