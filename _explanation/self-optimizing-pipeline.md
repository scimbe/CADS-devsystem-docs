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

## The inverse: a channel for questions the pipeline cannot answer

Every path above is "the pipeline wants to do something and needs signed off." A run can just as
easily hit the opposite shape: a role-filler reaches a genuine product question it has no standing
to decide -- not a technical proposal, a real fork in what the project should even do -- and until
2026-08-10 there was no structured channel for that at all. It degraded to prose: an iteration would
write something like `"OPERATOR DECISION NEEDED (escalated by iteration 19, devsystem.plan): should
this project ever support offline/store-and-forward delivery?"` into a plain backlog item, because a
backlog item (`{done: bool, text: string}`) was the only container available. Nothing indexed it,
nothing summarized it, nothing waited for it -- a real evaluator's finding, [issue
#39](https://github.com/scimbe/CADS-devsystem/issues/39).

`RunState.pending_decisions` is the structured inverse of `pending_stage_proposals` and its five
proposal-queue siblings: `POST /api/runs/{id}/decisions` raises a real question (`{"question": "...",
"options": [...]}`, `options` optional), at the *same* trust level as a role-filler's own
iteration-embedded `StageProposal` above -- no owner-gate, applies immediately, because asking a
question changes nothing about the run's real state, it only makes a real gap visible. `POST
/api/runs/{id}/decisions/{decision_id}/answer` is the operator's own real answer, gated like
`checkin/acknowledge`, recorded exactly once (a `400`, not a silent overwrite, on a second attempt).

An unanswered decision is a real [Open Point]({{ '/how-to/work-through-open-points/' | relative_url }})
-- kind `pending_decision`, with its own input field and Answer button, not just a proposal's
Approve/Reject -- and the mandatory [check-in document]({{ '/how-to/review-a-checkin/' | relative_url }})
now enumerates every real open question by name in its own "Decision needed" section, instead of the
static boilerplate that used to sit there regardless of what the run actually needed. See [REST API
reference: Decisions]({{ '/reference/rest-api/#decisions' | relative_url }}) for the full shape.

Deliberately scoped, not the whole ask: [issue #39](https://github.com/scimbe/CADS-devsystem/issues/39)'s
own suggested gating policy -- a run should not be allowed to burn its final iteration, or otherwise
proceed, with a blocking question still unanswered -- is real, separate work, not attempted in this
same increment. The channel exists and is fully visible; nothing yet *stops* a run for an unanswered
decision the way a paused checkpoint does.

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
`libreoffice` install, not just the PR's own claim. `image`/OCR stayed honestly unbuilt at the time
(no `tesseract` CLI on the bidder's host, confirmed rather than assumed) -- an unsupported request
got a real `Error`, never a fabricated extraction.

**Correction, 2026-08-09 -- the image/OCR gap above is closed, real and independently verified,
not merely merged**: the blocker in the paragraph above was wrong about *why*, not merely
outdated -- *installing* `tesseract-ocr`'s CLI needs root, but *obtaining* it does not (`apt-get
download` + `dpkg -x` into a userspace prefix), and `libtesseract5` was already present. Reviewed
the same way every real external contribution here is (not a rubber stamp): re-ran the full 23/23
test suite in an isolated worktree after a clean rebuild, and built the real compiled binary myself
-- fed it a real, freshly generated PNG (via ImageMagick, not a fixture) and got back the exact text
that image actually contained. `image/svg+xml` stays honestly unsupported (leptonica, the OCR
library this path uses, does not rasterize SVG). Merged as
[CADS-devsystem@c1e09f4](https://github.com/scimbe/CADS-devsystem/commit/c1e09f4). At the time this
page was first written, neither role
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

**Update, 2026-08-09 -- a real defect no existing gate could see, found by measuring the actual
published artifact rather than trusting a green build**: iteration 31 downloaded `app-debug.apk`
from a real CI run and looked inside it, rather than trusting `assembleDebug`'s own exit code.
It packaged seven ABI directories -- `arm64-v8a`, `armeabi`, `armeabi-v7a`, `mips`, `mips64`, `x86`,
`x86_64` -- while `app/src/main/jniLibs/` ships `libnative_bridge.so` for exactly two of them. The
other five carried nothing but the JNA `@aar` dependency's own `libjnidispatch.so`: real bytes that
can never do any work. `armeabi-v7a` sits squarely inside this app's `minSdk 26` device range, so a
real device selecting it as its primary ABI installs the app, then dies with `UnsatisfiedLinkError`
on the very first FFI call -- every function the app exists to perform. Nothing in the pipeline
could see this: `assembleDebug` exits 0 regardless, and the Robolectric unit tests run on the JVM
and never load a `.so` at all -- exactly the class of defect requirement #5 (a real, verifiable,
downloadable artifact) exists to catch.

Fixed as [`CADS-webconference-android` PR #13](https://github.com/scimbe/CADS-webconference-android/pull/13),
merged [`716c206`](https://github.com/scimbe/CADS-webconference-android/commit/716c206): a real
`defaultConfig.ndk.abiFilters` pins the packaged ABI set to the set an actual native bridge exists
for, plus a new Android CI step that fails the build if the packaged ABI set and the `jniLibs` ABI
set ever diverge again, or if a packaged ABI is missing its own `libnative_bridge.so` -- a real,
bidirectional invariant, not just the one-time fix. Both directions of that guard were checked
against real bytes before it shipped: it failed on the actual, already-published defective APK
(naming the five offending ABIs) and passed once they were gone.

**This is also the review the goal doc's own `no_review_for_succeeded_work` risk exists to
demand, not skip**: this fix, plus a separate, earlier real security fix
([`CADS-webconference-android@79774cd`](https://github.com/scimbe/CADS-webconference-android/commit/79774cd),
`recv_text` overriding a forged `sender_pubkey` with the real handshake-authenticated key), had
both landed as real `succeeded: true` iterations with no substantive review since -- the live run's
own real risk annotation said so. Iteration 32 closed that honestly: the native-bridge test suite
rebuilt and run hermetically from scratch (15/15 pass, including the exact forged-key regression
test), `cargo clippy -- -D warnings` independently clean, the ABI-fix diff read directly against
`origin/main` rather than trusted from its own commit message, and both the PR branch's and
post-merge `main`'s real CI runs confirmed green. No new defect found in either -- the point of a
real review isn't to always find something, it's to have actually looked.

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

**Update, 2026-08-09**: a fourth and fifth real increment on the handler side -- real image OCR
(PNG/JPEG/TIFF/WebP/BMP/GIF, via a real `tesseract` invocation; `image/svg+xml` deliberately stays
unsupported, since leptonica -- tesseract's own image library -- does not rasterize SVG), and the
scanned-PDF fallback that becomes possible once OCR exists: `pdftotext` still runs first, and only
when it comes back empty does the document get rasterized (`pdftoppm`) and OCR'd page by page.
Bounded at 20 pages *by erroring, not by truncating* -- the real page count is read from `pdfinfo`
before any rendering happens, so an over-cap document reports its own real size rather than quietly
returning its first pages as the whole thing. See [Add, remove, and manage RAG
documents]({{ '/how-to/manage-rag-documents/' | relative_url }}) for the current, full format list.

This PR (#57) merged without the real, hands-on review every prior increment on this thread got --
found and closed as its own gap, not waved through retroactively: a real verification image
(`rust:1-slim` plus the actual `poppler-utils`/`tesseract-ocr`/`imagemagick`/`libreoffice-writer`
toolchain, none of which CI's own image carries) drove the compiled binary directly, not the
unit-tested fakes -- confirmed all 6 image formats OCR real generated text, a real 2-page scanned
PDF (control-verified via `pdftotext` to genuinely have no text layer first) OCR-falls-back
correctly in page order, a real 21-page scanned PDF is rejected naming its real page count rather
than truncated, and a real text-layer PDF still takes the fast `pdftotext` path with `tesseract`
genuinely absent from `PATH`. One real, minor, non-blocking finding along the way: a BMP encoded
with an alpha channel fails with an honest leptonica error (`cannot read compressed BMP files`) --
reproduced identically invoking `tesseract` directly, so a real leptonica format gap, not a bug in
this PR's own code; the handler's own behavior (a real, honest error, never a fabricated success)
is still correct.

**Still genuinely open**: the same OIDC bearer-token blocker above, still. Format coverage on the
handler side is real, tested, and now reviewed -- none of it is the same as a live upload actually
reaching this channel in production yet.

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

**Update, 2026-08-09 -- requirement #5 is genuinely 5/5, and a real platform bug found getting the
last artifact uploaded.** [`CADS-webconference-android` PR #16](https://github.com/scimbe/CADS-webconference-android/pull/16)
added a real `verify-release-install` CI job (a clean, GitHub-hosted x86_64 Android emulator, `adb
install` against the actual built release APK, then a real package-manager check -- not just an
install command's exit code) after its first two real CI attempts genuinely failed for real reasons,
not flakiness: dash's `set -o pipefail` incompatibility, then this action's own per-line `sh -c`
execution splitting a `\` line continuation into a literal argument on the next line. Both root-caused
from the actual failing logs, not assumed, before shipping the fix. Merged
[`c045c2d`](https://github.com/scimbe/CADS-webconference-android/commit/c045c2d).

Confirming criterion 3 against that evidence needed the real post-merge artifact, not the PR-preview
run's own ephemeral merge-commit SHA (`$GITHUB_SHA` on a `pull_request` event, not the real branch
head) -- waited for the actual `push`-triggered run on `main`, downloaded that artifact, and
independently recomputed its SHA-256 by hand before trusting it. With that, only criterion 0
("downloadable from this run's own UI") remained -- and issue #36 (the platform gap that criterion
depended on) had already closed earlier the same session with the Build Artifacts panel. Uploading
the real, CI-verified APK through it hit a second real bug on the very first genuine attempt: axum's
`Multipart` extractor enforces its own default 2 MiB request-body limit, completely independent of
`upload_artifact`'s own stated 150MB cap -- the handler's limit was dead code for anything past
2MiB, and no prior upload in this project had ever been large enough to hit it. Fixed with
`DefaultBodyLimit::max(MAX_ARTIFACT_BYTES)` on the router, a regression test uploading a real 3MB
payload (deliberately over the old failure point), then the real APK re-uploaded, downloaded back
byte-identical, and confirmed genuinely visible in the live Build Artifacts panel via Playwright.
**Every acceptance criterion requirement #5 names is now true, live-verifiably, closing an item that
was 0/5 at the start of this session** -- see [Upload, download, and remove a build
artifact]({{ '/how-to/manage-build-artifacts/' | relative_url }}) for the real, current screenshot.

Requirement #17's criterion 0 (the exact `hex_decode_32` non-ASCII repro, pasted through the real
UI) closed the same way: [PR
#17](https://github.com/scimbe/CADS-webconference-android/pull/17) added a real Espresso
instrumented test, reusing PR #16's own emulator CI infrastructure. Its first real run also failed
for a real, root-caused reason -- Espresso's `typeText()` drives the on-device IME's own key-event
synthesis, and the AVD's default IME has no key event for U+20AC, throwing before the app ever saw
the input. Fixed by switching to `replaceText()`, which is also the more faithful simulation of the
actual bug report this test reproduces: a *pasted* key, not one typed character by character.

Merged as [`00fb390`](https://github.com/scimbe/CADS-webconference-android/commit/00fb390) --
but not confirmed on the strength of the merge alone. Waited for the real post-merge `push` run on
`origin/main` (databaseId 31336212256), not the `pull_request`-triggered preview run the fix
itself was validated against, then read the actual job log rather than trusting the conclusion
field: `Starting emulator.` -> `Starting 1 tests on test(AVD) - 10` -> `Finished 1 tests on
test(AVD) - 10` -> `BUILD SUCCESSFUL in 2m 16s`, no `FAILED` line anywhere. Criterion 0 confirmed
against the real deployment on that evidence.

**Update, 2026-08-09 -- requirement #17 is now genuinely 5/5, fully closed.** Criterion 4, the
last one open, made a distinct claim from criterion 0: not just that the UI shows the right error,
but that no Rust panic trace appears in `logcat` itself. `ConnectFlowInstrumentedTest` (the same
test criterion 0 closed) never inspected `logcat` at all -- its own KDoc had claimed this since
PR #17, but the assertion never existed, only implicit survival (reaching the final Espresso
check). Closed for real in
[PR #18](https://github.com/scimbe/CADS-webconference-android/pull/18): `logcat -c` before the
Connect interaction, `logcat -d` after, asserting none of three real markers appear (Rust's own
default panic-hook output, the platform's `FATAL EXCEPTION` line any uncaught exception produces
regardless of app-side logging setup, and UniFFI's own `Rust panic` `InternalException` message
text).

A real, separate bug surfaced shipping it: the local `webconference-android` checkout still had
PR #17's own pre-squash commits as its "main", so the new branch conflicted with the real
`origin/main` (`00fb390`'s squash-merge) the moment a PR was opened. Root-caused rather than
force-pushed blind -- rebuilt the branch cleanly on the real `origin/main` plus just the one new
commit before pushing. Merged as
[`a00f9ed`](https://github.com/scimbe/CADS-webconference-android/commit/a00f9ed), confirmed on the
real post-merge `push` run (not the pre-merge preview), criterion 4 toggled via the live API and
re-verified through `/requirements/export`: requirement #17 stands at 5/5, every criterion
carrying real confirmation evidence.

**Update, 2026-08-10 -- requirement #22 (native-bridge frame-decoding robustness), two of four
criteria real and merged.** `ChannelSession::recv_text` decrypts whatever bytes a peer sends on an
established Noise_IK channel -- authenticated, but still attacker-controlled content, exactly the
class of input a hostile or simply buggy peer can corrupt. Criteria 0 and 1 (a hermetic hostile-frame
test set in `native-bridge/src/message.rs`, and a named `MAX_MESSAGE_BYTES` bound checked before any
real parsing work) were already real and passing.

Criterion 3 -- "a rejected frame is dropped without tearing down the established channel or dropping
subsequent well-formed messages from the same peer" -- traced into a real, live bug reading the
actual `MainActivity.receiveLoop()` code: it caught `ChannelException` around its whole
`while(true)`, so a single malformed frame ended the loop identically to a genuinely dead connection
and reset the UI with a misleading "disconnected" status, even though the transport was still
healthy. Fixed in [PR #20](https://github.com/scimbe/CADS-webconference-android/pull/20): the catch
now scopes to one `recvText()` call, and a new, directly-testable `isFatalChannelError()`
distinguishes a single rejected frame (keep listening) from a genuinely dead channel (reset).
Confirmed against the real CI test-report artifact, not just "build succeeded": the new Robolectric
test's own result row, 1/1, 0 failures. Merged as
[`b3d9bcb`](https://github.com/scimbe/CADS-webconference-android/commit/b3d9bcb), both the pre-merge
PR run and the real post-merge `push` run fully green.

Criterion 2 (no decode failure may cross the UniFFI boundary as a Rust panic, proven by a real
on-device emulator test inspecting `logcat`) remained open -- requirement #22 is not yet fully
verified.

**Update, 2026-08-10 -- criterion 2's native-session half proven hermetically, plus a real CI gap
found and closed along the way.** `send_text` always encodes a well-formed message (there's no way
to construct a malformed one through it), so a new hermetic test sends raw, deliberately invalid
bytes through the exact same real `a2a_send` encryption `send_text` uses internally -- a genuine
authenticated-hostile-peer scenario, not a mock -- and confirms `recv_text` returns a real, typed
`ChannelError::Decode`, never panics, and (criterion 3's own claim, proven again at this level) a
real follow-up message from the same peer still arrives normally afterward.

A real, significant CI gap surfaced verifying this same test actually ran anywhere:
`native-bridge`'s own hermetic test suite -- every test this whole thread has relied on -- was never
actually run by CI at all. `verify-native-bridge` only rebuilds and diffs committed artifacts;
`build-and-test`'s own "Unit tests" step is Gradle's `testDebugUnitTest`, the Kotlin/Robolectric
side, never `cargo test`. A real regression in this crate's own logic could have merged with zero CI
signal. Fixed with a real `cargo test --locked` step, verified locally first before trusting it in
CI. Merged as [PR #21](https://github.com/scimbe/CADS-webconference-android/pull/21)
(`2a2e36b`) -- confirmed by reading the actual CI job log, not just its conclusion field: the new
test's own name and a real `24 passed; 0 failed` line both appear in it. Both the pre-merge PR run
and the real post-merge `push` run went fully green (4/4 jobs each).

Requirement #22 now stands at 3.5/4: criteria 0, 1, 3 fully verified; criterion 2's native-session
half is real and tested, its on-device emulator half remains open -- no UniFFI API exists yet to
send deliberately malformed bytes from Kotlin, so that half needs a new native-bridge API surface
plus new instrumented-test infrastructure, correctly scoped as a separate, larger follow-up.
