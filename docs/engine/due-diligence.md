# Engine due diligence — OrbStack vs Rancher Desktop

Researched 2026-10-14 against `@lando/core` 3.26.9 (`main`). Source line
references were checked against that checkout. Items marked *unverified* are
inferred and must be proven on a live engine before we rely on them.

## Recommendation

**Try OrbStack first. Keep Rancher Desktop as the open-source fallback.**

- **OrbStack is fastest for Lando's workload.** Lando 3 has no Mutagen, so app
  code is a raw VirtioFS bind mount. OrbStack is the only engine benchmarked
  fast with file sync off.
- **OrbStack already mimics Docker Desktop where Lando depends on it.** It
  provides `/run/host-services/ssh-auth.sock` and can own
  `/var/run/docker.sock`.
- **OrbStack costs money for us.** Commercial use, freelancers, and anyone
  earning over $10k a year from related work need Pro — about $8 per user per
  month. Facet needs a paid seat per developer.
- **Rancher Desktop is Apache-2.0 and free**, but it is slower without Mutagen
  and lacks the SSH agent socket.

## Performance

The primary independent source is DDEV's nightly Drupal benchmark on Apple
Silicon, all providers on VZ + VirtioFS. These are medians for
2026-08-31 – 2026-09-30, from
[ddev.com/blog/docker-performance-2023](https://www.ddev.com/blog/docker-performance-2023),
updated 2026-10-06.

| Provider | Mutagen | Drupal install | Cold start |
|---|---|---|---|
| Rancher Desktop | on | 12.0 s | 16.5 s |
| OrbStack | on | 13.4 s | 12.2 s |
| OrbStack | **off** | 16.8 s | **9.2 s** |
| Docker Desktop | on | 16.7 s | 21.0 s |

- **Mutagen off is Lando's case**, and only OrbStack was measured that way.
  DDEV's Nov 2023 run found OrbStack "by far the fastest" with Mutagen off and
  Rancher and Colima "pretty sluggish".
- **Vendor claims are marketing.** OrbStack advertises "75–95% of native" file
  I/O ([orbstack.dev/blog/fast-filesystem](https://orbstack.dev/blog/fast-filesystem)).
- **No independent idle memory or CPU numbers exist** for either engine. We
  will measure our own — see the benchmark plan in [plan.md](plan.md).
- **Our own measurement (2026-10-14, this Mac)** confirms the direction.
  OrbStack is 2.8× faster on bind-mount reads and 4.2× faster on small-file
  writes than Docker Desktop. Numbers and caveats are in
  [architecture.md](../../architecture.md#live-result-2026-10-14-ossc-68).

## Risks

| Risk | OrbStack | Rancher Desktop |
|---|---|---|
| License | Proprietary; paid for commercial use | Apache-2.0 |
| Socket takeover | With admin access, symlinks `/var/run/docker.sock` to itself and creates context `orbstack` | Only with "Administrative Access"; otherwise `~/.rd/docker.sock`, context `rancher-desktop` |
| Coexistence with Docker Desktop | Fights over `/var/run/docker.sock`; `credsStore: desktop` in `~/.docker/config.json` can break pulls | Same socket fight when admin access is on; macOS deletes the socket at boot, so it re-prompts for a password |
| Runtime mode | dockerd only | **Must use dockerd (moby) mode** — containerd mode cannot run Lando |
| Ports | `*.orb.local` routes to container IPs; should not claim host 80/443 (*unverified*) | Admin-access vmnet takes port 53 |

### Risk to the stable Lando install on this machine

On 2026-10-14 this Mac runs ten Lando containers on Docker Desktop
(`facetcrm`, `ffxdbt`, `cfops`, and the proxy). `/var/run/docker.sock` points
at Docker Desktop.

- **Starting OrbStack or Rancher can repoint `/var/run/docker.sock`.** Stable
  Lando would then talk to an engine where those apps do not exist. Volumes are
  not lost, but apps look destroyed until the socket is switched back.
- **Mitigation:** run the dev build with its own config root and an explicit
  socket, and turn off socket takeover in both apps. This is how
  `scripts/lando-switch` and the isolated runtime work — see [TOOLS.md](../../TOOLS.md).

## Incompatibilities in lando/core

These come from the source. C-numbers match [architecture.md](../../architecture.md).

1. **C1 — `null` version crash** (core#138). `lib/daemon.js` reads
   `Docker.app/Contents/Info.plist`; without it `getVersions()` returns
   `desktop: null`, and `utils/is-compatible-version.js:11`
   (`semver.clean(null)`) throws `reading 'trim'`. `daemon.up()` then calls
   `semver.gte(null, …)` and also throws. A running engine skips this, because
   `isUp()` returns early.
2. **C2 — start command.** Only `docker desktop start` or
   `open /Applications/Docker.app`.
3. **C3 — setup installs Docker Desktop.**
   `hooks/lando-setup-build-engine-darwin.js:110` reports not-installed
   without Docker.app. `setup.buildEngine: false` skips it (line 90).
4. **C5 — split brain.** `engineConfig.socketPath` steers dockerode only. The
   `docker` and `docker compose` subprocesses follow the active docker context
   because `DOCKER_HOST` is exported only for a non-`127.0.0.1` TCP host
   (`utils/get-engine-config.js:25`).
5. **C6 — spurious warnings** from Docker Desktop version ranges.
6. **C7 — SSH agent.** `packages/ssh-agent/ssh-agent.js:16` hardcodes
   `/run/host-services/ssh-auth.sock`. OrbStack provides it; Rancher does not
   ([rancher-desktop#2072](https://github.com/rancher-sandbox/rancher-desktop/issues/2072)).
7. **C8 — file ownership.** Key-file group differs per engine (core#53). The
   `keys:` workaround fixed OrbStack for one user. Treat it as *unverified*.
8. **`host.lando.internal`** uses `extra_hosts: host-gateway`
   (`builders/_lando.js:198`). Expected to work on both (*unverified*).
9. **Proxy and DNS.** Traefik binds `127.0.0.1:80/443`; `*.lndo.site` is
   public DNS to `127.0.0.1`. No engine dependency.

## Workarounds users already rely on

- **OrbStack:** run Docker Desktop once, quit it, then use OrbStack (core#138).
  Or `sudo ln -s /Applications/OrbStack.app /Applications/Docker.app` — the
  official guide's hack. That reports OrbStack's version as Docker Desktop's
  and still triggers C6 warnings.
- **Colima / Rancher:** `engineConfig.socketPath` in `~/.lando/config.yml`
  (lando#3388).

## Podman (added 2026-10-15, OSSC-73)

**Verdict:** it works as an opt-in provider, but it's the slowest of the
three. Keep it for teams that can't use OrbStack or Docker Desktop.

### Published data

- **DDEV's nightly benchmark** (independent; 2026-09-15 to 10-14, Mutagen on;
  Podman rootless with applehv):
  - Podman ties the others on Drupal install.
  - It is **3.6× slower on cold start** (45 s vs OrbStack's 12 s) and **2.5×
    slower on rebuild**.
- **Podman 6 defaults to the libkrun provider,** which upstream measured
  about 26% slower than applehv on builds (libkrun#414). On this M3 Max the
  default was applehv.
- **There is no controlled data comparing M1–M4 or Pro/Max chips** for any
  engine. DDEV doesn't record which chip each runner uses.
- **Lando requests for Podman** (lando#3238, core#363) are open with no
  implementation.

### Measured on this Mac (M3 Max)

Round-robin, 5 rounds, all engines under the same background load:

| | OrbStack | Rancher | Podman 6.1.4 (rootful, applehv) |
|---|---|---|---|
| Cold engine boot | **1.7 s** | 22.1 s | 10.3 s |
| Bind-mount read, median | **10.8 s** | 23.8 s | 39.6 s |
| Small-file write, median | **1.7 s** | 3.5 s | 3.4 s |
| `lando start` with the engine stopped | **20 s** | 25 s | 46 s |

### Incompatibilities found and fixed

1. **Privileged ports:** Podman's `gvproxy` publishes ports without root, and
   macOS refuses ports below 1024 on `127.0.0.1` to non-root processes. Fixed
   in core `56bed58b`: providers can declare `unprivilegedPorts`, so the proxy
   stays on loopback and uses 8000/4433.
2. **SELinux:** the Podman VM is Fedora CoreOS with SELinux enforcing, so
   Traefik was denied the docker socket and served 404 for every app. Fixed in
   core `b6d3c0a6`: the proxy gets `label=disable` on providers that declare
   `selinux`.
3. **Socket location:** the default machine's Docker-compatible API socket is
   `$TMPDIR/podman/podman-machine-default-api.sock`. It is deterministic, so
   there is no need to shell out to `podman machine inspect`.

### Still unverified

- **Rootless Podman:** needs `keep-id` UID mapping that Lando's compose files
  don't emit.
- **Compose `links`:** Podman rejects them. Lando's generated compose was not
  audited, but the Drupal recipe works.

## Sources

- [lando/lando#3388](https://github.com/lando/lando/issues/3388) — Rancher Desktop request; maintainer defers pluggable engines to Lando 4 with no ETA.
- [lando/core#138](https://github.com/lando/core/issues/138) — OrbStack `trim` crash.
- [lando/core#53](https://github.com/lando/core/issues/53) — SSH keys on OrbStack and Colima.
- [docs.orbstack.dev/licensing](https://docs.orbstack.dev/licensing), [orbstack.dev/pricing](https://orbstack.dev/pricing).
- [docs.ddev.com — Docker installation](https://docs.ddev.com/en/stable/users/install/docker-installation/).
