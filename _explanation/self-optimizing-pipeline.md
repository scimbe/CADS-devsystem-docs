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
a real `Error`, never a fabricated extraction. At the time this was first written, neither role
showed up in `stalled_stages` -- and that's the real, important distinction this section exists to
make: **auction liveness and "the work got done" are two separate signals.** A role can have
already delivered real, shipped work and still show as stalled once its bidder's process isn't
actively running `--serve` any more -- stalled means "nobody is bidding on this role *right now*,"
not "this was never done."

**Re-checked live, 2026-08-06**: exactly what that distinction predicts has since happened --
`stalled_stages` on the real flagship run now reads
`["devsystem.document_extraction", "devsystem.android_emulator_test", "devsystem.review"]`, both
roles back in the list. Neither bidder's `--serve` process is running right now. That's not a
regression or a sign the earlier work was undone -- the real APK build, the real PDF/DOCX
extraction handler, and the real M1 milestone confirmation (two actual Android emulator instances
exchanging a real message, see [issue #13](https://github.com/scimbe/CADS-devsystem/issues/13))
all still genuinely happened and are still real. It's the exact case this section's own reasoning
was written to cover, now observed rather than only described.

**Correction, 2026-08-07 -- the signal's own real meaning changed underneath this section**: what
"stalled" actually keys on was, until [issue #53](https://github.com/scimbe/CADS-devsystem/issues/53),
subtly different from what the paragraphs above describe. It was never a live/right-now bidding
check at all -- there's no time window, no polling of whether a `--serve` process is currently up.
It was permanent, binary, and keyed on one thing: does *any* iteration record -- succeeded or not --
exist for this stage in the run's own history. A real evaluator (the `bastler` persona) found the
consequence live on this exact flagship run: `devsystem.document_extraction`,
`devsystem.android_emulator_test`, and a third role, `devsystem.android_native_build_ci`, had each
had exactly one iteration recorded against them, every one `succeeded: false` -- two of the three
opening with "drive the stalled `<stage>` stage" and describing what *would* need to be built, not
reporting that it had been. A failed attempt satisfied the old check identically to a real delivery,
permanently, with no way to re-arm it.

Fixed to key on "has a *succeeded* iteration ever run as this stage" instead. Practically, for this
page's own running example: nothing changes for the two roles discussed above, since their real
bidders' work genuinely succeeded, then the bidder processes stopped running -- `stalled_stages` on
this run today, after the fix,
reads `["devsystem.document_extraction", "devsystem.android_emulator_test", "devsystem.android_native_build_ci"]`
-- the third role above is now correctly named too, which the old, looser reading of this section
would have missed. The distinction this section exists to make is unchanged and, if anything, sharper
now: "delivered, then went idle" and "never once delivered" both surface as stalled today, correctly
-- but only the fix makes that second case actually true rather than an accident of no record ever
having been attempted.

**Update, 2026-08-07**: `devsystem.android_native_bridge` -- the very first of the four proposed
roles above, and the one this page's own list names but had never followed up on -- delivered real,
verified work too, the identical "declared is not filled, filled is not the same as delivered"
pattern this section already traces for the other two roles. A real iteration (21) landed real per-
message delivery status in the actual Kotlin app: `SENT`/`FAILED`, honestly scoped to what this
run's own direct, unacknowledged Noise_IK channel can actually prove (no acknowledgement protocol
above the transport layer exists yet, so "delivered"/"read" would have to be fabricated rather than
verified -- deliberately left for separate, later work instead)
([CADS-webconference-android@7e325e6](https://github.com/scimbe/CADS-webconference-android/commit/7e325e6)).
Submitting it as a real iteration, not a side-channel commit, mattered here for a reason this page's
own mechanism explains directly: it's what correctly reset the run's `consecutive_failures` back to
`0` (two real, honest failures from other stages had already landed since) and correctly triggered
the run's own mandatory check-in pause -- the same real bounded-loop enforcement [Why did my run
pause itself?]({{ '/how-to/why-did-my-run-pause/' | relative_url }}) documents, observed here firing
on the actual flagship run rather than only in a hermetic test.

**Update, 2026-08-07 -- the run's own review/merge gate closed a real panic, not just proposed a
fix**: iteration 27 above delivered real work but honestly stopped short of shipping it --
[`CADS-webconference-android` PR #11](https://github.com/scimbe/CADS-webconference-android/pull/11)
sat open, CI-green, unmerged, because a role-filler's own iteration can submit a PR but has no
merge authority over the target repo. Closing that gap is a distinct real action from the auction
itself: a real code review (not a rubber stamp) of the actual diff, independent
confirmation the root cause was correctly diagnosed (`hex_decode_32` gated on `str::len()`, a BYTE
count, then indexed the same `&str` by byte range -- a 64-BYTE peer key holding any multi-byte
UTF-8 character cleared the length check and then panicked on a non-char-boundary slice), and
direct confirmation of both CI jobs independently -- not just trusted from a green checkmark. The
`verify-native-bridge` job's own log was read directly: a fresh NDK rebuild's exported symbols
matched the committed `.so` exactly, 105 symbols on each of `arm64-v8a`/`x86_64`, proving the
binary the APK actually ships contains the fix, not just the Rust source. Merged as
[`ff3864d`](https://github.com/scimbe/CADS-webconference-android/commit/ff3864d), both CI jobs
re-confirmed green on the resulting `main`.

Requirement #17's own criteria (live-checked against the real run, not assumed) stay honestly,
deliberately partial after this: criteria 1-3 -- the hermetic regression test itself, the
round-trip proof the fix isn't over-restrictive, and `onConnectClicked`'s real typed-error catch --
are confirmed. Criterion 0's own text asks for the exact repro pasted into the peer-key field with a
real tap on Connect; code-level and binary-level proof isn't the same claim as a real device
interaction, so it stays unconfirmed rather than being marked done on the strength of a unit test.
Criterion 4 (an automated emulator test) remains what it always was: real, open work belonging to
`devsystem.android_emulator_test`, a separate stalled role this merge deliberately didn't touch.

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

**Update, 2026-08-06**: `devsystem_document_extraction_client` is rewritten for real to speak the
broker-mediated, relay-only model -- `CT_CHANNEL_BROKER`/`CT_CHANNEL_RELAY`/`CT_CHANNEL_GRANT`/
`CT_CHANNEL_HOLDER_KEY`/`CT_CHANNEL_NOISE_KEY`, with `CT_CHANNEL_RELAY_ONLY=1` hardcoded rather
than configurable -- this deployment's own caller identity has no dialable public address either,
so relay-only isn't a choice, it's the only mode that was ever going to be correct here.

**A second real gap found alongside the rewrite, before it ever reached production**: the client
binary was never actually built into `web/Dockerfile` at all -- its sibling
`github_issue_channel_client` was, this one wasn't. The wiring and hermetic tests were real, but a
live deployment would have hit a real "No such file or directory" the moment every
`DOCUMENT_EXTRACTION_*` env var was finally configured, since the binary genuinely didn't exist in
the container. Fixed with the same build-and-copy shape already established for its sibling.
Confirmed live in the actual running container, not assumed from the Dockerfile diff: the binary
exists at `/app/devsystem_document_extraction_client` and runs.

**Still genuinely open**: `ct-agent channel register`'s own OIDC bearer-token prerequisite --
registering a channel authority with the control-plane, apparently required before the broker will
accept a join for a given channel id. This needs a real Keycloak admin action this deployment
doesn't have access to provision itself, and won't self-provision without the operator's own
go-ahead regardless (new production auth credentials are a real, hard-to-reverse action on shared
infrastructure, not a call to make alone). Raised directly with the operator on
[CADS-Tunnel#382](https://github.com/scimbe/CADS-Tunnel/issues/382). Until it's resolved, every RAG
upload still uses Unstructured or the honest `503`, same as before this investigation -- protocol,
grant, and binary presence are all real and ready; only this one credential is missing. Handler
code being real and merged, a real authorization existing, real caller-side wiring speaking the
right protocol, and the binary now genuinely present in the deployed image -- none of that is the
same as the role actually being dialed in production traffic yet -- another instance of the same
"declared/won is not the same as live-serving" distinction this section already makes for auction
liveness.

**Update, 2026-08-06**: a third real increment on the handler side, reviewed and merged the same
way as the DOCX increment above -- real hands-on verification, not a rubber stamp (isolated
worktree, the hermetic suite run independently, and the freshly-built binary actually executed
against real crafted requests, not just trusted from the PR's own description). Added real plain
`text/plain`/`text/markdown` support (pure UTF-8 decode, no subprocess at all -- there's nothing to
convert) and legacy **`.doc`** support, reusing the exact same `libreoffice` conversion path DOCX
already uses. See [Add, remove, and manage RAG documents]({{ '/how-to/manage-rag-documents/' |
relative_url }}) for the current, full list of what an upload actually accepts.

**A real gap found stress-testing that increment end to end, not just at the handler**: the handler
side gained `.doc` support, but `devsystem_document_extraction_client` -- the caller that actually
determines the `mime_type` sent over the wire from a file's extension -- was never updated to
recognize `.doc` at all. A real `.doc` file would have fallen through to
`application/octet-stream` and been rejected by the handler as unsupported, even though the handler
genuinely now supports it -- the exact "fixed at one end, not the whole real path" pattern this
project's own stress-test methodology keeps finding elsewhere in this codebase. Fixed the same day.

**Still genuinely open**: the same OIDC bearer-token blocker above -- none of this format work
changes that a live upload through this channel still needs the operator's own Keycloak action
first.

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

## `rationale` has to be trustworthy to read, not just present

The paragraph above only holds if `rationale` genuinely says what it appears to say. A real gap
closed 2026-08-06, closing out the last candidate of a [Trojan Source
(CVE-2021-42574)]({{ '/explanation/requirements-and-automode/' | relative_url }})
bidi-control-character sweep that started with requirement text and had already spread to
milestones, backlog items, and custom-panel titles: a rationale laced with a real right-to-left
override character used to sail through untouched at both real entry points, visually rendering as
something like "Needed for real testing" while what was actually stored -- and what a reviewer
would see if the text rendered honestly -- admitted "This is a dangerous stage, exposes actual data
extraction." Text whose on-screen order doesn't match its real content defeats the entire point of
`rationale` existing as a real, readable safety net; a role-filler proposal reaching the embedded
path (skipping the review queue, per the design above) is exactly the case with the least room for
that kind of deception to go unnoticed.

Fixed at both real entry points -- the embedded/immediate path (no queue to catch it) and
`devsystem.assistant`'s own pending-review `propose_stage` path (where a human approving from the
queue trusts this exact text) -- with the identical rule already documented for requirements,
milestones, and panel titles:

```
$ curl -X POST .../api/runs/{id}/stages/propose -d '{"stage_id":"devsystem.load_test",
    "tag":"load_test","rationale":"Needed for real testing‮ ..."}'
rationale contains a Unicode bidi control character (e.g. a right-to-left override) -- these
can make the visually displayed text not match what's actually stored
HTTP 400
```

With this fix, the bidi-control-character class is closed across every real free-text field a human
reads and trusts for a decision in this codebase -- requirement statement/criteria, milestones,
backlog items, custom-panel title, and now stage-proposal rationale. Custom-panel `html` stays
deliberately untouched: it's rendered only inside a sandboxed iframe, untrusted-by-design, so the
same rule there would contradict its own existing security model rather than close a gap.
