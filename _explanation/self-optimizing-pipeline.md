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

The `webconference-android` run started with one role (`plan`). Four real StageProposals landed on
it since, each from a real `devsystem_iterate --remote` submission:

1. `devsystem.android_native_bridge` -- the Android/Rust FFI work needed a real role to auction.
2. `devsystem.document_extraction` -- RAG's document-extraction capability, proposed once the
   architecture decision (auction-discovered agent, not a static API key) was made.
3. `devsystem.android_emulator_test` -- a real gap found while verifying the Android work (nobody
   had actually watched the app run on a device), proposed as its own scoped role rather than
   silently worked around.
4. `devsystem.review` -- forward-looking groundwork so the mandatory review gate (see
   [Requirements, verification, and automode]({{ '/explanation/requirements-and-automode/' | relative_url }}))
   has a real role to actually check requirements against on this run, not just in hermetic tests.

Logged in as a real account, the Pipeline panel shows exactly this -- five real roles, not a mockup:

<figure>
<img src="{{ '/assets/img/explanation-self-optimizing/01-pipeline-panel-with-proposed-stages.png' | relative_url }}" alt="The Pipeline panel for the webconference-android run, showing five roles: plan, android_native_bridge, document_extraction, android_emulator_test, and review">
<figcaption>Four of these five roles didn't exist when the run started -- each was proposed by a real iteration, not configured up front.</figcaption>
</figure>

A pending proposal from `devsystem.assistant`'s own gated path (rather than a role-filler's
immediate one) shows up one level up, too -- the panel toggle bar's Pipeline chip carries a real
badge with the pending count, so it doesn't take opening this panel on spec to notice something is
waiting on a decision. **Real count, all five real gated proposal kinds**: pipeline stage, custom
panel add, custom panel remove, custom panel edit, and GitHub issue -- confirmed live against a run
carrying one of each, the badge reads `5`. Worth naming as its own small history: the badge's first
slice only summed three of these (stage/panel-add/issue); the other two (panel-remove, panel-edit)
shipped later and the badge's own count didn't grow with them until a stress-test firing caught a
real pending removal proposal silently showing a `0` badge -- fixed for real, not just noted, since
this exact signal is what a human is meant to trust to know when to look.

## Declared is not filled

A proposal adding a role to `PipelineSpec` is a real, structural change -- but it doesn't mean
anyone is actually doing that work yet. When this page was first written, `document_extraction` and
`android_emulator_test` above were both real, open, un-won auctions: declared, biddable, and waiting
for a real bidder to `devsystem_offer` on them and start submitting real iterations. A role showing
up here is the pipeline saying "this is now a real thing you can bid on," not "this is done."

**Update, since then**: both roles found a real bidder and did real, verified work --
`android_emulator_test` actually ran two emulator instances and confirmed the run's own M1 milestone
(closed as [issue #13](https://github.com/scimbe/CADS-devsystem/issues/13)); `document_extraction`
shipped a real PDF-extraction handler, merged after independent re-verification (a genuine
end-to-end run against a hand-built PDF, not just trusting the PR), then a second real increment
added real DOCX support via headless `libreoffice --convert-to txt:Text` -- independently
re-verified the same way: a hand-built, valid `.docx` through the actual compiled binary and a real
`libreoffice` install, not just the PR's own claim. `image`/OCR stays honestly unbuilt (no
`tesseract` CLI on the bidder's host, confirmed rather than assumed) -- an unsupported request gets
a real `Error`, never a fabricated extraction. Neither role currently shows a live offer in
`stalled_stages` as of this writing -- and that's the real, important distinction this section
exists to make: **auction liveness and "the work got done" are two separate signals.** A role can
have already delivered real, shipped work and still show as stalled once its bidder's process isn't
actively running `--serve` any more -- stalled means "nobody is bidding on this role *right now*,"
not "this was never done."

**Update, 2026-08-06**: the two real `SignedChannelGrant`s are minted. The blocker really was the
bidder's real full holder public key -- the auction view deliberately only ever shows a 4-byte
display prefix (the section above explains why), and no `AgentCard` for this role was registered in
the control-plane's agent directory to look it up another way. Once the bidder posted their real key
(verified against the auction's own 4-byte prefix before trusting it), the channel id itself was
derived, not guessed: `ct_common::channel::channel_id_for_pipeline_role(operator_pubkey, pipeline_id,
role_tag)` -- the same real, tested, deterministic function `PipelineSpec::role_channel_id` calls
server-side -- run hermetically against this repo's own pinned `ct-common` tag. Two grants followed:
one `accept`-direction for the winning bidder (to actually serve the role's channel), one
`initiate`-direction for a freshly-minted caller identity on devsystem-web's own side (to actually
call it) -- both real private keys persisted to this host's key store at mode 600, never posted
anywhere, the grants themselves (not secrets -- signed authorizations) delivered where each side
needs them.

**Update, 2026-08-06**: `devsystem_document_extraction_client` is genuinely wired into
`web/src/rag.rs` now, not left as caller-side plumbing -- a real upload falls back to this channel
when `RAG_UNSTRUCTURED_API_KEY` isn't configured. See
[Add, remove, and manage RAG documents]({{ '/how-to/manage-rag-documents/' | relative_url }})
for the real, current behavior of both extraction paths and how to tell which one actually ran.

**Correction, 2026-08-06**: the grant and the client wiring above turned out to assume two
different, incompatible connection models -- a real mistake, caught by the winning bidder asking a
clarifying question rather than guessing at which env vars to set. This Agent-Fabric ecosystem has
two genuinely separate ways for two channel members to actually connect:

- **Direct-address** -- `CT_CHANNEL_ADDR`/`CT_CHANNEL_PEER_NOISE_KEY`/`CT_CHANNEL_PEER_CERT`, no
  grant involved at all. What `devsystem_document_extraction_client` (and its sibling
  `github_issue_channel_client`) actually implements -- confirmed by rereading its own source, not
  assumed. Needs a real, dialable address on one side.
- **Broker-mediated** -- `CT_CHANNEL_BROKER`/`CT_CHANNEL_RELAY`/`CT_CHANNEL_GRANT`/
  `CT_CHANNEL_HOLDER_KEY`, the model this host's own already-running `alice` identity actually
  uses. The `SignedChannelGrant`s minted above assumed this model. Empirically confirmed (a real
  run of the actual `ct-agent channel` binary, not just read from source) that this model supports
  a real relay-only mode (`CT_CHANNEL_RELAY_ONLY=1`) for a member with no dialable address at all
  -- exactly the winning bidder's own real constraint (no port forwarding on their dev box).

**Still genuinely open, and now more precisely scoped**: `devsystem_document_extraction_client`
needs a real rewrite to speak the broker-mediated model instead of direct-address -- a different
`ct-agent channel` invocation shape, not a config change. Investigating this further surfaced one
more real prerequisite: `ct-agent channel register` (registering a channel authority with the
control-plane, apparently required before the broker will accept a join for a given channel id)
itself needs a real OIDC bearer token this deployment doesn't currently have provisioned for that
specific purpose. Until both are real, every RAG upload still uses Unstructured or the honest
`503`, same as before this investigation. Handler code being real and merged, a real authorization
existing, and real caller-side wiring existing -- none of that is the same as the role actually
being dialed in production traffic, using the *right protocol*, yet -- another instance of the same
"declared/won is not the same as live-serving" distinction this section already makes for auction
liveness.

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
