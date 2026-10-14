# Engine provider — plan

Epic: **OSSC-60** — Engine.Provider: run Lando on OrbStack or Rancher Desktop
instead of Docker Desktop. Architecture is in
[ARCHITECTURE.md](../../ARCHITECTURE.md); evidence is in
[due-diligence.md](due-diligence.md).

## Repositories and branches

| Repo | Branch | Holds |
|---|---|---|
| `jordan8037310/lando` | `docs/engine-architecture` | This plan, architecture, due diligence |
| `jordan8037310/lando` | `feat/agentic-hydration` | `CLAUDE.md`, `TOOLS.md`, `.claude/settings.json`, `scripts/lando-switch` |
| `jordan8037310/core` | `feat/engine-provider` (to create) | All code changes, one commit per ticket |

The `lando/lando` fork's `master` matched upstream `main` on 2026-10-14
(0 ahead, 0 behind). `lando/core` was forked the same day.

## Phases and tickets

| Phase | Ticket | Work | Gate to next phase |
|---|---|---|---|
| 0. Analysis | OSSC-61 | Coupling map, due diligence | ✅ Done 2026-10-14 |
| 0. Hydration | OSSC-62 | Agent setup, `lando-switch`, Node 20 toolchain | PR merged on the fork |
| 0. Decision | OSSC-63 | OrbStack Pro seats (commercial licence) | **Jordan decides.** The trial covers phases 1–2. |
| 1. Fix the crash | OSSC-64 | `null` version → `skip` (core#138) | Unit specs green; smallest upstreamable PR |
| 1. Provider | OSSC-65 | `utils/get-engine-provider.js`, `engine` config key | Docker Desktop path unchanged (specs) |
| 1. One socket | OSSC-66 | Export `DOCKER_HOST` for every socket | Spec: compose subprocess gets provider socket |
| 1. Setup | OSSC-67 | No Docker Desktop install or warnings on other engines | Specs green |
| 2. OrbStack | OSSC-68 | Isolated smoke + benchmark vs Docker Desktop | Drupal recipe passes the checklist below |
| 2. Rancher | OSSC-69 | Same on Rancher Desktop (dockerd mode) | Same checklist; SSH agent result recorded |
| 3. Docs | OSSC-70 | Update `core/docs` user pages | Upstream PRs opened |

Phase 1 needs no running alternative engine — it is unit-tested code. Phase 2
is the first time OrbStack or Rancher is started on this Mac.

## Isolation protocol for phase 2

Stable Lando keeps running on Docker Desktop throughout.

1. In OrbStack settings, **turn off** "Use admin privileges" before the first
   start, so it does not repoint `/var/run/docker.sock`. In Rancher Desktop,
   leave **Administrative Access** off and choose the **dockerd (moby)**
   runtime.
2. Do **not** run `docker context use orbstack`. The dev build gets its socket
   from its own config, not from the global context.
3. Run the dev build with its own config root:
   `LANDO_CORE_USERCONFROOT=~/.lando-dev`, where `~/.lando-dev/config.yml`
   sets `engine` (or `engineConfig.socketPath`) and `setup.buildEngine: false`.
4. Before and after each session, confirm
   `readlink /var/run/docker.sock` still points at
   `~/.docker/run/docker.sock` and `docker context show` is `desktop-linux`.

## Smoke checklist (OSSC-68, OSSC-69)

- `lando start` on a Drupal recipe completes with no Docker Desktop warning.
- `lando ssh -c 'id'` shows the expected UID; files in `/app` are writable.
- `lando drush status` and `lando db-import` succeed.
- SSH keys load into the container (core#53).
- `host.lando.internal` resolves from a container.
- `https://<app>.lndo.site` serves through the proxy.
- `lando poweroff` leaves Docker Desktop apps untouched.

## Benchmark (OSSC-68)

Same app, same machine, three runs each, median reported:

- Cold `lando start` after `lando poweroff`.
- `lando rebuild -y`.
- A bind-mount-heavy command: `lando composer install` with an empty `vendor/`.
- Engine idle memory after 5 minutes (Activity Monitor, VM process).

## User docs to update (OSSC-70)

These live in `lando/core/docs` (docs.lando.dev) and change only after the
code lands upstream:

- `docs/guides/how-do-i-use-orbstack.md` — drop the `Docker.app` symlink hack.
- `docs/getting-started/requirements.md` — list supported engines.
- `docs/cli/setup.md` — `setup.buildEngine: false` and engine detection.
- `docs/config/global.md` — the `engine` key.
- New guide: Rancher Desktop.

## Looping

After the review of this plan, OSSC-64 through OSSC-69 run through
`/superloop`: one ticket per tick, a feature branch per ticket off
`feat/engine-provider` in the core fork, and the functional gate (acceptance
criteria met, specs that fail when the logic breaks) before merge. The full
adversarial review runs once, on `feat/engine-provider` → `main` of the fork,
before anything is offered upstream.
