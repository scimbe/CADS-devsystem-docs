---
title: Bookmark or share a link to a specific run
description: The URL now names the run you're looking at, so a reload, a bookmark, or a pasted link all land on the right one.
order: 19
---

# Bookmark or share a link to a specific run

Selecting a run in the [Runs panel]({{ '/how-to/claim-a-run/' | relative_url }}) puts a real,
bookmarkable address in the URL bar -- `#run=<the-run-id>`. Select `webconference-android` and the
address bar reads:

```
https://devsystem-demo.bunsenbrenner.org/#run=webconference-android
```

<figure>
<img src="{{ '/assets/img/howto-bookmark-a-run/01-app-with-run-selected.png' | relative_url }}" alt="The app with webconference-android selected -- every run-scoped panel's title bar and the top bar both name it, and the real URL is #run=webconference-android">
<figcaption>Real, current data -- captured with <code>webconference-android</code> selected. The URL, the top bar, and every panel title all agree on which run this is.</figcaption>
</figure>

## What this actually gets you

- **Reload the page** and you land back on the exact run the URL names, not an arbitrary one and
  not even last session's choice if the URL says otherwise -- see below for which one wins.
- **Bookmark it.** The bookmark reopens directly on that run.
- **Paste the link to a teammate**, or into another browser tab, and it opens on that same run --
  no "click `webconference-android` in the Runs panel" instructions needed.

## A linked run wins over your last-remembered choice

The app also remembers the last run you had open (a real, separate mechanism, still there for when
you just open the site with no `#run=` at all). If both exist -- a link/bookmark naming one run,
and a different run remembered from your last visit -- **the URL wins**. That's deliberate: a link
was followed or a bookmark was opened on purpose, naming a specific run; silently opening a
different, merely-remembered one instead would be exactly the same "which run am I actually
writing into?" trap [issue #58](https://github.com/scimbe/CADS-devsystem/issues/58) originally
reported, just moved one step later.

If the URL names a run that no longer exists (deleted, or a typo), it's ignored -- the app falls
back to your last real choice, or to no run selected at all, rather than guessing.

## Pasting a link into an already-open tab

If you already have the app open and paste a different `#run=...` link into the same tab's address
bar, it switches to that run immediately -- no reload needed. The same real mechanism handles a
teammate's shared link and your own browser history.

## What this isn't

This doesn't add a full URL-per-panel or full browser back/forward history for every run switch --
selecting a run updates the address in place (so the back button doesn't fill up with one entry per
run you clicked through) rather than adding a new history entry each time. The one thing this closes
is the real gap issue #58 reported: no way to reload, bookmark, or link to a specific run at all.
