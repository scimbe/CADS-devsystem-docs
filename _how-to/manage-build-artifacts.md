---
title: Upload, download, and remove a build artifact
description: Attach a real, checkable build output to a run, and trace it back to the iteration and commit that produced it.
order: 17
---

# Upload, download, and remove a build artifact

An iteration's `feedback` is free text -- nothing stopped a claim like "APK built, sha256 abc123,
commit deadbeef" from being marked `succeeded: true` with no way for anyone to actually check it.
The **Build Artifacts** panel gives a run a real, downloadable file with a server-computed hash,
traceable to the exact iteration that produced it -- not just a sentence someone has to trust.

## A run with nothing uploaded yet

Open the panel (via the panel launcher, if it isn't already visible -- like Docs Search and Custom
Panels, it isn't force-opened on a first-time user) and the upload form is already expanded, since
there's only one real thing to do here first:

<figure>
<img src="{{ '/assets/img/howto-manage-build-artifacts/01-empty-state.png' | relative_url }}" alt="The Build Artifacts panel on a run with nothing uploaded yet, showing the auto-expanded upload form">
<figcaption>No build artifacts uploaded yet -- the upload form is open by default, the same auto-expand precedent the Docs Search panel's own empty state already established.</figcaption>
</figure>

## Uploading real, traceable metadata

Choose a file (up to 150 MB, up to 10 artifacts per run), then fill in what the upload actually
needs:

- **Produced by iteration** is a real dropdown built from this run's own history -- you pick an
  iteration that actually happened, you can't type in an arbitrary number. The server independently
  re-checks this against the run's real history too, so this isn't just a client-side nicety.
- **Producing stage** is free text (e.g. `devsystem.android_native_build_ci`).
- **Source commit**, **version name**, and **version code** are all optional, but every one you
  fill in becomes a real, checkable field on the artifact record afterward -- not decoration.

You never type in a hash. `sha256` is always computed server-side from the actual bytes you upload,
shown afterward on the artifact itself:

<figure>
<img src="{{ '/assets/img/howto-manage-build-artifacts/02-uploaded-with-metadata.png' | relative_url }}" alt="The Build Artifacts panel showing one real uploaded artifact with its filename, server-computed sha256, source iteration and stage, version, and commit, plus the still-open upload form for a second one">
<figcaption>A real upload, with every field shown exactly as the server recorded it -- the sha256 here is independently verifiable against the actual file (try <code>sha256sum</code> on whatever you download).</figcaption>
</figure>

## Downloading and removing

**Download** links to the real file, byte-identical to what was uploaded -- no separate export
step, no regenerating anything. **Remove** asks for confirmation first, the same permanent-delete
discipline every other destructive action in this GUI already uses: this deletes the real file, not
just the row that describes it. There's no undo.

## No GUI listing without uploading anything here first

Repo-synced content and this panel are entirely separate -- an artifact only ever gets here by a
real upload, either through this panel or `POST /api/runs/{id}/artifacts` directly. See the
[REST API reference]({{ '/reference/rest-api/' | relative_url }}#build-artifacts) for the full
request/response shape, including the endpoints this panel itself calls.
