# gwbtc/release

What goes into a Groundwire release, and the CI that builds it.

Every input (the kernel, the runtime, each desk, the sidecar, the miner, the
onboarding script) is pinned **by commit** in the `env` block of
[`.github/workflows/release.yml`](.github/workflows/release.yml). One commit
to this repo is one complete, reproducible description of a release.

## Cutting a release

1. `./bump` shows every pin beside the tip of its usual branch.
2. `./bump NODE_REF` (or `./bump GROUNDWIRE_BRANCH gw/release`, or a bare
   commit) moves one pin. Move as many as the release needs.
3. Open a pull request. Its build is the test: the pill is built from the
   PR's pins and booted, and nothing is published.
4. Merge. The push to `main` builds again and publishes a **prerelease**
   tagged `groundwire-alpha-<date>` here, with the three tarballs and
   `SHA256SUMS`.
5. Promote: when a prerelease has been tried on a real comet, clear its
   prerelease flag (`gh release edit <tag> --prerelease=false`). GitHub then
   serves it at `releases/latest`, which is what the website's installer
   follows. Nobody has to promote for a merge to happen.

A test build from main's pins without publishing:
`gh workflow run release.yml -f publish=false`; the run's artifacts hold the
tarballs. The `*_ref` inputs override a pin for that run only.

## What is coupled at runtime

Two desks compile apart and only fail together on a live ship: `%node`
(`NODE_REF`) and `%gw-btc` (`GROUNDWIRE_BRANCH`) must agree on the shapes of
the facts between them, and the first reorg is where a mismatch shows. Move
them together, and move the distro ship's published desks from the same pins.

No secrets: every pinned repository is public, and the release is created in
this repository with the workflow's own token.
