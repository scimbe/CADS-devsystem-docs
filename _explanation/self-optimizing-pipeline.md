---
title: How the pipeline proposes and grows its own stages
description: The real mechanism behind "self-optimizing, not fixed" -- two real StageProposal paths, one immediate, one gated, and why they differ.
order: 1
---

# How the pipeline proposes and grows its own stages

CADS-Tunnel#382's own framing is explicit: this is meant to be a **self-optimizing pipeline, not a
static n8n-style fixed one**. That's not marketing language -- it's a real, specific mechanism, and
this page walks through it against a real run, logged in as a real account, not a mockup.

## Two real paths, one function, different trust boundaries

Every new stage ultimately goes through the same idempotent core:

```rust
pub fn apply_proposal(spec: &mut PipelineSpec, proposal: &StageProposal) -> ProposalOutcome {
    let service = ServiceType::Custom(proposal.stage_id.clone());
    if spec.roles.iter().any(|r| r.service == service) {
        return ProposalOutcome::AlreadyPresent;
    }
    spec.roles.push(RequiredRole { service, units: proposal.units, tag: proposal.tag.clone(), selection_policy: None });
    ProposalOutcome::Added
}
```

But there are genuinely **two different real paths** that call it, gated differently, on purpose:

1. **A role-filler's own iteration.** Any [`devsystem_iterate`]({{ '/how-to/submit-an-iteration/' | relative_url }})
   submission can carry `proposals: [...]` in its body -- every proposal in that list applies to the
   live `PipelineSpec` **immediately, as part of the same iteration that submits it**, no separate
   review step. By explicit operator design: *"these proposals get built into the live PipelineSpec
   (with or without asking the user, per the proposal's own framing)."*
2. **`devsystem.assistant`'s suggestions.** `POST /api/runs/{id}/stages/propose` -- the GUI chat
   assistant's own path -- does **not** apply immediately. It lands in a real pending queue
   (`run_state.pending_stage_proposals`) and needs an explicit
   `POST /api/runs/{id}/stages/proposals/{proposal_id}/approve` (or `/reject`) before
   `apply_proposal` ever runs. Checked directly in `web/src/main.rs`'s `propose_stage` handler, not
   assumed from path (1)'s behavior.

The real distinction is *trust boundary*, not inconsistency: a role-filler's iteration is already
accountable real work happening on an auction it won -- its proposal is "here's what I found I need,"
applied at the same speed as the work itself. `devsystem.assistant`'s proposal is a suggestion from a
conversation, with no won auction backing it -- a human in the loop confirms it before it becomes a
real, biddable role. Same underlying mechanism, different gate, because the caller's real
accountability differs.

## What that looks like on a real run

The `webconference-android` run started with one role (`plan`). Three real StageProposals landed on
it since, each from a real `devsystem_iterate --remote` submission:

1. `devsystem.android_native_bridge` -- the Android/Rust FFI work needed a real role to auction.
2. `devsystem.document_extraction` -- RAG's document-extraction capability, proposed once the
   architecture decision (auction-discovered agent, not a static API key) was made.
3. `devsystem.android_emulator_test` -- a real gap found while verifying the Android work (nobody
   had actually watched the app run on a device), proposed as its own scoped role rather than
   silently worked around.

Logged in as a real account, the Pipeline panel shows exactly this -- four real roles, not a mockup:

<figure>
<img src="{{ '/assets/img/explanation-self-optimizing/01-pipeline-panel-with-proposed-stages.png' | relative_url }}" alt="The Pipeline panel for the webconference-android run, showing four roles: plan, android_native_bridge, document_extraction, and android_emulator_test">
<figcaption>Three of these four roles didn't exist when the run started -- each was proposed by a real iteration, not configured up front.</figcaption>
</figure>

## Declared is not filled

A proposal adding a role to `PipelineSpec` is a real, structural change -- but it doesn't mean
anyone is actually doing that work yet. `document_extraction` and `android_emulator_test` above are
both real, open, un-won auctions as of this writing: declared, biddable, and waiting for a real
bidder to `devsystem_offer` on them and start submitting real iterations. A role showing up here is
the pipeline saying "this is now a real thing you can bid on," not "this is done."

## Why role-filler proposals skip the queue

The operator's own reasoning (stated directly, not inferred): the pipeline is meant to *inform
itself* about what a project actually needs and grow toward that, at real iteration speed -- gating
every role-filler proposal behind human approval would turn "self-optimizing" back into "propose,
then wait," closer to the static pipeline this design explicitly rejects. The real safety net for
*that* path is elsewhere: [`AbortCriteria`]({{ '/how-to/submit-an-iteration/' | relative_url }})
bounds how far any single run can go before a mandatory human check-in, and every proposal's
`rationale` field is a real, human-readable record of *why* -- visible in the run's iteration
history, not hidden. `devsystem.assistant`'s path keeps a real approval gate precisely because it
doesn't have that same accountable-work backing to justify skipping one.
