---
title: How the pipeline proposes and grows its own stages
description: The real mechanism behind "self-optimizing, not fixed" -- StageProposal, applied immediately, no approval gate.
order: 1
---

# How the pipeline proposes and grows its own stages

CADS-Tunnel#382's own framing is explicit: this is meant to be a **self-optimizing pipeline, not a
static n8n-style fixed one**. That's not marketing language -- it's a real, specific mechanism, and
this page walks through it against a real run, logged in as a real account, not a mockup.

## The real rule: a proposal applies immediately, no human approval gate

This is worth stating plainly because it's easy to assume otherwise (a GitHub-issue proposal
elsewhere in this system *does* go through a pending/approve/reject queue -- StageProposals don't):

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

Any [`devsystem_iterate`]({{ '/how-to/submit-an-iteration/' | relative_url }}) submission can carry
`proposals: [...]` in its body. Every proposal in that list is applied to the live `PipelineSpec`
**as part of the same iteration that submits it** -- idempotently (`AlreadyPresent` if the stage
already exists, so a repeated proposal is a safe no-op, not a duplicate role). There is no separate
review step for this, by explicit operator design: *"these proposals get built into the live
PipelineSpec (with or without asking the user, per the proposal's own framing)."*

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

## Why this design, not a review queue

The operator's own reasoning (stated directly, not inferred): the pipeline is meant to *inform
itself* about what a project actually needs and grow toward that, at real iteration speed -- a human
approval gate on every proposed stage would turn "self-optimizing" back into "propose, then wait,"
which is closer to the static pipeline this design explicitly rejects. The real safety net is
elsewhere: [`AbortCriteria`]({{ '/how-to/submit-an-iteration/' | relative_url }}) bounds how far any
single run can go before a mandatory human check-in, and every proposal's `rationale` field is a
real, human-readable record of *why* -- visible in the run's iteration history, not hidden.
