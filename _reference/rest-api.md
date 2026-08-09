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
| `GET /api/runs/{id}/open-points` | Every real item this run is actually waiting on a human to decide, ordered: the paused checkpoint first if paused (with its real `pause_reason`, and any real draft next-step options nested inside it), then the same real pending-proposal queues badged elsewhere in the GUI (panel add/edit/remove, stage, issue, deleting the run itself, and -- added 2026-08-09, issue #56 -- a proposed requirement, `kind: "requirement_proposal"`), same order, then any leftover draft next-step option once the run isn't paused anymore (its own real open point, `kind: "next_step_draft"` -- never lost to a resume). Deliberately excludes unverified requirements and stalled stages -- both are normal run states, not a stalled decision. Powers the **Open Points** panel -- see [Work through a run's open points]({{ '/how-to/work-through-open-points/' | relative_url }}). |
| `POST /api/runs/{id}/next-steps/propose` | `devsystem.assistant` drafts one real, plain-text next-iteration-plan option (`{"text": "..."}`) at a paused checkpoint. No approve/apply step -- a draft never mutates anything on its own, it's advisory text a human reads, edits, or discards. Rejects empty/whitespace-only or oversized (>4,000 byte) text. |
| `POST /api/runs/{id}/next-steps/{draft_id}/update` | A human edits an existing draft's text directly (`{"text": "..."}`) -- same validation as proposing one. Real `404` for an unknown draft id. |
| `POST /api/runs/{id}/next-steps/{draft_id}/remove` | A human discards a draft, permanently. Real `204` on success, `404` for an unknown draft id. |
| `POST /api/runs/{id}/iterate` | Submit a real `IterationRecord` -- see the how-to guide above. A submission byte-identical to the run's own immediately-preceding entry is rejected with a real `409` -- see [Why /iterate rejects exact duplicates]({{ '/explanation/duplicate-iteration-guard/' | relative_url }}). |
| `GET /api/runs/{id}/checkin` | Renders the latest iteration as real check-in markdown (run summary, risk annotations, decision prompt) -- callable any time, not gated on whether one is actually due. See [Review a mandatory check-in]({{ '/how-to/review-a-checkin/' | relative_url }}). |
| `POST /api/runs/{id}/checkin/acknowledge` | Real, explicit, idempotent acknowledgment that a human has reviewed the run's most recently fired check-in (added 2026-08-07) -- clears `health.checkin_pending` and the Runs list's `needs_attention` badge for it. Viewing the check-in markdown alone never counts; this is the one real action that does. Body is genuinely optional; an optional `{"note": "..."}` (added 2026-08-09) is persisted as a new, append-only `RunState::checkin_notes` entry with real provenance (`X-Gate-Email`, a real timestamp, which iteration it answers) -- same length cap and bidi rejection every other free-text field gets. The response, and `GET /api/runs/{id}`'s own `state`, also carry `checkin_acknowledged_through_id` (and each `checkin_notes` entry an `iteration_id`) -- the real, stable id of the iteration actually being acknowledged, not just its position (added 2026-08-09, issue #42 suggestion #1) -- see [How real risk annotations work]({{ '/explanation/risk-annotations/' | relative_url }})'s `check-in acknowledgment watermark` check for what reads this field. See [Review a mandatory check-in]({{ '/how-to/review-a-checkin/' | relative_url }}#knowing-a-check-in-is-actually-due-even-if-you-missed-the-moment). |
| `POST /api/runs/{id}/history/{iteration_id}/withdraw` | Tombstone a bad history record in place (added 2026-08-09, issue #42 suggestion #2): `{"reason": "..."}`, `reason` required and non-empty (same length cap and bidi rejection as every other short free-text field). Looks the record up by its real, stable `id` (issue #38/#52), never by array position, and only ever sets `withdrawn: true`/`withdrawn_at`/`withdrawn_by`/`withdrawn_reason` -- never deletes or reorders. The record keeps its original `iteration` number and array index forever, so no other record's ordinal, and no existing prose cross-reference to it, ever moves. Real `404` for an unknown id, `400` for an already-withdrawn record or an empty reason. Owner-restricted like every other GUI mutation. Deliberately does not (yet) exclude a withdrawn record from `iterations_completed`, checkin cadence, or `max_iterations` -- see [Why /iterate rejects exact duplicates]({{ '/explanation/duplicate-iteration-guard/' | relative_url }}) for the incident this replaces a worse fix for. |
| `POST /api/runs/{id}/criteria` | Update a run's `AbortCriteria` (`max_iterations`, `max_consecutive_failures`, `checkin_every`). `max_iterations`/`max_consecutive_failures` must be at least 1; all three fields must be at most 10,000 -- a bounded super loop needs a real, finite bound (a live test once got a real `200` for `u32::MAX`, before this cap existed). `checkin_every: 0` is still a legitimate value (flagged as an advisory risk, not rejected). |
| `POST /api/runs/{id}/pause` / `/resume` | Pause/resume a run. |
| `POST /api/runs/{id}/repo` | Set the run's target `repo_url`. Must start with `https://` (or be empty, to clear it) and be under 2,000 characters -- the same real length cap every other short free-text field in this API has, closed 2026-08-06 after a live test found this one had none (a genuine GitHub URL is nowhere near this length). |
| `POST /api/runs/{id}/operator-pubkey` | Set the real ed25519 operator public key a role's `ChannelId` derives from. |
| `POST /api/runs/{id}/adopt` | Give an unowned run a real owner -- the real `X-Gate-Email` caller, only when `owner_email` is currently unset (added 2026-08-09). Real `409` (naming the existing owner) against an already-owned run -- one-shot claim, not a transfer. Real `401` for a headless/no-gate-header caller. Not admin-restricted -- narrower than the write access an unowned run already grants everyone, not a new permission. See [Claim an unowned run]({{ '/how-to/claim-a-run/' | relative_url }}). |

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
| `POST /api/runs/{id}/requirements` | Add a requirement (EARS statement + acceptance criteria). Each criterion needs at least 5 alphanumeric characters and at most 500; a request with multiple bad criteria gets all of them named in one `400`, not just the first -- see [Requirements, verification, and automode]({{ '/explanation/requirements-and-automode/' | relative_url }})'s "what actually counts as a real acceptance criterion" section. The created requirement's `created_by` is stamped from the real `X-Gate-Email` session header (added 2026-08-07) -- honestly `null` for a header-less (CLI/M2M) submission, never guessed. Distinct from `proposed_by`, which is unrelated: human-authored (`null`) vs. LLM-proposed (`Some(stage_tag)`), not which account. |
| `POST /api/runs/{id}/requirements/{index}/update` | Correct a requirement's statement and/or acceptance criteria in place (added 2026-08-07) -- the same real EARS/length/bidi validation `POST /requirements` applies. Resets `verified` and every criterion's own confirmation back to unconfirmed (the previously-confirmed text may no longer be what's being asked); leaves `created_by`/`proposed_by` untouched. No remove endpoint exists -- an index is what iteration history's `requirement_indices` points at, so removing would renumber every later index and break existing references; update-in-place avoids that. See [Correct a wrong requirement]({{ '/how-to/edit-a-requirement/' | relative_url }}). |
| `POST /api/runs/{id}/requirements/propose` | Queue a real `{"statement": ..., "acceptance_criteria": [...], "rationale": ...}` requirement proposal for a human to review (added 2026-08-09, issue #56's first slice) -- reuses the exact same EARS/length/bidi validation `POST /requirements` applies to `statement`/`acceptance_criteria`; `rationale` is separately required, non-empty, under 2,000 characters, and bidi-checked. Never touches the real `requirements` list on its own -- see the two endpoints below. |
| `POST /api/runs/{id}/requirements/proposals/{proposal_id}/approve` | Move a pending requirement proposal into the real `requirements` list. `proposed_by` is always `"devsystem.assistant"` (never client-supplied, matching `propose_stage`'s own accountability convention); `created_by` is the real approving human's own `X-Gate-Email` session header, honestly `null` for a header-less approval. Real `404` for an unknown proposal id, real `400` past the `requirements` defensive cap. |
| `POST /api/runs/{id}/requirements/proposals/{proposal_id}/reject` | Discard a pending requirement proposal outright -- nothing was ever a real requirement, so there's nothing to undo beyond removing it from the pending list. Real `404` for an unknown or already-resolved proposal id. |
| `POST /api/runs/{id}/requirements/{index}/toggle` | Toggle a requirement's overall `verified` flag. Marking it verified is a real, hard-blocked `409` if this run declares a `review` role and no successful `devsystem.review` iteration has addressed this requirement yet -- see [Requirements, verification, and automode]({{ '/explanation/requirements-and-automode/' | relative_url }})'s "real, mandatory review gate" section. Un-verifying is always unconditional. |
| `POST /api/runs/{id}/requirements/{index}/auto-judge/toggle` | Opt a requirement into/out of assistant automode -- see [Requirements, verification, and automode]({{ '/explanation/requirements-and-automode/' | relative_url }}). |
| `POST /api/runs/{id}/requirements/{index}/criteria/{criterion_index}/toggle` | Toggle one acceptance criterion. As of 2026-08-07 a confirmed criterion is a real object, `{"confirmed_by": ..., "confirmed_at": ...}`, not a bare `true` -- `confirmed_by` is the real `X-Gate-Email` session header (honestly `null` if the request carried none), `confirmed_at` a real Unix timestamp. Un-confirming clears the whole record back to `null`, not just a boolean flip. A pre-migration run's legacy `true` becomes `{"confirmed_by": null, "confirmed_at": null}` on first load -- confirmed, but who/when predates this field and is honestly unknown, never fabricated. See [See a requirement's real decision basis]({{ '/how-to/see-a-requirements-decision-basis/' | relative_url }}#who-confirmed-a-criterion-and-when). |
| `GET /api/runs/{id}/requirements/export` | A real, downloadable Markdown document of every requirement -- statement, a real checklist per acceptance criterion, and provenance (human vs. LLM-proposed, per `proposed_by`). Each heading uses the run's own real 0-based ordinal (`## #{i}`, matching the GUI and `requirement_indices` exactly, fixed 2026-08-07 -- see [Requirements, verification, and automode]({{ '/explanation/requirements-and-automode/' | relative_url }})'s export section for why the old 1-based numbering silently pointed at the wrong requirement), and a real coverage line per requirement (`*Addressed by iteration(s) N.*` / `*Not yet addressed by any iteration.*`). Real `Content-Disposition: attachment`. |

## Build artifacts

Added 2026-08-07 (issue #36): a real, downloadable build artifact per run -- the structural gap that
made a requirement like `webconference-android`'s #5 ("SHALL produce a downloadable, installable
release APK artifact that is traceable to the exact source commit") impossible to actually satisfy,
since an iteration is free text only and nothing stopped an unverifiable claim like "APK built, sha256
abc123" from being marked `succeeded: true`. A defensive cap of 10 artifacts per run and
150,000,000 bytes (150 MB) per upload applies. Shipped backend-only first (the same backend-first
precedent `devsystem_document_extraction_client` set); the **Build Artifacts** GUI panel followed
2026-08-09 -- see [Upload, download, and remove a build artifact]({{ '/how-to/manage-build-artifacts/' | relative_url }}).

| Route | What it does |
|---|---|
| `GET /api/runs/{id}/artifacts` | List every real artifact uploaded to this run (added 2026-08-09, alongside the GUI panel -- nothing could previously enumerate a run's own artifacts at all). Owner-gated like every other per-run read that isn't the top-level listing. |
| `POST /api/runs/{id}/artifacts` | Upload a real build artifact -- `multipart/form-data` with a `file` field plus required `producing_iteration` (an integer) and `producing_stage` fields, and optional `source_commit`/`version_name`/`version_code`/`signing_identity` text fields. `sha256` is always computed server-side from the actual uploaded bytes via `sha2` -- there is no client-supplied `sha256` field to send, and one is silently ignored if sent as an extra multipart field, since the response's `sha256` always reflects what the server itself hashed. `producing_iteration` is cross-checked against this run's own real iteration history and rejected with a real `400` if it doesn't name one that actually exists -- traceability that's actually checkable, not just a number typed into a form. `uploaded_by` is stamped from the real `X-Gate-Email` session header, honestly `null` for a header-less upload. Real `400` past the 10-artifact-per-run cap. |
| `GET /api/runs/{id}/artifacts/{artifact_id}/download` | Download the real file, byte-identical to what was uploaded, as `Content-Disposition: attachment` under its original filename. Owner-gated like every other per-run read that isn't the top-level listing. Real `404` for an unknown artifact id. |
| `POST /api/runs/{id}/artifacts/{artifact_id}/remove` | Permanently delete a real artifact -- both its metadata and the actual file on disk. No undo. Real `404` for an unknown artifact id. |

Live-verified 2026-08-09 against the deployed `devsystem-demo.bunsenbrenner.org` container: uploaded a
real file, independently confirmed the server-computed `sha256` against a local `sha256sum` of the same
bytes, confirmed the downloaded bytes were byte-identical to the original, and confirmed a forged
client-supplied `sha256` field was silently ignored and the real hash recomputed server-side regardless.

## Auction and roles

| Route | What it does |
|---|---|
| `GET /api/runs/{id}/auction` | The real `PipelineSpec::auction_view` -- per-role bids and the winner, computed live. See [How auction selection policies work]({{ '/explanation/auction-selection-policies/' | relative_url }}) for what decides the winner. |
| `POST /api/runs/{id}/offers/submit` | Submit a real signed `CapacityOffer` (what `devsystem_offer` posts here). |
| `POST /api/runs/{id}/offers/quick-submit` | A GUI-friendly offer shortcut -- signs and submits in one call, no separate CLI identity needed. |
| `POST /api/runs/{id}/roles/{tag}/fill-mode` | Switch a role between auction-fill and a directly-assigned dedicated filler. Directly accepting a bid (`accepted_bid`) now gets a real `400` if its price exceeds the role's own real `price_ceiling` (closed 2026-08-07 -- see [Accepting a real bid directly]({{ '/how-to/set-auto-refresh-and-fill-mode/#accepting-a-real-bid-directly' | relative_url }})); a role with no real ceiling set still accepts any price. |

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
`pending_delete_run_proposal` (below) doesn't need this cap -- it's a single `Option`, not a queue,
since there's only ever one real run to propose deleting.

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
| `POST /api/runs/{id}/delete-proposal` | Propose deleting this run outright (pending). `rationale` is required (non-empty, under 2,000 characters, rejects a Unicode bidi control character) -- a run disappearing for good deserves a real, stated reason. Added 2026-08-07, the same propose-then-approve trust model as panel removal, since deleting a run is exactly as destructive and irreversible. |
| `POST /api/runs/{id}/delete-proposal/{proposal_id}/approve` | Resolve a pending delete proposal by actually deleting the run -- the identical real `fs::remove_dir_all` `DELETE /api/runs/{id}` above uses. Real `204` on success, matching `DELETE`'s own; no run left afterward to persist a "resolved" proposal into. |
| `POST /api/runs/{id}/delete-proposal/{proposal_id}/reject` | Decline the proposal -- the run was never touched. |

## RAG (document search)

| Route | What it does |
|---|---|
| `POST /api/runs/{id}/rag/sync` | Re-sync the run's indexed repo docs. |
| `GET /api/runs/{id}/rag/search` | Search the run's RAG index -- keyword always, or real semantic matching (`match_kind: "semantic"` per result) when either a static `RAG_EMBEDDING_API_KEY` credential or the `devsystem.embedding` channel (added 2026-08-07, same auction-discovered-ct-agent model as `devsystem.document_extraction`) is configured on this deployment. See [Add, search, and remove indexed documents]({{ '/how-to/manage-rag-documents/' | relative_url }}#search-is-live-as-you-type). |
| `POST /api/runs/{id}/rag/documents` | Add a document by URL/text. |
| `POST /api/runs/{id}/rag/upload-file` | Upload a real file. Unstructured API first if configured; otherwise the real `devsystem.document_extraction` channel if that's configured instead. As of 2026-08-09 both paths handle the same real format set -- PDF, DOCX, legacy DOC, plain text/markdown, and PNG/JPEG/TIFF/WebP/BMP/GIF images (real OCR via `tesseract` on the channel path) -- `image/svg+xml` stays unsupported on both. Real `503`, naming both, if neither is set. |
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
| `GET /api/version` | `{"git_sha": "..."}` -- the running binary's own build-time `git rev-parse HEAD` (added 2026-08-07), baked in via `web/Dockerfile`'s `GIT_SHA` build-arg. `"unknown"` for a local `cargo run` outside Docker, or an image built before this endpoint existed -- never fabricated. `deploy-devsystem-web.sh` compares this against the real, current source right after every deploy and fails loudly on a mismatch, catching a stale Docker build-cache regardless of which specific feature it happens to affect. See [The DAU lens and the incompetent-agent stress test]({{ '/explanation/dau-lens-and-stress-testing/' | relative_url }}) for the real incident this closes. |
