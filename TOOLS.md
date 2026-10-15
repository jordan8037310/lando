# Tools and testing standards

How to build, test and lint Lando from this fork. All runtime code is in the
`lando/core` submodule, so nearly every command runs in a core checkout. The
engine work uses a standalone clone of the fork at `~/code/oss/lando-core`.

## Toolchain

| Tool | Version | Why |
|---|---|---|
| Node | **20** (`core/.node-version` = 20.17.0) | Node 22+ breaks mocha's bundled `yargs` (`require is not defined in ES module scope`). On macOS: `brew install node@20` (keg-only, does not replace your default `node`). |
| npm | bundled with Node 20 | `npm ci` from `package-lock.json`; never `yarn` (`yarn.lock` is gitignored). |
| Docker engine | Docker Desktop today; OrbStack / Rancher Desktop under OSSC-60 | Only needed for Leia and manual smoke tests, not for unit tests. |

```bash
export PATH=/opt/homebrew/opt/node@20/bin:$PATH
cd ~/code/oss/lando-core && npm ci
```

## Test suites

| Suite | Command | What it covers | Run locally? |
|---|---|---|---|
| Unit | `npm run test:unit` | Mocha + chai + sinon specs in `test/*.spec.js`, nyc coverage. Baseline 2026-10-14: **291 passing, 37 pending**. | **Yes** — required before every commit. |
| One spec | `npx mocha --timeout 5000 test/daemon.spec.js` | A single file. | Yes |
| Lint | `npm run lint` | ESLint, `google` + `eslint:recommended`, `max-len` 140, JSDoc on function declarations. | **Yes** — required before every commit. |
| Full gate | `npm test` | Lint then unit; what CI's `pr-unit-tests.yml` and `pr-linter.yml` run. | Yes |
| Types | `npm run typecheck` | `tsc` over JSDoc types. Not gated in CI. | Optional |
| Leia | `npm run test:leia` | Executes `examples/**/README.md` as end-to-end specs against a real engine. | **No.** It modifies the host. CI-only (`pr-core-tests.yml`, `pr-setup-*-tests.yml`). |
| Docs | `npm run docs:build` | VitePress build of `docs/`. | Only when editing user docs |

### Standards

- **Every behavior change ships a spec** that fails if the logic breaks. Put it
  in `test/<module>.spec.js`, mirroring the file under test.
- **Stub the engine, never call it.** Unit specs mock `run-command`, `shell`,
  and `fs` with sinon or `mock-fs`; see `test/get-docker-x.spec.js` and
  `test/docker.spec.js` for the pattern.
- **Platform branches need a spec per platform.** Set
  `process.landoPlatform` in the spec rather than relying on the host OS.
- **Leia specs are executable docs.** Editing `examples/**/README.md` changes CI
  behavior.
- **New source directories** must be added to `jsconfig.json`,
  `package.json` `nyc.include`, and `pkg.scripts`, or the packaged binary
  misses them.

## Running the dev build

| Command | Does |
|---|---|
| `scripts/lando-dev <args>` | Runs `~/code/oss/lando-core/bin/lando` on Node 20 with its own config root, `~/.lando-dev`. Stable Lando's `~/.lando` is never read or written. |
| `scripts/lando-switch status` | Shows whether `lando` points at stable or dev. |
| `scripts/lando-switch dev` | Points `~/.lando/bin/lando` at `scripts/lando-dev`, saving the stable target first. |
| `scripts/lando-switch stable` | Restores the saved stable target. |

Overrides: `LANDO_CORE_DIR` (core checkout), `LANDO_DEV_ROOT` (config root),
`LANDO_DEV_NODE` (Node 20 bin dir), `LANDO_PROXY_NAME` (default `landoproxydev`).

**`scripts/lando-dev` doesn't mount your whole home.** Lando bind-mounts its
`home` setting into every container at `/user`. By default that's your entire
home directory, so anything recursive in a container walks all of it through
the engine. On OrbStack that showed up as OrbStack crawling the disk.

`lando-dev` sets `home` to `~/.lando-dev/home` instead. That folder holds only
`.ssh`, hard-linked to `~/.ssh` so no key is copied, and it is refreshed on
every run. SSH keys and git over SSH keep working. Set `LANDO_DEV_FULL_HOME=1`
to mount the real home again.

`scripts/lando-dev` gives the dev build its own proxy, `landoproxydev`. All
Lando 3 installs otherwise share one proxy container,
`landoproxyhyperion5000gandalfedition`. On 2026-10-14, a dev run on Docker
Desktop recreated stable's proxy from dev files, on ports 8000/444.

### Point the dev build at OrbStack

```bash
# one-time: stop OrbStack taking the docker context and /var/run/docker.sock
O=/Applications/OrbStack.app/Contents/MacOS/bin/orbctl
$O config set setup.use_admin false
$O config set docker.set_context false
$O config set network.https false
$O stop && $O start
# the very first start switches the context before these apply: check and reset it
grep currentContext ~/.docker/config.json      # must be desktop-linux

echo 'engineProvider: orbstack' > ~/.lando-dev/config.yml
cd <app> && scripts/lando-dev start
docker --context orbstack ps                    # the app's containers live here
```

### Move a project's data to the new engine

`scripts/lando-migrate <app> <from-context> <to-context> [--images] [--force]`
copies one app's volumes, and optionally its images, between engines. For
example: `lando-migrate myapp desktop-linux orbstack --images`.

- **It only copies.** The source is never modified.
- **It refuses while the app runs** on either engine.
- **Each volume is verified** by a content checksum on both sides.
- **Volumes keep their compose labels,** so Lando adopts them on the next
  `lando-dev start`. Verified 2026-10-14: a Drupal site seeded on Docker
  Desktop came up on OrbStack with its data. Three volumes took 13 s; a 904 MB
  image took 8 s to save and load.
- **Image IDs differ across engines on purpose.** OrbStack uses the
  containerd image store. Compare `RootFS.Layers` instead of IDs.
- **Use it instead of `orb docker migrate`** for client work. OrbStack's tool
  copies everything from Docker Desktop at once, with no per-project filter,
  and refuses if OrbStack already holds data unless forced.

Use `engineProvider: rancher` for Rancher Desktop.

### Podman (opt-in, slower)

```bash
brew install podman
podman machine init --rootful --now          # rootful: docker-like bind-mount ownership
echo 'engineProvider: podman' > ~/.lando-dev/config.yml
```

`auto` never picks Podman. On macOS its port forwarder can't bind ports
below 1024 on loopback, so the proxy serves on `:8000` / `:4433`, for example
`https://myapp.lndo.site:4433`. The proxy runs with `label=disable` because
the Podman VM enforces SELinux.

### Engine comparison on this Mac (M3 Max, 2026-10-15)

Round-robin, 5 rounds per engine, with all three engines up under the same
background load:

| | OrbStack | Rancher | Podman |
|---|---|---|---|
| Cold engine boot | 1.7 s | 22.1 s | 10.3 s |
| Bind-mount read (median) | 10.8 s | 23.8 s | 39.6 s |
| Small-file write (median) | 1.7 s | 3.5 s | 3.4 s |
| `lando start` with the engine stopped | 20 s | 25 s | 46 s | It needs the dockerd (moby)
runtime and Administrative Access off.

Stable Lando (`v3.26.2`, installed at `~/.data/lando/v3.26.2/lando`) stays
installed. Running both is safe while they share an engine; the risk starts when
a second engine takes over `/var/run/docker.sock` — see
`docs/engine/plan.md` → *Isolation protocol*.

## Engines

| Engine | Installed | Socket | Context |
|---|---|---|---|
| Docker Desktop | `/Applications/Docker.app` | `~/.docker/run/docker.sock` (via `/var/run/docker.sock`) | `desktop-linux` |
| OrbStack 2.2.3 | `brew install --cask orbstack` | `~/.orbstack/run/docker.sock` | `orbstack` |
| Rancher Desktop 1.24.0 | `brew install --cask rancher` | `~/.rd/docker.sock` (dockerd mode) | `rancher-desktop` |

Check the live state before and after engine work:

```bash
readlink /var/run/docker.sock   # expect ~/.docker/run/docker.sock
docker context show             # expect desktop-linux
```

## CI (upstream `lando/core`)

- `pr-unit-tests.yml` — unit specs on macOS 15, Ubuntu 24.04 and Windows 2022.
- `pr-linter.yml` — ESLint on the same matrix.
- `pr-core-tests.yml` — the Leia matrix.
- `pr-setup-{linux,macos,windows}-tests.yml` — `lando setup` on each OS.
- `pr-docs-tests.yml` — docs build.
