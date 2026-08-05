---
title: Set up your first run
description: From no project at all to a real requirement, on a real pipeline, with real screenshots.
order: 1
---

# Set up your first run

By the end of this tutorial you'll have created a real CADS-devsystem run, written a real
fine-grained requirement for it, and seen the real pipeline/role state the system generates for a
brand-new project — all against the live deployment at
[devsystem-demo.bunsenbrenner.org](https://devsystem-demo.bunsenbrenner.org/). This page was
written by actually walking through the flow (real browser automation, not a mockup) and fixing a
real bug it found along the way — see the note at the end.

The example project here targets a real repo, [`CADS-webconference-android`](https://github.com/scimbe/CADS-webconference-android)
— the Android client for [webconference.bunsenbrenner.org](https://webconference.bunsenbrenner.org/).

## 1. Sign in and land on a clean layout

Every panel in the GUI (Runs, Requirements, Pipeline, Backlog, and a dozen more) is its own
independently movable/resizable window, toggled from the chip bar at the top. On your very first
visit, only four are open by default — enough to get oriented without being overwhelmed:

<figure>
<img src="{{ '/assets/img/tutorial-first-run/01-landing.png' | relative_url }}" alt="The GUI landing page with a small default set of panels open: Runs, Process, Pipeline, Requirements">
<figcaption>Runs, Process, Pipeline, and Requirements are open by default. Every other panel — Backlog, Milestones, Roles, Check-in, and so on — is one click away on the chip bar above.</figcaption>
</figure>

## 2. Create a new project

Click **+ New Project** in the Runs panel. You need a project id and, optionally, a GitHub repo URL
— the repo URL is what lets the Pipeline panel and Docs Search actually sync against real source.

<figure>
<img src="{{ '/assets/img/tutorial-first-run/02-new-project-dialog.png' | relative_url }}" alt="The New Project dialog, empty">
</figure>

<figure>
<img src="{{ '/assets/img/tutorial-first-run/03-new-project-filled.png' | relative_url }}" alt="The New Project dialog filled in with a project id and the CADS-webconference-android repo URL">
<figcaption>A real project id and the real target repo.</figcaption>
</figure>

Submitting creates a genuinely empty run — a plan-only `PipelineSpec`, no history, no roles beyond
`plan` yet. Every other stage (`implement`, `test`, `verify`, `review`, `improve`, `remember`)
enters the live spec later, through a real `StageProposal` a role-filler raises for itself once it's
actually needed — not pre-declared up front.

<figure>
<img src="{{ '/assets/img/tutorial-first-run/04-run-created.png' | relative_url }}" alt="The newly created run selected, showing an empty pipeline state">
</figure>

## 3. Write a real, fine-grained requirement

CADS-devsystem's requirements use [EARS notation](https://en.wikipedia.org/wiki/Easy_Approach_to_Requirements_Syntax)
(`WHEN <trigger>, THE SYSTEM SHALL <behavior>`) with concrete, checkable acceptance criteria — the
same shape a role-filler agent (or a human reviewer) can actually verify against, not a vague wish.

Real fine-grained specs are, honestly, hard to make perfectly reproducible between runs — the exact
wording you choose shapes what a role-filler builds. What *is* reproducible is the process itself:
write a real trigger, a real system behavior, and acceptance criteria specific enough that "done"
isn't a judgment call.

<figure>
<img src="{{ '/assets/img/tutorial-first-run/05-requirement-filled.png' | relative_url }}" alt="The requirement form filled in with a real EARS-style statement and three acceptance criteria">
<figcaption>A real requirement for the Android messaging feature, with three concrete acceptance criteria.</figcaption>
</figure>

<figure>
<img src="{{ '/assets/img/tutorial-first-run/06-requirement-added.png' | relative_url }}" alt="The requirement now listed in the Requirements panel with its acceptance criteria as checkboxes">
<figcaption>The requirement now lives on the run, each criterion independently checkable.</figcaption>
</figure>

## 4. Decide who judges "done" — human, or automode

Every requirement's acceptance criteria are human-checked by default. If you'd rather let
`devsystem.assistant` judge a *specific* requirement's criteria itself, opt it into **automode** —
a real, per-requirement flag, not a project-wide default:

<figure>
<img src="{{ '/assets/img/tutorial-first-run/07-auto-judge.png' | relative_url }}" alt="The automode checkbox checked on the new requirement">
<figcaption>Automode opted in for this one requirement. It stays off for every other requirement unless you opt those in too.</figcaption>
</figure>

Honest caveat: as of this writing, checking this box only *authorizes* the assistant to judge this
requirement later — it doesn't perform any judgment yet. The actual judgment logic (reading a run's
real iteration history/evidence and deciding whether a criterion is genuinely met) is real, separate,
still-open work.

## 5. See the real pipeline state

The Pipeline panel shows the run's actual `PipelineSpec` — which roles exist, which auction policy
governs them, and (via the Roles panel) the real bidding state for each:

<figure>
<img src="{{ '/assets/img/tutorial-first-run/08-roles-panel.png' | relative_url }}" alt="The Roles panel for the fresh run, showing the plan role and its auction state">
<figcaption>A fresh run starts with just the plan role — everything else arrives via real StageProposals as the pipeline discovers what it needs.</figcaption>
</figure>

## What's next

From here, a real role-filler (human or agent) wins the `plan` role's auction and submits a real
iteration via `devsystem_iterate`. See
[Bid for a role and submit a real iteration]({{ '/how-to/submit-an-iteration/' | relative_url }})
for exactly that, continuing this same run.

---

**A real gap this tutorial found and fixed (2026-08-05):** the first version of this walkthrough
hit a genuine bug — with no saved layout (everyone's first session), *every* panel defaulted to
open, not just four. At a real, unremarkable browser width, the resulting overlap meant the Custom
Panels window's own textarea sat directly on top of the Requirements panel's "Add requirement"
button, making it truly unclickable on a first-time user's very first real action. Fixed in
[CADS-devsystem`56182ea`](https://github.com/scimbe/CADS-devsystem/commit/56182ea) — screenshots
above are from the deployment *after* that fix, verified live before being written down here.
