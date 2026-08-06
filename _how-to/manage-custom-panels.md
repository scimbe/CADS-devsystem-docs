---
title: Add, propose, and remove custom panels
description: A human can add or remove a panel directly; devsystem.assistant can only propose either, gated behind your approval.
order: 8
---

# Add, propose, and remove custom panels

The Custom Panels manager lets a run carry its own sandboxed HTML panels -- a burndown chart, a
release note, anything project-specific the fixed panel set doesn't cover. This page walks through
the real flow against a live run, and the two genuinely different trust models involved depending
on who's asking.

## Adding one directly

Open **Custom Panels** and fill in a title and HTML. The HTML renders sandboxed
(`<iframe sandbox="allow-scripts">`, no `allow-same-origin`) -- no access to this page or its
session, whatever the panel's own markup does.

<figure>
<img src="{{ '/assets/img/howto-manage-custom-panels/01-add-and-live-panel.png' | relative_url }}" alt="The Custom Panels manager showing one real live panel, 'Release Burndown', with an Open button, and the add-panel form below it">
<figcaption>A real panel already added ("Release Burndown"), and the direct add form below it -- title, HTML, one click.</figcaption>
</figure>

**Add panel** takes effect immediately -- no approval step, because a human directly filling this
form in is already the accountable decision. Removing one you added is just as direct: the
existing **Remove** button next to a live panel asks for a real confirmation first (this is
permanent -- there's no undo, the panel's HTML isn't kept anywhere once it's gone).

## What devsystem.assistant can do instead

The chat assistant can suggest a new panel or suggest removing an existing one -- but neither takes
effect on its own. Both land in a real pending queue and need your explicit approval, the same
gated shape as its pipeline-stage and GitHub-issue proposals (see
[How the pipeline proposes and grows its own stages]({{ '/explanation/self-optimizing-pipeline/' | relative_url }})
for why a role-filler's own proposals skip this queue but the assistant's never do). Ask it
something like *"can you propose removing the Release Burndown panel"* and it calls the real
`propose_remove_custom_panel` action -- never the direct remove endpoint -- so nothing disappears
without you seeing it first.

<figure>
<img src="{{ '/assets/img/howto-manage-custom-panels/02-pending-removal-proposal.png' | relative_url }}" alt="The Custom Panels manager showing a pending removal proposal for 'Release Burndown', with Approve and remove (orange) and Reject (keep it) buttons, above the still-live panel entry"><figcaption>The panel is still live below -- proposing removal doesn't touch it. Nothing is gone until Approve & remove is clicked.</figcaption>
</figure>

## The two proposal kinds are inverted on purpose

Approving vs. rejecting a proposal isn't symmetric across the two directions, and the GUI's
confirmation dialogs reflect that honestly rather than guarding both sides the same way:

| Proposal | The destructive click | Guarded with `confirm()` |
|---|---|---|
| **Add** a panel | **Reject** -- discards a drafted panel nobody saved elsewhere | Reject |
| **Remove** a panel | **Approve** -- actually deletes a real, live panel | Approve |

For an add-proposal, rejecting is the one-way door (the drafted title/HTML only ever existed in
that pending entry). For a removal proposal, it's the opposite -- approving is the one-way door (a
real panel with real content disappears), while rejecting is completely safe and just clears the
pending entry, leaving the panel exactly as it was. Same reasoning the direct **Remove** button
already uses, applied consistently to the assistant's gated path too.
