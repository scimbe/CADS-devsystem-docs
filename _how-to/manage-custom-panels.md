---
title: Add, edit, propose, and remove custom panels
description: A human can add, edit, or remove a panel directly; devsystem.assistant can only propose any of the three, gated behind your approval.
order: 8
---

# Add, edit, propose, and remove custom panels

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

**"No access to this page or its session" was verified live, 2026-08-06, not just asserted.** Every
prior note about this sandbox in this project's own code and docs was a claim read from the
`sandbox="allow-scripts"` attribute -- never actually attacked this session until three real,
separate live attempts, each opened through this real GUI's own **Open** button (a real floating
panel window), inspected with a headless browser:

1. **Page/session access** -- a panel whose script tried to overwrite the main page's title via
   `window.parent.document`, read `document.cookie`, and write to `window.localStorage`. All three
   blocked (`"Blocked a frame with origin \"null\" from accessing a cross-origin frame"`,
   `"...lacks the 'allow-same-origin' flag"` twice) -- the main page's own title was never touched.
2. **Navigation/popups** -- a panel whose script tried `window.top.location.href = "https://evil.example/..."`
   and `window.open(...)`. Both blocked (`"the flag of 'allow-top-navigation' ... is not set"`,
   `"'allow-popups' permission is not set"`) -- the real browser tab's own URL never left
   `devsystem-web`, zero popups actually opened.
3. **Direct API mutation** -- the most consequential test: a panel whose script tried a real
   `fetch()` `POST` straight to this run's own `/milestones` endpoint with a planted description, no
   UI involved at all. Blocked at the browser's CORS preflight (`"Response to preflight request
   doesn't pass access control check"`) -- confirmed a third, independent way by re-fetching this
   run's own real state afterward: the planted milestone was never written, `milestones: []`.

Nothing here needed a fix -- the sandbox model documented above was already correct. Worth recording
that it was actually attacked and held, not just described, across the three meaningfully different
things a hostile panel could try.

**Add panel** takes effect immediately -- no approval step, because a human directly filling this
form in is already the accountable decision.

**A title and some real HTML content are both required.** A blank or whitespace-only title was
already rejected; a genuinely blank HTML body wasn't, until a real gap closed 2026-08-06: every
other real free-text field in this pipeline (milestones, backlog, requirement statements, stage
proposals) already rejects whitespace-only content, but a custom panel's own HTML was the one
exception -- live-confirmed, a panel with `html: ""` got a real `200` and sat there as a genuinely
blank, useless panel with nothing telling you it was empty:

```
$ curl -X POST .../api/runs/{id}/panels -d '{"title": "T", "html": "   "}'
html must not be empty
HTTP 400
```

Enforced identically at all four real entry points that accept panel HTML -- adding, editing, and
both of the assistant's proposal paths below.

**A panel's `title`, unlike its `html`, is real trusted UI chrome** -- it's what shows in the panel
list and what gets interpolated directly into this feature's own confirmation dialogs ("Remove
custom panel "Title"?"), so it needs the opposite trust treatment from `html` (deliberately
untrusted-by-design, sandboxed, never sanitized). A real gap closed 2026-08-06, extending the same
[Trojan Source bidi-control-character
fix]({{ '/explanation/requirements-and-automode/' | relative_url }}) already documented for
requirements: a title like `"Safe Panel"` followed by a right-to-left override character and
reversed text used to sail through untouched, displaying as an apparently-safe title while hiding
real content after it. Fixed at all four real title entry points (add, edit, and both assistant
proposal paths) --

```
$ curl -X POST .../api/runs/{id}/panels -d '{"title": "Safe Panel‮ ...", "html": "<p>x</p>"}'
title contains a Unicode bidi control character (e.g. a right-to-left override) -- these can
make the visually displayed text not match what's actually stored
HTTP 400
```

-- while `html` deliberately keeps its existing untrusted-by-design treatment; adding this same
check there would be inconsistent with the sandboxed-iframe model the whole feature already relies
on, not a fix.

## Editing one directly

Every live panel now has its own **Edit** button next to **Open** -- click it and that panel's
card swaps into a real inline form, pre-filled with its current title and HTML:

<figure>
<img src="{{ '/assets/img/howto-manage-custom-panels/03-edit-form.png' | relative_url }}" alt="The Custom Panels manager showing a live panel's card swapped into an inline edit form, pre-filled with its real current title 'GUI Demo Panel' and HTML">
<figcaption>Editing "GUI Demo Panel" in place -- the same panel, same real id, not a remove-and-re-add.</figcaption>
</figure>

**Save** applies immediately, same trust level as **Add panel** (it's your own content, your own
call) -- but it's a real overwrite, so it asks for confirmation first, the same way the direct
**Remove** button does: the panel's previous title/HTML isn't kept anywhere once you save over it.
**Cancel** discards the in-progress edit with no request at all -- the panel is untouched either
way until you actually click Save.

Removing a panel is just as direct: the existing **Remove** button next to a live panel asks for a
real confirmation first (this is permanent -- there's no undo, the panel's HTML isn't kept anywhere
once it's gone).

## What devsystem.assistant can do instead

The chat assistant can suggest a new panel, suggest editing an existing one, or suggest removing
one -- but none of the three takes effect on its own. All three land in a real pending queue and
need your explicit approval, the same gated shape as its pipeline-stage and GitHub-issue proposals
(see [How the pipeline proposes and grows its own stages]({{ '/explanation/self-optimizing-pipeline/' | relative_url }})
for why a role-filler's own proposals skip this queue but the assistant's never do). Ask it
something like *"can you propose removing the Release Burndown panel"* and it calls the real
`propose_remove_custom_panel` action -- never the direct remove endpoint -- so nothing disappears
without you seeing it first. Ask it to change an existing panel's content and it calls
`propose_edit_custom_panel` instead, with the full replacement title/HTML (not a diff).

<figure>
<img src="{{ '/assets/img/howto-manage-custom-panels/02-pending-removal-proposal.png' | relative_url }}" alt="The Custom Panels manager showing a pending removal proposal for 'Release Burndown', with Approve and remove (orange) and Reject (keep it) buttons, above the still-live panel entry"><figcaption>The panel is still live below -- proposing removal doesn't touch it. Nothing is gone until Approve & remove is clicked.</figcaption>
</figure>

<figure>
<img src="{{ '/assets/img/howto-manage-custom-panels/04-pending-edit-proposal.png' | relative_url }}" alt="The Custom Panels manager showing a pending edit proposal, 'GUI Demo Panel -> GUI Demo Panel v2', with the proposed new HTML shown, Approve and overwrite (orange) and Reject (keep original) buttons, above the still-live unchanged panel entry">
<figcaption>The real old title -&gt; new title and the proposed new HTML, so you're never approving a blind diff. The live panel below still shows its original content until you decide.</figcaption>
</figure>

## The proposal kinds are inverted on purpose

Approving vs. rejecting a proposal isn't symmetric across the three kinds, and the GUI's
confirmation dialogs reflect that honestly rather than guarding all three the same way:

| Proposal | The destructive click | Guarded with `confirm()` |
|---|---|---|
| **Add** a panel | **Reject** -- discards a drafted panel nobody saved elsewhere | Reject |
| **Edit** a panel | **Approve** -- overwrites the real panel's real content | Approve |
| **Remove** a panel | **Approve** -- actually deletes a real, live panel | Approve |

For an add-proposal, rejecting is the one-way door (the drafted title/HTML only ever existed in
that pending entry). For an edit or removal proposal, it's the opposite -- approving is the one-way
door (real content is overwritten or a real panel disappears), while rejecting is completely safe
and just clears the pending entry, leaving the panel exactly as it was. Same reasoning the direct
**Remove**/**Edit** buttons already use, applied consistently to the assistant's gated path too.
