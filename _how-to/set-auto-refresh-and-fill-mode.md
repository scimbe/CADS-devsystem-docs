---
title: Set a panel's auto-refresh interval or a role's fill mode
description: The auto-refresh gear on every panel and the fill-mode menu on every Roles row -- what they do, and how to close either one.
order: 12
---

# Set a panel's auto-refresh interval or a role's fill mode

Two small, easy-to-miss controls exist in the live control panel that had zero docs coverage until
this page: the auto-refresh **⚙** gear in every panel's header, and the fill-mode **⋯** menu on
every row of the **Roles** panel.

## Auto-refresh interval

Every panel window has a small ⚙ gear next to its title. Clicking it opens a popover with a real
choice of intervals -- off, 15s, 30s, 1min, 5min:

<figure>
<img src="{{ '/assets/img/howto-panel-controls/02-refresh-interval-popover.png' | relative_url }}" alt="The Roles panel's gear icon clicked, showing an AUTO-REFRESH popover with options off, 15s, 30s, 1min, 5min">
<figcaption>The auto-refresh popover, opened from the Roles panel's own ⚙ gear -- every panel has its own independent interval.</figcaption>
</figure>

Picking an interval re-fetches and re-renders just that one panel on the chosen cadence -- other
panels are untouched, and an idle panel left on **off** costs nothing. The gear itself lights up
(a filled accent color) whenever its panel has an interval set, so you can tell at a glance which
panels are actively polling without opening each one's popover.

## Role fill mode

Open the **Roles** panel and look for the small **⋯** next to each declared role. Clicking it opens
a popover offering two real fill strategies for that specific role:

<figure>
<img src="{{ '/assets/img/howto-panel-controls/01-fillmode-popover.png' | relative_url }}" alt="A role's fill-mode popover open, showing 'FILL MODE -- PLAN' with Auction (default) selected, a dedicated agent label field, and a Set as dedicated button"/>
<figcaption>The fill-mode popover for the "plan" role -- Auction is the default; "Set as dedicated" pins the role to one named agent instead.</figcaption>
</figure>

- **Auction (default)** -- the role stays open to any signed, qualifying bid, same as every role
  starts out. This is what most roles should stay on; it's what lets the pipeline's own
  self-optimization actually pick the best available filler each round.
- **Set as dedicated** -- type a label identifying the specific agent this role should be pinned
  to, and submit. New bids from other agents on that role are no longer eligible while it stays
  dedicated -- useful when you already know exactly who should be filling a role (a specific
  labor-setup.com service, a specific person) and don't want the auction re-deciding it every
  round.

Switching modes takes effect immediately -- there's no separate approval step, since choosing a
role's own fill strategy is the same kind of direct, accountable action as bidding on it yourself.

**The dedicated agent label is a real trust decision, protected the same way as every other one,
2026-08-06**: this label is exactly the shape of text this project already guards against [Trojan
Source (CVE-2021-42574) bidi
spoofing]({{ '/explanation/requirements-and-automode/' | relative_url }}) everywhere else --
short, human-typed, displayed and trusted -- but it had never actually been checked. Typing a label
containing a Unicode bidi override character already looks wrong in the input itself:

<figure>
<img src="{{ '/assets/img/howto-panel-controls/03-fillmode-bidi-typed.png' | relative_url }}" alt="The fill-mode popover's dedicated-agent-label input showing the literal text 'Trusted AgentThis is really a malicious agent'">
<figcaption>What's typed: "Trusted Agent" + a right-to-left override character + reversed text. What's displayed: "Trusted AgentThis is really a malicious agent".</figcaption>
</figure>

Submitting it is rejected immediately, with the real reason stated plainly:

<figure>
<img src="{{ '/assets/img/howto-panel-controls/04-fillmode-bidi-rejected.png' | relative_url }}" alt="The same popover after clicking Set as dedicated, showing a red error: 'label contains a Unicode bidi control character (e.g. a right-to-left override) -- these can make the visually displayed text not match what's actually stored'">
<figcaption>Rejected -- a label whose on-screen text could be made to lie about who you're actually trusting with a role never gets saved.</figcaption>
</figure>

## Closing either popover

Click anywhere outside the popover, or press **Escape** -- both close it without submitting
anything. Escape support is a real, live 2026-08-06 fix: before it, these two popovers were the
only interactive surfaces in this app that didn't honor the universal "close this" key already used
everywhere else (every native confirmation dialog in this app, and the assistant-prompt
autocomplete, already did) -- a user who reflexively hit Escape found it did nothing and had to go
hunt for where to click instead
([CADS-devsystem@5352d4c](https://github.com/scimbe/CADS-devsystem/commit/5352d4c)). Live-verified
against the real deployment via a headless-browser walkthrough: both popovers open on click and
close cleanly on Escape, with no console errors either way -- the same run used for the two
screenshots above.
