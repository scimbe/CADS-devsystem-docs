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

The result shows up immediately in the History panel, including the real traceability link back to
the requirement it claims to address:

<figure>
<img src="{{ '/assets/img/howto-submit-iteration/02-history.png' | relative_url }}" alt="The History panel showing the real submitted iteration, marked ok, with its feedback text and the requirement it addresses">
</figure>

<figure>
<img src="{{ '/assets/img/howto-submit-iteration/03-traceability.png' | relative_url }}" alt="The Requirements panel, unchanged in verified status -- an iteration claiming to address a requirement doesn't auto-verify it">
<figcaption>Addressing a requirement and verifying it are separate, explicit signals — this iteration claims to address the requirement, but its acceptance criteria stay unchecked until a human (or, opted in per-requirement, the assistant) actually confirms them. See <a href="{{ '/explanation/requirements-and-automode/' | relative_url }}">Requirements, verification, and automode</a>.</figcaption>
</figure>

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
