---
title: Open panels with the launcher, and find every keyboard shortcut
description: The green dot bottom-left replaced the old flat chip bar -- what it does, why panels are different sizes, and where the real Keyboard Shortcuts list lives.
order: 13
---

# Open panels with the launcher, and find every keyboard shortcut

The flat row of panel-name buttons across the top of the screen is gone (operator design ask,
2026-08-06) -- in its place, a fixed, un-draggable green dot sits in the bottom-left corner. The top
bar itself now shows the current project's real name instead.

<figure>
<img src="{{ '/assets/img/howto-panel-launcher/01-closed.png' | relative_url }}" alt="The Development System's landing page with the flat chip bar gone, the current project name shown in the top bar instead, and a small green dot fixed in the bottom-left corner">
<figcaption>The green dot is always there, always the same size (~1.5cm), never moves.</figcaption>
</figure>

## Opening it

Click the dot (or press Tab to it and hit Enter/Space, it's a real `<button>`, not a decorative
element). It unfolds into a real, animated circle segment confined to the corner -- not a
full-screen overlay -- with every real panel shown as a "bubble" inside it.

<figure>
<img src="{{ '/assets/img/howto-panel-launcher/02-open.png' | relative_url }}" alt="The launcher open, showing panel bubbles of different sizes fanning out from the green dot: large labeled circles for Process, Runs, Pipeline, and Requirements, smaller unlabeled circles for the rest, and a real red badge showing '1' on the Pipeline bubble" >
<figcaption>Pipeline's real red badge here is the same live pending-proposal count the old chip bar's badge used to show -- nothing about what a badge means changed, only how it's presented.</figcaption>
</figure>

## Panels are not all the same size, on purpose

Every panel is shown, but not with equal weight. Size encodes real, current relevance, not a fixed
ranking:

- A panel with a real pending decision (a proposal waiting on your approval, same signal as the old
  badge) is weighted up.
- The four starter panels (Runs, Process, Pipeline, Requirements -- the same set that's open by
  default on a first-time visit) are weighted up.
- A panel you already have open right now is weighted up (it also gets a small teal dot under its
  icon, a second, independent signal from size alone).
- Everything else is a plain, smaller icon.

The most important panels get a real, permanent, never-abbreviated text label. The rest are
icon-only -- hover or focus one to see its real full name in a floating label, the same convention
macOS' own Dock uses for exactly this reason: full names for everyone would either force the segment
to cover most of the screen, or force every name to truncate. Neither is on -- click any bubble,
labeled or not, and it opens (or brings to the front, if already open) the real panel behind it.

## Prefer typing? There's a real filter for that too

Live feedback after shipping the bubble-click version: hunting for one specific bubble by eye felt
worse than just typing a panel name, the way the Process Prompt's own `./show`/`./hide`/`./toggle`
commands already work. Rather than dropping the visual overview, the launcher now opens with a real
text field, already focused -- start typing and every non-matching bubble dims out of the way,
matching bubbles get a teal highlight:

<figure>
<img src="{{ '/assets/img/howto-panel-launcher/04-filter.png' | relative_url }}" alt="The launcher open with 'back' typed into its filter field, the Backlog bubble highlighted with a teal border, every other bubble dimmed out" >
<figcaption>Typing "back" narrows this down to Backlog, the one real match, highlighted -- everything else fades out of the way rather than disappearing outright.</figcaption>
</figure>

Press **Enter** once exactly one panel matches and it opens immediately, launcher closed -- the
identical matching rule (panel id or title, substring) the real `./show` command already trusts, not
a second guess at what counts as a match. If your filter still matches more than one panel, Enter
does nothing rather than guessing which one you meant -- keep typing (or click) instead.

## Keyboard

- **Escape** closes the launcher, same as it closes every other real popover/dialog in this app.
- **Ctrl+C** also closes it, while it's actually open -- but only when there's genuinely nothing
  selected. The filter field is focused the instant the launcher opens, so a blanket "always
  intercept Ctrl+C while open" rule would have made a real copy-to-clipboard from that field
  impossible. Select real text in the filter first, and Ctrl+C copies it normally, same as anywhere
  else -- it only closes the launcher when there's no selection to protect.
- Clicking (or Enter/Space-activating) a bubble always opens/brings-to-front that panel. It never
  toggles a panel closed, even if you pick one that's already open -- there's no way to see which
  panels are currently open while the launcher covers them, so a toggle would risk silently closing
  the very panel you just asked to see.

## Every real keyboard shortcut, in one place

Type `./shortcuts` in the Process Prompt (bottom of the screen) to open a real, accessible
**Keyboard Shortcuts** dialog -- every fixed behavior built into this app, plus any custom shortcut
you've bound yourself with `./bind <combo> <command>` (e.g. `./bind ctrl+k show requirements`), each
with a real **unbind** button right there.

<figure>
<img src="{{ '/assets/img/howto-panel-launcher/03-shortcuts.png' | relative_url }}" alt="The Keyboard Shortcuts dialog listing Escape, Ctrl+C, Enter, and Tab/Shift+Tab under 'Built in', and a message that no custom shortcuts are bound yet under 'Your custom bindings'" >
<figcaption>This used to be a one-line status message -- genuinely the wrong widget for a real list. Escape closes this dialog too, same as everything else.</figcaption>
</figure>
