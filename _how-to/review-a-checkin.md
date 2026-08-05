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
- **What this stage found** -- the latest iteration's own `feedback` text in full.
- **Proposals** -- any new stages/panels/issues this iteration proposed.
- **Stages added to the live spec so far** -- the run's `added_stages`.
- **Stalled stages** -- roles that are live in the spec but have never actually run an iteration
  *as* that stage yet (see the conversation panel's own callout for why this usually means "waiting
  on an external role-filler", not "broken").
- **Risk annotations** -- see below.
- **Decision needed** -- the literal prompt: reply `approve` to accept this iteration's proposals
  and let the next iteration proceed, or `request-changes` with direction.

## The three real pre-flight checks

`preflight_annotations` (`pipeline/src/preflight.rs`) runs three mechanical checks over a run's
history -- not an LLM judgment call, just patterns a human reviewer would otherwise have to notice
by hand:

| Check | Fires when |
|---|---|
| **touches auth/security** | The latest iteration's feedback, or a proposal's rationale, mentions a security-relevant keyword (e.g. "session", "auth", "credential"). |
| **no test stage before implement** | `devsystem.implement` ran before any `devsystem.test` iteration ever ran on this run. |
| **no price ceiling set** | A proposal declares a brand-new service (no `use_existing_service`) with no `price_ceiling` -- nothing bounds what filling it could actually cost. |

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
