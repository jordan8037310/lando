# CLAUDE.md — agentic contribution to Lando

This is `jordan8037310/lando`, a fork of `lando/lando`. It is a **meta-repo**:
no runtime code lives here; every component is a git submodule under `lando/*`.
The current initiative is a pluggable container engine (OrbStack, Rancher
Desktop) — epic **OSSC-60** in Facet Jira.

## Read first

| When you need | Read |
|---|---|
| How Lando reaches its engine, and the target design | [ARCHITECTURE.md](ARCHITECTURE.md) |
| Why OrbStack first, and every known incompatibility | [docs/engine/due-diligence.md](docs/engine/due-diligence.md) |
| Phases, tickets, isolation protocol, smoke checklist | [docs/engine/plan.md](docs/engine/plan.md) |
| Test suites, lint, Node version, dev-build commands | [TOOLS.md](TOOLS.md) |
| Rules for code inside `lando/core` | `core/AGENTS.md` (upstream-maintained) |

`ARCHITECTURE.md` and `docs/engine/` land through their own PR
(`docs/engine-architecture`). Until it merges, read them on that branch.

## Where work happens

| Change | Repo | Branch from |
|---|---|---|
| Engine code (`lib/`, `utils/`, `hooks/`) | `jordan8037310/core`, cloned at `~/code/oss/lando-core` | `feat/engine-provider` |
| User docs (docs.lando.dev) | same core fork, `docs/` | `feat/engine-provider` |
| Plans, architecture, agent setup | this repo | `master` |
| A recipe or service plugin | that plugin's own fork | its `main` |

- **Never commit through a submodule path in this repo.** Submodules here pin
  upstream SHAs; bumping them is an upstream release task.
- **Submodules are not initialised by default.** Clone the fork you need
  instead of `git submodule update --init`, which checks out upstream remotes.
- **Remotes:** `origin` is the fork; `upstream` is `lando/*`. Sync with
  `gh repo sync jordan8037310/<repo> --source lando/<repo>`. Never push to
  `upstream`.

## Standards (from the repo's own config)

- **`.editorconfig`:** LF, final newline, UTF-8; JS indented 2 spaces.
- **`.gitattributes`:** LF enforced for `*.js *.sh *.conf *.cnf *.ini *.php *.vcl`.
- **ESLint (core):** `google` + `eslint:recommended`, `max-len` 140,
  `arrow-parens: as-needed`, JSDoc required on function declarations.
- **Types:** add JSDoc `@param`/`@return` to any function you touch; shared
  typedefs go in a co-located `*.types.js` (core `AGENTS.md`).
- **Changelog:** user-facing changes only, under `## {{ UNRELEASED_VERSION }}`
  in core's `CHANGELOG.md`, matching existing bullet style.
- **Commits:** Conventional Commits — `fix(daemon): …`, `feat(engine): …`,
  `docs(engine): …`. Reference the OSSC key in the body.
- **PRs upstream:** one concern per PR, smallest first (OSSC-64 before
  OSSC-65). Upstream PRs are opened only after Jordan reviews.

## Verification gate (every commit)

```bash
export PATH=/opt/homebrew/opt/node@20/bin:$PATH
cd ~/code/oss/lando-core && npm test   # lint + unit; baseline 291 passing
```

- **Never run Leia locally** (`npm run test:leia`); it modifies the host. CI only.
- **Every behavior change ships a spec** that fails when the logic breaks.

## Machine safety

This Mac runs live client apps on stable Lando + Docker Desktop.

- **Do not change the global docker context** or `/var/run/docker.sock`.
  `.claude/settings.json` denies both.
- **Use `scripts/lando-dev`** for the dev build. It runs with its own config
  root (`~/.lando-dev`), so it never touches stable's `~/.lando`.
- **Start OrbStack or Rancher only under the isolation protocol** in
  `docs/engine/plan.md`, with socket takeover off.
- **`scripts/lando-switch dev|stable`** moves the `lando` command between
  builds. It asks before running.

## Tracking

- **Jira:** OSSC project. Epic OSSC-60, children OSSC-61 – OSSC-70. Comment
  proof-of-work (commit SHAs, PR links, spec names) on the ticket you worked.
- **Time:** Harvest project *Open Source Software Contributions*. Log a
  quarter hour per agentic hour.
- **Loop:** OSSC-64 → OSSC-69 run through `/superloop`, one ticket per tick.
  The full adversarial review runs once, at `feat/engine-provider` → fork
  `main`.
- **Out of scope for now:** a story map of Lando features.
