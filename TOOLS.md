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
`LANDO_DEV_NODE` (Node 20 bin dir).

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
