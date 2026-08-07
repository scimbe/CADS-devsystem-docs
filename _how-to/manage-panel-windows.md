---
title: Close, minimize, maximize, drag, and resize a panel
description: The traffic-light window controls every open panel carries, and what each one actually preserves.
order: 14
---

# Close, minimize, maximize, drag, and resize a panel

Every open panel is a real floating window (macOS convention: close = red, minimize = yellow,
maximize = teal), not a fixed layout block -- drag it by its header, resize it from its bottom-right
corner, and use the three traffic-light buttons to change how much of the screen it takes up.

<figure>
<img src="{{ '/assets/img/howto-panel-windows/01-normal-window.png' | relative_url }}" alt="The Pipeline panel in its normal floating-window state, with red, yellow, and teal traffic-light buttons in its header">
<figcaption>A panel in its normal state -- close, minimize, maximize, left to right, same order and colors on every panel.</figcaption>
</figure>

<figure>
<img src="{{ '/assets/img/howto-panel-windows/02-traffic-lights-closeup.png' | relative_url }}" alt="A close-up of the three traffic-light buttons in a panel's header">
<figcaption>Close-up: red closes, yellow minimizes, teal maximizes.</figcaption>
</figure>

## Close (red)

Hides the panel -- for the core panel set (Pipeline, Runs, Process, and the rest of the fixed set),
this doesn't delete anything, it just stops showing it. Reopen it any time from the
[panel launcher]({{ '/how-to/navigate-with-the-panel-launcher/' | relative_url }}) or with
`./show <panel>` in the Process Prompt. A [custom panel]({{ '/how-to/manage-custom-panels/' | relative_url }})'s
close button is different -- it really deletes that panel, and asks you to confirm first.

## Minimize (yellow)

Collapses the panel down to just its header -- the body's real state (scroll position, any typed
text, auto-refresh setting) is preserved underneath, not destroyed. Click the yellow button again
(or `./restore <panel>`) to bring it back exactly as you left it.

<figure>
<img src="{{ '/assets/img/howto-panel-windows/03-minimized.png' | relative_url }}" alt="The Pipeline panel minimized to just its header, sitting over the Requirements panel underneath">
<figcaption>Minimized: just the header remains, everything else preserved underneath.</figcaption>
</figure>

## Maximize (teal)

Expands the panel to fill the available desktop area (the real work surface to the left of
**devsystem.assistant**, which always stays visible and is never covered by a maximized panel).
Click the teal button again to restore the panel to its exact previous position and size -- the
real pre-maximize rectangle is remembered, not a generic default.

<figure>
<img src="{{ '/assets/img/howto-panel-windows/04-maximized.png' | relative_url }}" alt="The Pipeline panel maximized to fill the desktop area, with devsystem.assistant still visible on the right">
<figcaption>Maximized: fills the desktop, devsystem.assistant stays put on the right.</figcaption>
</figure>

The same command exists in the Process Prompt: `./max <panel>` maximizes (idempotent -- running it
again while already maximized leaves it maximized, it doesn't toggle back), `./restore <panel>`
undoes either a maximize or a minimize.

**A real bug here was found and fixed, 2026-08-07** ([issue #26](https://github.com/scimbe/CADS-devsystem/issues/26)):
a panel you'd left maximized in an earlier session used to come back **small** on your next visit,
even though it still silently believed it was maximized -- the first click on that panel's teal
button then appeared to do nothing (it was actually performing a correct-but-invisible "restore" from
small back to small), and only the *second* click visibly maximized it. Root cause: reopening the app
never re-rendered a restored panel at its real maximized size, only its old pre-maximize one. Fixed
at the source -- a panel you left maximized now correctly renders maximized again when you come back,
so the very first click behaves as expected.

## Drag and resize

Drag anywhere on a panel's header (not on one of the three buttons) to move it. Drag the small
handle in the bottom-right corner to resize it. Both are clamped to the visible desktop area -- a
panel can't be dragged or resized to a position that would put it partly off-screen.

## Shrinking the browser window itself

Making the whole browser window smaller (docking it, moving it to a smaller display, opening
devtools) can leave a panel positioned somewhere the new, smaller desktop area no longer reaches.
Rather than let it render off-screen or overlapping unpredictably, that panel is temporarily
hidden -- and a real, plain-text notice tells you so, naming exactly which panel(s):

<figure>
<img src="{{ '/assets/img/howto-panel-windows/05-viewport-fit-notice.png' | relative_url }}" alt="A notice reading 'Hid 3 panels that no longer fit this window: Process, Requirements, Pipeline -- comes back automatically if the window grows, or reopen from the launcher', shown top-left of the screen">
<figcaption>Real, current data -- not illustrative. The notice fades on its own after a few seconds; it's informational, not a decision.</figcaption>
</figure>

This used to be permanent -- growing the window back, or even reloading the page, wouldn't bring a
hidden panel back on its own ([issue #30](https://github.com/scimbe/CADS-devsystem/issues/30), a
real evaluator finding). Fixed: grow the window back to where a hidden panel would fit again, and it
comes back automatically -- no need to reopen it from the [panel launcher]({{ '/how-to/navigate-with-the-panel-launcher/' | relative_url }}),
though that still works too. Nothing you typed into a hidden panel is lost either way -- the panel's
DOM (and whatever draft text it held) survives underneath, it's just not shown until it fits again.
