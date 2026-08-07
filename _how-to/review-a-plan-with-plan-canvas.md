---
title: Review a plan with the Plan Canvas panel
description: Point at the part of a plan you mean instead of retyping a whole review -- a real annotate-and-approve panel, live-captured.
order: 15
---

# Review a plan with the Plan Canvas panel

Direct operator request, verbatim, 2026-08-07: *"ein echtes Plan Canvas panel: review plans by
pointing, not retyping."* This project's own architecture has named ECC's `ecc-plan-canvas` as the
intended plan-stage human-review mechanism since [The Development System](https://github.com/scimbe/CADS-Tunnel/issues/382)'s
very first framing -- this page documents the real panel that request shipped, live-captured while
writing it.

## Not the same thing as ecc-plan-canvas -- and why

[Review a mandatory check-in]({{ '/how-to/review-a-checkin/' | relative_url }}) already documents a
*different*, pre-existing real integration: `devsystem_checkin` hands a run's check-in markdown to
the actual `ecc-plan-canvas open` CLI tool -- a real, working annotate-an-element-and-approve
browser session. So why build something new instead of reusing that?

Investigated directly before writing any code: `ecc-plan-canvas` is a single **loopback-only** local
server (`127.0.0.1:4517`), keyed by local file path, with no concept of runs, account ownership, or
multiple concurrent reviewers -- it's built for one developer reviewing one local artifact on their
own machine. `devsystem-web` is the opposite shape: a real multi-tenant web app, owner-scoped runs,
accessed by different signed-in browsers over the internet, same as every other panel in this app.
Embedding the real tool would mean running a separate process/port, reverse-proxying it through
Caddy, and living with review state that's local files rather than `RunState` like everything else.

So this panel is a **native reimplementation of the same real UX** -- click the part of a plan you
mean, leave a comment anchored to it, deliver a verdict -- backed by this run's own real state, with
verdicts feeding directly into the same real gates every other iteration submission goes through.
Both mechanisms are real and both stay; they review different things (a periodic check-in cadence vs.
a `devsystem.plan` iteration's own content) and neither substitutes for the other.

## 1. Open the panel

The launcher (the orange dot, bottom-left) has a **Plan Canvas** puck once a run has at least one
`devsystem.plan` iteration to review. Selecting it shows that iteration's real feedback text, split
into real, independently-clickable blocks -- not a single wall of prose:

<figure>
<img src="{{ '/assets/img/howto-plan-canvas/01-empty-canvas.png' | relative_url }}" alt="The Plan Canvas panel showing a real three-phase plan split into three clickable blocks, zero annotations yet, Approve plan and Request changes buttons at the bottom">
<figcaption>A real <code>devsystem.plan</code> iteration's feedback, split on its own paragraph breaks. Each block is a real click target.</figcaption>
</figure>

## 2. Point at the part you mean

Click (or focus + Enter/Space, the same keyboard-accessible pattern every other real click target in
this app uses) any block to open a real annotation form anchored to it -- the exact excerpt you
clicked, not a line number or a CSS selector that would mean nothing once the plan text scrolls or
changes:

<figure>
<img src="{{ '/assets/img/howto-plan-canvas/02-annotate-form-open.png' | relative_url }}" alt="Phase 2's block highlighted with an accent border, the annotation form open below it reading 'Annotating: Phase 2: add Room-backed persistence so message history survives an app restart.'">
<figcaption>The form names the real block you pointed at -- write the comment, nothing else needs retyping.</figcaption>
</figure>

Submitting calls the real `POST /api/runs/{id}/plan-canvas/annotate` endpoint
(`{"anchor_snippet": "...", "text": "..."}`) -- both fields go through the same real validation
discipline as every other free-text field in this app: non-empty after trim, a real length cap, and
the same Unicode bidi-control-character check that closes the Trojan Source class everywhere else.
The annotation lands immediately, listed below the plan text:

<figure>
<img src="{{ '/assets/img/howto-plan-canvas/03-annotation-added.png' | relative_url }}" alt="Annotations (1): 're: Phase 2: add Room-backed persistence...' followed by the real comment text and a Remove button">
<figcaption>A real, removable annotation -- click <b>Remove</b> before delivering a verdict if you pointed at the wrong thing.</figcaption>
</figure>

You can add as many annotations as the plan actually needs, each anchored to its own block.

## 3. Deliver a real verdict

Two buttons, two real, different outcomes -- both calling `POST /api/runs/{id}/plan-canvas/verdict`:

### Request changes

Requires **at least one real annotation** -- asking for changes with nothing pointed at gives the
plan's next author no actual signal to act on, so the server rejects an empty request outright
(`400`). A real request does two things: the annotations **stay in place** (not summarized away --
the exact things you pointed at remain visible on the panel), and a real backlog item names them for
whoever picks the plan up next:

<figure>
<img src="{{ '/assets/img/howto-plan-canvas/04-backlog-item.png' | relative_url }}" alt="The Backlog panel showing a real item: 'Plan Canvas: request changes -- 1 annotation(s): [Phase 2: add Room-backed persistence...] Split this into its own follow-up iteration...'">
<figcaption>A real, durable backlog item -- not a message that only existed in this one browser session.</figcaption>
</figure>

### Approve plan

Folds the whole review into a real `devsystem.review` iteration -- through the **exact same real
gates** a normal [`/iterate` submission]({{ '/how-to/submit-an-iteration/' | relative_url }}) goes
through (the paused check, the [consecutive-failure/iteration
ceiling]({{ '/how-to/why-did-my-run-pause/' | relative_url }}), the [duplicate-submission
guard]({{ '/explanation/duplicate-iteration-guard/' | relative_url }})) -- not a separate,
less-guarded path just because it started from a friendlier panel. Live-verified directly: a run
already at its real iteration ceiling refuses a Plan Canvas approval with the identical `409` a
normal submission would get.

The real feedback text is synthesized from your actual annotations (a genuinely empty review --
zero annotations -- still approves, with an honest "reviewed as written" feedback line, not
fabricated detail that was never there). Approving **clears** the annotations, since the review
session has concluded, and the real iteration lands immediately in History:

<figure>
<img src="{{ '/assets/img/howto-plan-canvas/05-history-after-approve.png' | relative_url }}" alt="The History panel: iteration 2 · devsystem.review · ok, feedback reading 'Plan approved via Plan Canvas (1 annotation(s) addressed): [Phase 2: add Room-backed persistence...] Split this into its own follow-up iteration...', above the original iteration 1 devsystem.plan entry">
<figcaption>A real, ordinary <code>devsystem.review</code> iteration -- satisfies the same mandatory review gate any other real review does, nothing Plan-Canvas-specific about how it's stored.</figcaption>
</figure>

## What this doesn't do (yet)

This is a real, working first slice, not the whole of issue #40's own broader "who did this" picture.
The approving account is recorded honestly via [`submitted_by`]({{ '/how-to/submit-an-iteration/' | relative_url }}#stage-itself-is-validated-now-too-2026-08-07) --
but there's no per-annotation authorship if more than one reviewer points at the same plan, and
annotations aren't yet threaded into a back-and-forth conversation the way `ecc-plan-canvas`'s own
real chat feature is. Both are real, open, un-built follow-ons, not silently assumed solved.
