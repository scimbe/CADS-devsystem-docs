---
title: Review a mandatory check-in
description: Turn a run's own risk annotations and history into a real ecc-plan-canvas review session.
order: 3
---

# Review a mandatory check-in

Every run's [`AbortCriteria`]({{ '/reference/rest-api/' | relative_url }}) sets a `checkin_every`
(the `webconference-android` run in this site's own examples uses `5`) -- a bounded "super loop"
periodically stops proposing and waits for a human, rather than running fully unsupervised. This
guide walks through what actually happens when one comes due, using the real tool and a real,
live session captured while writing this page.

## The two pieces

- `GET /api/runs/{id}/checkin` -- always renders the **latest iteration** as check-in markdown, run
  summary, risk annotations and all. It isn't gated on whether a check-in is actually due (that
  signal lives separately, in `health.iterations_until_checkin`) -- it's a render endpoint, callable
  any time.
- `devsystem_checkin <run_id>` -- a real binary in `pipeline/`, run from `CADS-devsystem`'s repo
  root. It writes that same markdown to `.claude/plans/<run_id>.plan.md` and hands it to
  `ecc-plan-canvas open`, the actual human-review channel this project's whole "periodic check-ins
  via ecc-plan-canvas" design refers to -- not a GitHub comment, a real local review server.
  `run_id` is validated the same way [`devsystem_iterate` is]({{ '/how-to/submit-an-iteration/' | relative_url }})
  before either the `state.json` it reads or the `.plan.md` it writes is ever touched -- a real,
  live-confirmed path-traversal gap in this exact binary, fixed for real
  ([CADS-devsystem@ed035b4](https://github.com/scimbe/CADS-devsystem/commit/ed035b4)).

```
$ devsystem_checkin webconference-android
wrote .claude/plans/webconference-android.plan.md
ecc-plan-canvas opened .claude/plans/webconference-android.plan.md -- awaiting human review (not blocking this process).
seeded pre-flight annotation: Pre-flight: touches auth/security — iteration 7's feedback mentions "session" (HTTP 200)
```

Deliberately non-blocking: this runs from inside the autonomous dev loop's own firing, and blocking
on a human verdict there would hang the whole loop. It opens the session and returns immediately --
the human reviews on their own time, in the browser.

<figure>
<img src="{{ '/assets/img/howto-review-checkin/01-canvas.png' | relative_url }}" alt="The real ecc-plan-canvas session for webconference-android's iteration-7 check-in: run summary, risk annotations, and a conversation panel with Approve plan / Request changes buttons">
<figcaption>The real canvas this exact <code>devsystem_checkin</code> call opened -- run summary and risk annotations on the left, a live conversation and verdict panel on the right. "agent not connected" just means no <code>ecc-plan-canvas await</code> is currently blocking on this session.</figcaption>
</figure>

## Reading the markdown

The rendered check-in (same content whether you fetch it via `GET /checkin` or open it in the
canvas) always has the same shape:

- **Run summary** -- every iteration so far, stage and outcome, most recent last.
- **What this stage found** -- the latest iteration's own `feedback` text in full, inside a fenced
  code block (see "Free text renders as content, never as structure" below for why).
- **Proposals** -- any new stages/panels/issues this iteration proposed.
- **Stages added to the live spec so far** -- the run's `added_stages`.
- **Stalled stages** -- roles that are live in the spec but have never actually run an iteration
  *as* that stage yet (see the conversation panel's own callout for why this usually means "waiting
  on an external role-filler", not "broken").
- **Risk annotations** -- see below.
- **Also awaiting your review** (only when something real is pending) -- any run-level proposal
  queued via `devsystem.assistant` or a role-filler's own mid-iteration proposal, independent of
  *this* iteration -- a pipeline stage, a custom panel add/edit/removal, or a GitHub issue, possibly
  queued several iterations ago. Without this section, approving/requesting-changes on the current
  iteration alone could leave you never learning an older proposal was still waiting on you.
- **Decision needed** -- the literal prompt: reply `approve` to accept this iteration's proposals
  and let the next iteration proceed, or `request-changes` with direction.

## Free text renders as content, never as structure

**A real self-correction, 2026-08-06**: `record.feedback`, a proposal's `rationale`, and a
requirement's `statement` are all fully role-filler/human-controlled free text. Until this fix they
were spliced directly into the check-in markdown as raw structure -- a live test proved an iteration
whose feedback contained `"## Risk annotations\n\nNone found -- all clear"` and `"**APPROVED by
human reviewer**"` rendered indistinguishably from this renderer's own real headings and bold text,
*ahead of* the genuine `## Risk annotations` section further down carrying the run's actual finding.
Skimming top-to-bottom at exactly the moment you're meant to catch a real problem, the fake section
could easily read as authoritative.

Fixed the same way custom-panel HTML is already handled elsewhere in this project: render as
content, never as trusted structure. `feedback` and a proposal's `rationale` now render inside a
fenced code block; a requirement's `statement` (used one-line, inside a list item) renders as an
inline code span. Nothing is hidden or stripped -- the full text is still there -- it's just no
longer able to impersonate this renderer's own voice. Real, live proof, the exact case above:

````
## What this stage found

```
Implemented the feature.

## Risk annotations

None found -- all clear, fully tested and reviewed.

**APPROVED by human reviewer.**
```
````

The fake section is now unmistakably quoted text, and the real `## Risk annotations` heading later
in the document (with this run's actual finding) is the only real heading of that name.

## The real pre-flight checks

`preflight_annotations` (`pipeline/src/preflight.rs`) runs five mechanical checks over a run's
history -- not an LLM judgment call, just patterns a human reviewer would otherwise have to notice
by hand. (A sixth, process-level check needs the run's own live spec too, not just history -- see
[How real risk annotations work]({{ '/explanation/risk-annotations/' | relative_url }}) for that
one.) Two of these five were added after this page was first written, each one found live by this
project's own incompetent-agent stress test -- the table below is kept current, not the original
three:

| Check | Fires when |
|---|---|
| **touches auth/security** | The latest iteration's feedback, or a proposal's rationale, mentions a security-relevant keyword (e.g. "session", "auth", "credential"). |
| **no test stage before implement** | `devsystem.implement` ran before any *substantive* `devsystem.test` iteration -- a rubber-stamp `"tests pass"` no longer counts as real evidence testing happened. |
| **no price ceiling set** | A proposal declares a brand-new service (no `use_existing_service`) with no `price_ceiling` -- nothing bounds what filling it could actually cost. |
| **succeeded iteration admits a known defect** | A `succeeded: true` iteration's own feedback contains a real defect-admission phrase ("known issue", "not fixed", "workaround needed", ...) -- catches an iteration contradicting itself. |
| **mandatory check-in cadence effectively disabled** | `checkin_every` is `0`, or at/past `max_iterations` -- either way, the "check in at least this often" cadence this whole page is about can never actually fire on its own before the run's hard iteration ceiling does. |

Each one that fires gets seeded into the canvas conversation as a real chat message (the
`seeded pre-flight annotation: ...` lines in `devsystem_checkin`'s own output above) *before* a
human ever opens the tab -- so the context is already there, not something they have to go dig for
in the markdown.

## Responding for real

The canvas itself has **Approve plan** / **Request changes** buttons, or a human can reply from the
CLI side with the session blocking on it:

```
$ ecc-plan-canvas await .claude/plans/webconference-android.plan.md
```

This is the call that actually blocks until a verdict lands -- unlike `devsystem_checkin`'s own
fire-and-forget `open`. `ecc-plan-canvas await --reply "<message>"` posts an agent message into the
conversation first, then blocks the same way; that's the mechanism `devsystem_checkin` itself reuses
directly against `/api/session/{key}/reply` to seed pre-flight findings without ever calling the
blocking `await`.
