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

## Accepting a real bid directly

The **Roles** panel also lists every real, live bid on a role -- not just the auction's current
winner -- each with its own **Accept directly** button, right next to the price. This is a separate,
faster path to the same "Set as dedicated" outcome above: skip typing a label and picking the winner
yourself, just click the specific bid you want.

<figure>
<img src="{{ '/assets/img/howto-panel-controls/07-accept-directly-bid-pending.png' | relative_url }}" alt="The Roles panel showing the plan role with a live offer and the gpu_training role with a live offer of 200, each with an Accept directly button next to it">
<figcaption>Every real bid on a role gets its own "Accept directly" button -- not just the auction's own current winner.</figcaption>
</figure>

Clicking it asks for confirmation first (naming the real bidder and price), then submits the
identical request "Set as dedicated" above does, with the winning bid's own price attached as a real
snapshot.

**A real price_ceiling now actually bounds this, 2026-08-07**: if the role this bid is for was
proposed with a real, positive `price_ceiling` (see [How real risk annotations
work]({{ '/explanation/risk-annotations/' | relative_url }}) for how a run gets flagged when one is
missing), accepting a bid priced over that ceiling is now rejected outright, with the real numbers
stated plainly:

> this role's own real price_ceiling is 50 -- accepting this bid at 200 would exceed it; accept a
> lower bid, or raise the ceiling first by re-proposing this stage with a higher price_ceiling

Before this fix, `price_ceiling` was purely informational everywhere -- stored, shown, and flagged
by the risk panel when missing, but never actually compared against a real bid's price at the one
place a human accepts one directly. A role with **no** real ceiling set still accepts any price, same
as always -- there's nothing yet to enforce for that case, and the risk panel already flags it
honestly as unbounded. Auction-cleared bids (the more common path) still aren't checked against a
ceiling anywhere -- this fix covers the one real, local, one-click acceptance path this app owns
outright, not the whole auction pipeline.

**A real ceiling survives a careless re-proposal, found and fixed the same day**: re-proposing a role
(e.g. to change its rationale or units) without repeating its `price_ceiling` does **not** silently
remove an already-set one -- the effective ceiling is the last real, positive `price_ceiling` anyone
ever actually specified for that role, not just literally the most recent proposal. This matters
because the risk panel's own re-propose flow (the "Fix it" button, above) always shows an *empty*
price field, not the current value -- without this fix, using it for an unrelated reason (a role
already correctly bounded needing a units bump, say) and forgetting to re-type the price would have
silently un-bounded it again. A deliberate, later re-proposal that DOES set a different real ceiling
still changes it, exactly as you'd expect -- only an omission stops counting as removal.

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

**The other half of the same question, 2026-08-06: does Enter *submit*?** The dedicated-agent-label
input has no surrounding `<form>`, so before this fix Enter had no default behavior at all -- typing
a label and pressing Enter (the universal muscle memory for a lone single-line text field) did
nothing: no submission, no error, the popover just sat there.

<figure>
<img src="{{ '/assets/img/howto-panel-controls/05-enter-before-submit.png' | relative_url }}" alt="The fill-mode popover with 'Compass-1' typed into the dedicated-agent-label field, popover still open">
<figcaption>Typed and ready -- pressing Enter here used to do nothing at all.</figcaption>
</figure>

<figure>
<img src="{{ '/assets/img/howto-panel-controls/06-enter-after-submit.png' | relative_url }}" alt="The same Roles panel after pressing Enter, the fill-mode popover now closed">
<figcaption>After Enter: the popover closes and the role is really set as dedicated, the same as clicking "Set as dedicated" directly.</figcaption>
</figure>

Fixed by wiring Enter to the same submit path the button already uses, including its "label
required" validation
([CADS-devsystem@bfb2aee](https://github.com/scimbe/CADS-devsystem/commit/bfb2aee)).
