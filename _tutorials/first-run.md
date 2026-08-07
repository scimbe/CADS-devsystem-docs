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

Every panel in the GUI (Runs, Requirements, Pipeline, Backlog, and sixteen more as of this writing)
is its own independently movable/resizable window. On your very first visit, only four are open by
default — enough to get oriented without being overwhelmed:

<figure>
<img src="{{ '/assets/img/tutorial-first-run/01-landing.png' | relative_url }}" alt="The GUI landing page with a small default set of panels open: Runs, Process, Pipeline, Requirements, and a small orange dot fixed in the bottom-left corner">
<figcaption>Runs, Process, Pipeline, and Requirements are open by default. Every other panel — Backlog, Milestones, Roles, Check-in, and so on — is one click away on the orange dot in the bottom-left corner. See <a href="{{ '/how-to/navigate-with-the-panel-launcher/' | relative_url }}">Open panels with the launcher</a> for what it does and why panels aren't all the same size.</figcaption>
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

The Requirements panel itself doesn't leave you guessing what to do next, either — a genuinely
empty run gets an explicit nudge instead of a plain, easy-to-miss "no requirements yet" line, and
the statement field is already focused so you can start typing immediately:

<figure>
<img src="{{ '/assets/img/tutorial-first-run/04b-start-here-nudge.png' | relative_url }}" alt="The Requirements panel on a fresh run, showing a 'Start here' callout above the empty requirement form">
<figcaption>This banner is real, live GUI behavior — not tutorial-only hand-holding. It disappears the moment the run has its first requirement, and never reappears.</figcaption>
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

## 4. The "automode" flag -- a real placeholder, honestly labeled

Every requirement's acceptance criteria are human-checked by default. Each requirement also has a
per-requirement **automode** checkbox -- corrected, 2026-08-06, to say plainly what it actually
does: nothing yet. Live-verified directly against the real deployment, `auto_judge` is never read
anywhere in `devsystem.assistant`'s own code. You can already ask the assistant in plain chat to
judge and verify any requirement's criteria, and it works identically whether this box is checked
or not -- it always could be asked. The checkbox is a real, honestly-labeled placeholder for future
opt-in judgment logic, not a switch that unlocks anything today:

<figure>
<img src="{{ '/assets/img/tutorial-first-run/07-auto-judge.png' | relative_url }}" alt="The automode checkbox checked on the new requirement, labeled 'automode flag (not wired to any real behavior yet -- see tooltip)'">
<figcaption>The real, current label -- honest about what checking this box does today (nothing) rather than implying a permission model that was never real.</figcaption>
</figure>

See [Requirements, verification, and automode]({{ '/explanation/requirements-and-automode/' | relative_url }})
for the full live investigation that found this.

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
for exactly that, continuing this same run. Or just ask -- see
[Ask devsystem.assistant about your run]({{ '/how-to/ask-the-assistant/' | relative_url }}) for a
real, grounded answer about where things actually stand, no `devsystem_iterate` required.

---

**A real gap this tutorial found and fixed (2026-08-05):** the first version of this walkthrough
hit a genuine bug — with no saved layout (everyone's first session), *every* panel defaulted to
open, not just four. At a real, unremarkable browser width, the resulting overlap meant the Custom
Panels window's own textarea sat directly on top of the Requirements panel's "Add requirement"
button, making it truly unclickable on a first-time user's very first real action. Fixed in
[CADS-devsystem`56182ea`](https://github.com/scimbe/CADS-devsystem/commit/56182ea) — screenshots
above are from the deployment *after* that fix, verified live before being written down here.

**A second real gap, found and fixed 2026-08-06:** the New Project dialog in step 2 above looked
fine but had no real keyboard focus management at all. Opening it never actually moved focus into
the dialog (its `autofocus` attribute silently never took effect — a real quirk of markup inserted
dynamically rather than present at page load), and with no focus trap, `Tab` walked straight through
the *entire page hidden behind the overlay* — a keyboard-only user tabbing through this exact step
could land on and edit fields they couldn't see at all, with no indication anything was wrong.

<figure>
<img src="{{ '/assets/img/tutorial-first-run/09-new-project-focus.png' | relative_url }}" alt="The New Project dialog immediately after opening, with the browser's real focus ring visible on the Project id field">
<figcaption>Real, live evidence after the fix: opening the dialog now moves focus straight to <strong>Project id</strong> — no click required — and Tab/Shift+Tab now cycle only this dialog's own four fields/buttons, wrapping at each end rather than escaping to the page behind it.</figcaption>
</figure>

Fixed in [CADS-devsystem`ed39496`](https://github.com/scimbe/CADS-devsystem/commit/ed39496) with the
standard accessible-modal pattern — the same "verify against the real deployment, not just the code"
discipline behind this dialog's two custom-popover siblings gaining real `Escape`-to-close support;
see [Set a panel's auto-refresh interval or a role's fill mode]({{ '/how-to/set-auto-refresh-and-fill-mode/' | relative_url }})
for that earlier fix.
