---
title: Work through a run's open points
description: One panel that steps through everything a run is actually waiting on you to decide -- a pending proposal, or a paused checkpoint -- one at a time.
order: 10
---

# Work through a run's open points

A run can accumulate several real, separate things waiting on a human decision at once: a
role-filler's proposed pipeline stage, an assistant-drafted custom panel, a paused checkpoint. Each
already has its own panel (Pipeline, Custom Panels, Health & Criteria), but noticing all of them
means checking every one of those panels yourself. The **Open Points** panel is a single guided
queue instead -- open it from the panel chip bar and step through everything real, one item at a
time.

## What counts as an open point

Deliberately narrow: only things nothing else can proceed without a real decision on. As of this
writing that's the same five real pending-proposal queues the Pipeline chip's own badge already
counts (a new stage, a new/edited/removed custom panel, a proposed GitHub issue), plus a paused
run's own checkpoint. An unverified requirement or a stalled stage is a normal, common run state on
its own, not a stuck decision -- both are deliberately left out so the queue stays a real signal,
not noise.

## Stepping through the queue

<figure>
<img src="{{ '/assets/img/howto-open-points/01-one-open-point.png' | relative_url }}" alt="The Open Points panel showing 'open point 1 of 1', a card labeled New pipeline stage proposal with its real stage id and rationale, and Approve/Reject buttons plus Prev/Next navigation">
<figcaption>One real open point -- a role-filler's own proposed stage, showing its real rationale. Approve/Reject here call the exact same endpoint the Pipeline panel's own proposal card does.</figcaption>
</figure>

Each entry shows its kind (a new stage proposal, a custom-panel add/edit/removal proposal, a GitHub
issue proposal, or a paused checkpoint) and a real, human-readable summary -- a stage proposal's own
rationale, a panel's title, or the run's own real `pause_reason`. **Prev**/**Next** move through the
queue without acting on anything.

The action buttons differ by kind, but every single one calls the identical endpoint its own
dedicated panel already uses -- this panel adds no new way to approve, reject, or resume anything,
just a faster way to reach the same real action:

- A proposal (stage, panel add/edit/removal, GitHub issue) gets **Approve**/**Reject**.
- A paused checkpoint gets **Resume run**.

## After you act

<figure>
<img src="{{ '/assets/img/howto-open-points/02-empty-after-approve.png' | relative_url }}" alt="The Open Points panel showing 'Nothing open right now -- every real proposal is reviewed, and this run isn't paused.' after approving the one pending proposal">
<figcaption>Approving the one real open point above -- the queue genuinely re-fetches and empties, not just this panel: every other panel showing the same underlying state (Pipeline, Custom Panels) refreshes too.</figcaption>
</figure>

Acting on an item triggers the same full run refresh every other mutating action in this GUI
already does, so the Pipeline panel's own badge, the Custom Panels list, and everywhere else that
same real state shows up all stay honest, not just this one panel. Once nothing is left, the panel
says so plainly rather than sitting empty with no explanation.

## What this isn't, yet

This is the second of three real, separate slices behind a fuller guided "stack mode" -- stepping
through what's already real. Still open: `devsystem.assistant` drafting first-cut next-iteration
plan options at a paused checkpoint (editable, never silently applied), with its own real audit
trail of what it proposed and when.
