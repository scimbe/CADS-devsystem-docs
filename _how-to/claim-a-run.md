---
title: Claim an unowned run
description: Give a run with no recorded owner a real one -- what changes, and why it's one-shot.
order: 18
---

# Claim an unowned run

Every run created through this pipeline's own headless CLI/automation -- which is most of them --
has no recorded owner. [Delete a run]({{ '/how-to/delete-a-run/' | relative_url }}) already covers
what that means for deletion; this page covers the other half: since 2026-08-09, an unowned run no
longer has to stay that way forever.

## Finding the claim button

Any run with `owner_email: null` shows a small 🏳 button in the **Runs** panel, next to its own
name/summary button and its delete button:

<figure>
<img src="{{ '/assets/img/howto-claim-a-run/01-unowned-run-with-adopt-button.png' | relative_url }}" alt="The Runs panel showing several unowned runs, each ending 'no recorded owner' and carrying a small flag-shaped claim button next to its delete button">
<figcaption>Every one of these ends "no recorded owner" -- the 🏳 button only appears on a run that actually has none.</figcaption>
</figure>

A run that already has an owner never shows this button -- there's nothing to claim.

## What claiming actually does

Clicking 🏳 asks for a real confirmation first, naming what's about to change:

> Claim "&lt;run_id&gt;"? It has no recorded owner right now, so any signed-in account (including
> you) can write to it. Adopting makes future writes exclusive to your account -- this is one-shot,
> not reversible from here.

Confirming calls the real `POST /api/runs/{id}/adopt` route, which sets `owner_email` to your own
real, signed-in identity. From that moment on, this run behaves exactly like any other owned run
everywhere else in this app: a different signed-in account gets a real `403` trying to write to it
(or view it), the Runs panel row now reads `created by <you>` instead of `no recorded owner`, and
[deleting it]({{ '/how-to/delete-a-run/' | relative_url }}#ownership-only-protects-a-run-that-has-one)
is restricted to you specifically.

**It's one-shot.** There's no unclaim/transfer control here -- once a run has an owner, the 🏳
button stops appearing for everyone, including the account that claimed it. If ownership needs to
move to someone else, that's not something this action does.

## Why any signed-in account can do this, not just an "admin"

This platform has no admin role anywhere in its real access model. Claiming isn't a new permission
being granted -- it's strictly *narrower* than what an unowned run already allows: today, any
signed-in account can already write to milestones, backlog, `repo_url`, fill-mode, and custom
panels on an unowned run with zero attribution. Claiming converts that into "one specific account,
attributably." The first real signed-in person to click 🏳 on a run they care about is the whole
mechanism -- there's no separate approval step.

## A headless caller can't claim a run

`POST /api/runs/{id}/adopt` requires a real `X-Gate-Email` session header -- a genuine signed-in
human claiming responsibility, not something a harness or CLI script can do to itself invisibly.
Called with no session (the same shape `devsystem_iterate --remote`'s M2M bearer-token caller
uses), it's a real `401`.
