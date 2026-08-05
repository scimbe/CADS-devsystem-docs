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
| `GET /api/runs/{id}` | A run's full real state: spec, health, risks, backlog, milestones, requirements, history. |
| `POST /api/runs/{id}/iterate` | Submit a real `IterationRecord` -- see the how-to guide above. |
| `GET /api/runs/{id}/checkin` | Whether a mandatory human check-in is currently due. |
| `POST /api/runs/{id}/criteria` | Update a run's `AbortCriteria` (`max_iterations`, `max_consecutive_failures`, `checkin_every`). |
| `POST /api/runs/{id}/pause` / `/resume` | Pause/resume a run. |
| `POST /api/runs/{id}/repo` | Set the run's target `repo_url`. |
| `POST /api/runs/{id}/operator-pubkey` | Set the real ed25519 operator public key a role's `ChannelId` derives from. |

## Backlog, milestones, requirements

| Route | What it does |
|---|---|
| `POST /api/runs/{id}/backlog` | Add a backlog item (`{"text": "..."}`). |
| `POST /api/runs/{id}/backlog/{index}/toggle` | Toggle a backlog item's `done` flag. |
| `POST /api/runs/{id}/milestones` | Add a milestone. |
| `POST /api/runs/{id}/milestones/{index}/toggle` | Toggle a milestone's `achieved` flag. |
| `POST /api/runs/{id}/requirements` | Add a requirement (EARS statement + acceptance criteria). |
| `POST /api/runs/{id}/requirements/{index}/toggle` | Toggle a requirement's overall `verified` flag. |
| `POST /api/runs/{id}/requirements/{index}/auto-judge/toggle` | Opt a requirement into/out of assistant automode -- see [Requirements, verification, and automode]({{ '/explanation/requirements-and-automode/' | relative_url }}). |
| `POST /api/runs/{id}/requirements/{index}/criteria/{criterion_index}/toggle` | Toggle one acceptance criterion. |

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

| Route | What it does |
|---|---|
| `POST /api/runs/{id}/stages/propose` | Propose a new pipeline stage (pending). |
| `POST /api/runs/{id}/stages/proposals/{proposal_id}/approve` \| `/reject` | Resolve a pending stage proposal. |
| `POST /api/runs/{id}/panels/propose` | Propose a new custom panel (pending). |
| `POST /api/runs/{id}/panels/proposals/{proposal_id}/approve` \| `/reject` | Resolve a pending panel proposal. |
| `POST /api/runs/{id}/panels` | Add a custom panel directly (no proposal step). |
| `POST /api/runs/{id}/panels/{panel_id}/remove` | Remove a custom panel. |
| `POST /api/runs/{id}/issues/propose` | Propose filing a real GitHub issue (pending). |
| `POST /api/runs/{id}/issues/proposals/{proposal_id}/approve` \| `/reject` | Resolve a pending issue proposal -- approving actually files it. |

## RAG (document search)

| Route | What it does |
|---|---|
| `POST /api/runs/{id}/rag/sync` | Re-sync the run's indexed repo docs. |
| `GET /api/runs/{id}/rag/search` | Search the run's RAG index (keyword, or semantic if an embedding credential is configured). |
| `POST /api/runs/{id}/rag/documents` | Add a document by URL/text. |
| `POST /api/runs/{id}/rag/upload-file` | Upload a real file (PDF/DOCX/image, via Unstructured if configured). |
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
