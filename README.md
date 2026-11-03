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

## Upgrading a ship that is already running

For a comet booted from an earlier release (everything up to
`groundwire-alpha-2026.11.3`, which still shipped `tcp-sidecar`), moving to
the releases published here. This is a **runtime** upgrade: it replaces
`gw-vere`, `boot.sh`, the supervisor and the miner. It does not touch the
ship's kernel or desks. Those live inside the pier, and no comet has an OTA
source for `%base` (checked on two live comets on 2026-11-03: `kiln/sources`
has no `%base` entry), so the sponsor's kernel deploys reach nobody. An
upgraded ship therefore keeps its old `%node`, which still needs its
`tcp-sidecar` process: leave that process running. The new `boot.sh` neither
starts nor stops it, and the new tarball does not carry it, so after a host
reboot start it by hand from `bin/tcp-sidecar`. The `%mach` vane and the
sidecar-free `%node` come only with a new pier today.

The one rule: the runtime may only change over a current snapshot. vere
refuses to start a pier whose snapshot is behind its event log when the
binary changed (`stale snapshot, downgrade runtime to replay`), and the old
`boot.sh --stop` produces exactly that state: SIGTERM, a 30 s grace, then
SIGKILL, and a syncing ship does not exit in 30 s.

1. Stop the ship by hand, under the old runtime. Stop the supervisor first
   (`kill $(cat var/sup-*.lock/pid)`), then `kill -TERM` the `gw-vere -t`
   process and wait until both `gw-vere` processes are gone. Minutes are
   normal. Never `kill -9`.
2. If it was ever SIGKILLed (the new runtime prints `stale snapshot`), replay
   under the old binary first. Keep the previous `gw-vere` as
   `bin/gw-vere.old` before step 3 (or re-download the previous tarball),
   start it on the pier
   (`bin/gw-vere.old -t --loom 32 --http-port 8080 piers/<ship>`), wait until
   the ship answers on its HTTP port, then stop it as in step 1.
3. Fetch the current `boot.sh` (`causeway/public/boot.sh` on `gw/release` of
   gwbtc/groundwire) and run it against the same directory with an explicit
   `--version` from this repository's releases:

   ```sh
   ./boot.sh --vps --detach --dir <dir> --version groundwire-alpha-2026.11.3_2 --comet <ship>
   ```

   It installs the new binaries, finds the pier ("restarting it, not
   re-creating it"), confirms the identity, starts the supervisor and
   re-seeds peers.
4. Check: the ship log starts over with the new `urbit 4.6-<sha>` line and
   shows `tcp: sidecar connected`; the header count keeps moving. One
   `mach: %born failure` line is expected (the driver is in the runtime, the
   vane is not in the old kernel).

Measured on 2026-11-03 (2026.11.3 to 2026.11.3_2, a synced comet): the
`boot.sh` run took 3 m 40 s, identity confirmed after 25 s, 46 live peers
after 2 m 48 s. The stale-snapshot detour, when it happens, costs about 20
minutes.

