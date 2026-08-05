# CADS-devsystem-docs

Documentation for [CADS-devsystem](https://github.com/scimbe/CADS-devsystem) — "The Development
System" ([CADS-Tunnel#382](https://github.com/scimbe/CADS-Tunnel/issues/382)). Jekyll + [Diátaxis](https://diataxis.fr),
matching the same structure and hermetic-build process as
[CADS-Tunnel-docs](https://github.com/scimbe/CADS-Tunnel-docs).

Not yet deployed to a public subdomain — see the operator's own note before doing that
(deliberately not touched here). Build locally:

```
docker run --rm -v "$PWD":/srv/jekyll -e JEKYLL_UID=$(id -u) -e JEKYLL_GID=$(id -g) \
  jekyll/jekyll:4 bash -c 'bundle install && bundle exec jekyll build --trace'
```

## Process

Every procedural claim (screenshots, described flows) is verified against the real, live
deployment before being written down — see [`_tutorials/first-run.md`](_tutorials/first-run.md)
for an example: writing it found and fixed a real bug in the GUI itself
([CADS-devsystem `56182ea`](https://github.com/scimbe/CADS-devsystem/commit/56182ea)).
