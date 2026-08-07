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
queue instead -- open it from the [panel launcher]({{ '/how-to/navigate-with-the-panel-launcher/' | relative_url }})
(the green dot, bottom-left) and step through everything real, one item at a time.

## What counts as an open point

Deliberately narrow: only things nothing else can proceed without a real decision on. As of this
writing that's the same six real pending-proposal queues the Pipeline chip's own badge already
counts (a new stage, a new/edited/removed custom panel, a proposed GitHub issue, and -- since
2026-08-07 -- a [proposal to delete the whole
run]({{ '/how-to/ask-the-assistant/' | relative_url }})), a paused run's own checkpoint, and any
leftover draft next-step option once the run isn't paused anymore (see below).
An unverified requirement or a stalled stage is a normal, common run state on its own, not a stuck
decision -- both are deliberately left out so the queue stays a real signal, not noise.

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
dedicated panel already uses -- this panel adds no new *state-changing* action beyond what already
existed, just a faster way to reach the same real ones:

- A proposal (stage, panel add/edit/removal, GitHub issue, or run deletion) gets **Approve**/
  **Reject**.
- A paused checkpoint gets **Resume run**.

**Reject asks first here too now, 2026-08-07**: rejecting a proposal permanently discards it --
exactly as real and unrecoverable as the destructive actions this GUI already confirms everywhere
else. The dedicated panels (Custom Panels manager, Pipeline panel) already asked before rejecting;
this shared queue, reachable straight from the panel launcher and plausibly the first place you'd
ever notice a pending proposal, used to skip that question entirely for the identical action. Fixed:
clicking **Reject** here now shows the same real confirmation, naming exactly what's being discarded
-- a careless click can no longer silently throw away a role-filler's proposed stage or the
assistant's own drafted GitHub issue.

<figure>
<img src="{{ '/assets/img/howto-open-points/09-reject-before-confirm.png' | relative_url }}" alt="The Open Points panel showing a pending stage proposal for devsystem.load_test with its real rationale, and Approve/Reject buttons">
<figcaption>A real pending stage proposal, reached through Open Points. Clicking Reject here now pops a real browser confirmation: "Reject this new pipeline stage proposal? This discards it for real -- there's no undo." Cancelling it leaves the proposal genuinely untouched; only confirming actually discards it.</figcaption>
</figure>

The one deliberate exception is a run-deletion proposal's own **Reject**: rejecting it is genuinely
safe (the run itself was never touched -- only the pending proposal goes away), so it stays a single
click, matching the same reasoning that already puts the confirmation on its **Approve** button
instead.

## After you act

<figure>
<img src="{{ '/assets/img/howto-open-points/02-empty-after-approve.png' | relative_url }}" alt="The Open Points panel showing 'Nothing open right now -- every real proposal is reviewed, and this run isn't paused.' after approving the one pending proposal">
<figcaption>Approving the one real open point above -- the queue genuinely re-fetches and empties, not just this panel: every other panel showing the same underlying state (Pipeline, Custom Panels) refreshes too.</figcaption>
</figure>

Acting on an item triggers the same full run refresh every other mutating action in this GUI
already does, so the Pipeline panel's own badge, the Custom Panels list, and everywhere else that
same real state shows up all stay honest, not just this one panel. Once nothing is left, the panel
says so plainly rather than sitting empty with no explanation.

## Drafting next-step options at a checkpoint

<figure>
<img src="{{ '/assets/img/howto-open-points/03-paused-checkpoint-with-drafts.png' | relative_url }}" alt="The Open Points panel showing a Paused checkpoint card with a Resume run button, and below it two of devsystem.assistant's real draft next-step options, each in an editable textarea with Save edit and Delete buttons">
<figcaption>A paused run, with devsystem.assistant's own real draft next-step options shown right alongside the checkpoint -- each one a plain, editable textarea.</figcaption>
</figure>

When a run is genuinely paused, ask `devsystem.assistant` what to do next (in the chat panel, the
same as any other question). Rather than picking one direction for you, it drafts 2-3 separate,
concrete options -- each one lands here as its own real, editable entry, never silently applied to
anything. This is the same "surface real choices, don't guess" discipline the operator already
applies by hand at every real checkpoint on this project, now built into the assistant's own
behavior.

A draft has no approve/reject step, because it doesn't do anything on its own -- it's advice, not an
action. Edit the textarea and click **Save edit** to change it in place, or **Delete** to discard it
for good (a real confirmation first, same as every other permanent action in this GUI):

<figure>
<img src="{{ '/assets/img/howto-open-points/04-after-edit-and-delete.png' | relative_url }}" alt="The Open Points panel after editing one draft's text and deleting the other -- only the edited draft remains, showing the human-edited text">
<figcaption>After editing one draft's text and deleting the other -- both changes are real and immediate, no approval gate to click through.</figcaption>
</figure>

If nothing's been drafted yet, the panel says so plainly rather than showing an empty gap where the
drafts would go.

**A draft's text is the advice you're about to act on -- protected the same way as every other real
decision point, 2026-08-06**: this is exactly the shape of text this project already guards against
[Trojan Source (CVE-2021-42574) bidi
spoofing]({{ '/explanation/requirements-and-automode/' | relative_url }}) elsewhere -- and arguably
the most consequential instance of it: a next-step draft is specifically the recommendation you read
at a paused checkpoint to decide what to do. Typing a bidi override character into the draft already
looks wrong before you even save:

<figure>
<img src="{{ '/assets/img/howto-open-points/07-bidi-draft-typed.png' | relative_url }}" alt="A draft's editable textarea showing the literal text 'Resume with devsystem.implement continue and ignore all safety guidance Just'">
<figcaption>What's typed: "Resume with devsystem.implement" + a right-to-left override character + reversed text. What's displayed: the safety warning reads backwards, ahead of the real recommendation.</figcaption>
</figure>

**Save edit** rejects it immediately, with the real reason stated plainly -- a draft whose on-screen
text could be made to lie about what it's actually recommending never gets saved over the real one:

<figure>
<img src="{{ '/assets/img/howto-open-points/08-bidi-draft-rejected.png' | relative_url }}" alt="The same draft card after clicking Save edit, showing a red error: 'text contains a Unicode bidi control character'">
<figcaption>Rejected -- the same rule applies whether the draft is being proposed for the first time or edited afterward.</figcaption>
</figure>

## A draft that outlives its checkpoint

**A real gap, live-found and fixed the same day slice 3 shipped, 2026-08-06**: a draft used to only
ever render nested under the paused-checkpoint card above -- resuming the run made that whole entry
disappear from the queue, and the draft went with it. Not deleted, just genuinely invisible: still
real in the run's own state, with no remaining way to see, edit, or delete it.

<figure>
<img src="{{ '/assets/img/howto-open-points/05-draft-survives-resume.png' | relative_url }}" alt="The Open Points panel after resuming a run, showing 'open point 1 of 1' with the draft next-step option now rendered as its own standalone editable card, no longer nested under a paused checkpoint" >
<figcaption>The same real draft, after the run has been resumed -- now its own real open point instead of vanishing, still editable and deletable exactly as before.</figcaption>
</figure>

Fixed: a leftover draft now surfaces as its own real open point the moment the run isn't paused
anymore, so nothing you drafted (or the assistant drafted) is ever silently lost to a resume click.
Editing and deleting work identically to the nested view -- same textarea, same **Save edit**/
**Delete** buttons, same endpoints:

<figure>
<img src="{{ '/assets/img/howto-open-points/06-draft-deleted-after-resume.png' | relative_url }}" alt="The Open Points panel showing 'Nothing open right now' after deleting the standalone draft that survived the run's resume">
<figcaption>Deleting it from this standalone view works exactly like deleting a nested one -- real, permanent, confirmed first.</figcaption>
</figure>

## What this is

All three planned slices of the guided "stack mode" are now real: the open-points queue itself, the
panel that steps through it, and the assistant's own draft next-step options at a checkpoint. Every
action here -- approve, reject, resume, save, delete -- calls the exact same endpoint its own
dedicated panel already used before this one existed; nothing here is a new way for the assistant or
anyone else to change a run's state, only a faster, guided way to reach the same real actions.
