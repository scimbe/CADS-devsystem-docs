---
title: REST API reference
description: Every real route devsystem-web exposes, grouped by what it's for.
order: 1
---

# REST API reference

The real route table `web/src/main.rs` mounts, as of this writing -- not a design doc, the actual
`.route(...)` calls. `{id}` is a run id everywhere it appears.

## Auth model

- **Browser (gate cookie)**: a run's own writes (milestones, backlog, requirements, `repo_url`,
  fill-mode, custom panels, `devsystem.assistant`, ...) are scoped to the account that created the
  run -- `X-Gate-Email` must match `owner_email`, or the request gets `403`. A run created before
  per-run ownership existed has no recorded owner and stays open to any signed-in account.
- **Headless (no gate header)**: left deliberately unrestricted by `owner_authorized` -- every CLI
  tool this pipeline runs on (`devsystem_iterate`, this doc site's own screenshot automation, the
  autonomous dev loop) has never been gate-authenticated. See
  [Bid for a role and submit a real iteration]({{ '/how-to/submit-an-iteration/' | relative_url }})
  for the real M2M bearer-token path a headless caller uses against a gated public deployment.
- **`offers/submit`**: self-authenticating by real ed25519 signature (the offer itself), not gated
  at all -- "the signature is the authentication."

## Runs

| Route | What it does |
|---|---|
| `GET /api/runs` | List every run with a real summary (iterations, roles, stalled stages, risk count). |
| `POST /api/runs` | Create a new, empty run (`{"run_id": "..."}`). |
| `GET /api/runs/{id}` | A run's full real state: spec, health, risks, backlog, milestones, requirements, history. `risks` are real mechanical checks, not an LLM guess -- see [How real risk annotations work]({{ '/explanation/risk-annotations/' | relative_url }}). |
| `DELETE /api/runs/{id}` | Permanently remove a run's entire directory on disk -- every iteration, requirement, milestone, backlog item, chat exchange, and custom panel goes with it, no undo. Owner-restricted like every other GUI mutation. Real `204` on success, `404` for a run that's already gone (including a second delete racing the first). See [Delete a run]({{ '/how-to/delete-a-run/' | relative_url }}). |
| `GET /api/runs/{id}/open-points` | Every real item this run is actually waiting on a human to decide, ordered: the paused checkpoint first if paused (with its real `pause_reason`, and any real draft next-step options nested inside it), then the same five real pending-proposal queues `pending_reviews` already sums, same order, then any leftover draft next-step option once the run isn't paused anymore (its own real open point, `kind: "next_step_draft"` -- never lost to a resume). Deliberately excludes unverified requirements and stalled stages -- both are normal run states, not a stalled decision. Powers the **Open Points** panel -- see [Work through a run's open points]({{ '/how-to/work-through-open-points/' | relative_url }}). |
| `POST /api/runs/{id}/next-steps/propose` | `devsystem.assistant` drafts one real, plain-text next-iteration-plan option (`{"text": "..."}`) at a paused checkpoint. No approve/apply step -- a draft never mutates anything on its own, it's advisory text a human reads, edits, or discards. Rejects empty/whitespace-only or oversized (>4,000 byte) text. |
| `POST /api/runs/{id}/next-steps/{draft_id}/update` | A human edits an existing draft's text directly (`{"text": "..."}`) -- same validation as proposing one. Real `404` for an unknown draft id. |
| `POST /api/runs/{id}/next-steps/{draft_id}/remove` | A human discards a draft, permanently. Real `204` on success, `404` for an unknown draft id. |
| `POST /api/runs/{id}/iterate` | Submit a real `IterationRecord` -- see the how-to guide above. A submission byte-identical to the run's own immediately-preceding entry is rejected with a real `409` -- see [Why /iterate rejects exact duplicates]({{ '/explanation/duplicate-iteration-guard/' | relative_url }}). |
| `GET /api/runs/{id}/checkin` | Renders the latest iteration as real check-in markdown (run summary, risk annotations, decision prompt) -- callable any time, not gated on whether one is actually due. See [Review a mandatory check-in]({{ '/how-to/review-a-checkin/' | relative_url }}). |
| `POST /api/runs/{id}/criteria` | Update a run's `AbortCriteria` (`max_iterations`, `max_consecutive_failures`, `checkin_every`). `max_iterations`/`max_consecutive_failures` must be at least 1; all three fields must be at most 10,000 -- a bounded super loop needs a real, finite bound (a live test once got a real `200` for `u32::MAX`, before this cap existed). `checkin_every: 0` is still a legitimate value (flagged as an advisory risk, not rejected). |
| `POST /api/runs/{id}/pause` / `/resume` | Pause/resume a run. |
| `POST /api/runs/{id}/repo` | Set the run's target `repo_url`. Must start with `https://` (or be empty, to clear it) and be under 2,000 characters -- the same real length cap every other short free-text field in this API has, closed 2026-08-06 after a live test found this one had none (a genuine GitHub URL is nowhere near this length). |
| `POST /api/runs/{id}/operator-pubkey` | Set the real ed25519 operator public key a role's `ChannelId` derives from. |

## Backlog, milestones, requirements

Every list below is capped at 500 real items -- adding a 501st gets a real `400` naming the reason.
A defensive cap, not a design limit: nothing else stops a client from adding items in a tight loop,
and a run's `state.json` has to fit this host's own real, limited disk headroom. See
[The DAU lens and the incompetent-agent stress test]({{ '/explanation/dau-lens-and-stress-testing/' | relative_url }})
for how this was found and where else it applies.

| Route | What it does |
|---|---|
| `POST /api/runs/{id}/backlog` | Add a backlog item (`{"text": "..."}`). `text` must be non-empty and under 2,000 characters -- the same real cap every other free-text field in this API has, closed 2026-08-06 after a live test found this one had none (bounded only by the server's generic request-size limit). Also rejects a [Unicode bidi control character]({{ '/explanation/requirements-and-automode/' | relative_url }}), same day. |
| `POST /api/runs/{id}/backlog/{index}/toggle` | Toggle a backlog item's `done` flag. |
| `POST /api/runs/{id}/milestones` | Add a milestone. `description` has the identical non-empty/under-2,000-character requirement and bidi-control-character rejection as backlog `text` above -- worth noting here specifically, since a milestone's `achieved: true` transition auto-pauses the run as a real checkpoint a human trusts at face value. |
| `POST /api/runs/{id}/milestones/{index}/toggle` | Toggle a milestone's `achieved` flag. |
| `POST /api/runs/{id}/requirements` | Add a requirement (EARS statement + acceptance criteria). Each criterion needs at least 5 alphanumeric characters and at most 500; a request with multiple bad criteria gets all of them named in one `400`, not just the first -- see [Requirements, verification, and automode]({{ '/explanation/requirements-and-automode/' | relative_url }})'s "what actually counts as a real acceptance criterion" section. |
| `POST /api/runs/{id}/requirements/{index}/toggle` | Toggle a requirement's overall `verified` flag. Marking it verified is a real, hard-blocked `409` if this run declares a `review` role and no successful `devsystem.review` iteration has addressed this requirement yet -- see [Requirements, verification, and automode]({{ '/explanation/requirements-and-automode/' | relative_url }})'s "real, mandatory review gate" section. Un-verifying is always unconditional. |
| `POST /api/runs/{id}/requirements/{index}/auto-judge/toggle` | Opt a requirement into/out of assistant automode -- see [Requirements, verification, and automode]({{ '/explanation/requirements-and-automode/' | relative_url }}). |
| `POST /api/runs/{id}/requirements/{index}/criteria/{criterion_index}/toggle` | Toggle one acceptance criterion. |
| `GET /api/runs/{id}/requirements/export` | A real, downloadable Markdown document of every requirement -- statement, a real checklist per acceptance criterion, and provenance (human vs. LLM-proposed, per `proposed_by`). Real `Content-Disposition: attachment`. |

## Auction and roles

| Route | What it does |
|---|---|
| `GET /api/runs/{id}/auction` | The real `PipelineSpec::auction_view` -- per-role bids and the winner, computed live. See [How auction selection policies work]({{ '/explanation/auction-selection-policies/' | relative_url }}) for what decides the winner. |
| `POST /api/runs/{id}/offers/submit` | Submit a real signed `CapacityOffer` (what `devsystem_offer` posts here). |
| `POST /api/runs/{id}/offers/quick-submit` | A GUI-friendly offer shortcut -- signs and submits in one call, no separate CLI identity needed. |
| `POST /api/runs/{id}/roles/{tag}/fill-mode` | Switch a role between auction-fill and a directly-assigned dedicated filler. |

## Proposals -- pipeline stages, custom panels, GitHub issues

Each of the three proposal kinds follows the same real shape: `propose` lands in a pending queue,
`approve`/`reject` resolves it. See [How the pipeline proposes and grows its own stages]({{ '/explanation/self-optimizing-pipeline/' | relative_url }})
for why this queue exists for `devsystem.assistant`'s own proposals but not a role-filler's
iteration-embedded ones.

`custom_panels` and every pending-proposal queue here (stage, panel-add, panel-removal, panel-edit,
issue) share the same 500-item defensive cap the backlog/milestones/requirements lists above have --
a real gap for a while: only the latter three ever had it. Live-confirmed before the fix: 510 custom
panels added in a row with zero rejections. Now every list here rejects its own 501st item with a
real `400` too. See [The DAU lens and the incompetent-agent stress test]({{ '/explanation/dau-lens-and-stress-testing/' | relative_url }}).

| Route | What it does |
|---|---|
| `POST /api/runs/{id}/stages/propose` | Propose a new pipeline stage (pending). `rationale` rejects a [Unicode bidi control character]({{ '/explanation/self-optimizing-pipeline/' | relative_url }}), same as `stage_id`/`tag` being non-empty. |
| `POST /api/runs/{id}/stages/proposals/{proposal_id}/approve` \| `/reject` | Resolve a pending stage proposal. |
| `POST /api/runs/{id}/panels/propose` | Propose a new custom panel (pending). |
| `POST /api/runs/{id}/panels/proposals/{proposal_id}/approve` \| `/reject` | Resolve a pending panel proposal. |
| `POST /api/runs/{id}/panels` | Add a custom panel directly (no proposal step). |
| `POST /api/runs/{id}/panels/{panel_id}/remove` | Remove a custom panel directly. |
| `POST /api/runs/{id}/panels/{panel_id}/update` | Edit an existing custom panel's title/html directly, in place -- see [Add, edit, propose, and remove custom panels]({{ '/how-to/manage-custom-panels/' | relative_url }}). |
| `POST /api/runs/{id}/panels/{panel_id}/propose-remove` | Propose removing an existing custom panel (pending). |
| `POST /api/runs/{id}/panels/removal-proposals/{proposal_id}/approve` \| `/reject` | Resolve a pending panel-removal proposal -- approving actually removes the real panel. |
| `POST /api/runs/{id}/panels/{panel_id}/propose-edit` | Propose editing an existing custom panel's title/html (pending). |
| `POST /api/runs/{id}/panels/edit-proposals/{proposal_id}/approve` \| `/reject` | Resolve a pending panel-edit proposal -- approving actually overwrites the real panel's content. |
| `POST /api/runs/{id}/issues/propose` | Propose filing a real GitHub issue (pending). `title`/`body` reject a [Unicode bidi control character]({{ '/explanation/requirements-and-automode/' | relative_url }}), same class as every other real free-text field. |
| `POST /api/runs/{id}/issues/proposals/{proposal_id}/approve` \| `/reject` | Resolve a pending issue proposal -- approving actually files it. |

## RAG (document search)

| Route | What it does |
|---|---|
| `POST /api/runs/{id}/rag/sync` | Re-sync the run's indexed repo docs. |
| `GET /api/runs/{id}/rag/search` | Search the run's RAG index (keyword, or semantic if an embedding credential is configured). |
| `POST /api/runs/{id}/rag/documents` | Add a document by URL/text. |
| `POST /api/runs/{id}/rag/upload-file` | Upload a real file. Unstructured API first if configured (PDF/DOCX/image); otherwise the real `devsystem.document_extraction` channel if that's configured instead (PDF/DOCX/legacy DOC/plain text/markdown, never images). Real `503`, naming both, if neither is set. |
| `POST /api/runs/{id}/rag/documents/{doc_id}/remove` | Remove an indexed document. |

## Assistant

| Route | What it does |
|---|---|
| `POST /api/runs/{id}/assistant` | Ask `devsystem.assistant` a real question about this run -- see [Ask devsystem.assistant about your run]({{ '/how-to/ask-the-assistant/' | relative_url }}). |
| `GET /api/assistant/status` | Whether an assistant bridge is configured and actually reachable -- never leaks the bridge's internal address. |

## Misc

| Route | What it does |
|---|---|
| `GET /api/runs/{id}/memory` | A run's durable `memory.jsonl` (every iteration's zylos envelope). |
| `POST /api/runs/{id}/memory/{index}/govern` | Promote a memory entry from `Trust::Unreviewed` to `Trust::Governed` -- an explicit human review action, never automatic. |
| `GET /api/me` | The currently signed-in identity, if any. |
