# lefthook-configs

Shared, language-agnostic [lefthook](https://lefthook.dev) hook fragments for
MartinCa repositories. Each file is an independently consumable lefthook
config fragment so projects pull in only the hooks they need — through
lefthook's `remotes:` mechanism, never by vendoring.

## Layout

| Fragment | Hook group | What it does | Tooling required (on PATH) |
| --- | --- | --- | --- |
| `lefthook-shared.yml` | `pre-commit` | Secret-scan the staged diff (betterleaks) and audit staged GitHub Actions workflow files (zizmor) | `betterleaks`, `zizmor` |
| `commit-msg.yml` | `commit-msg` | Enforce Conventional Commits on the commit message | none (POSIX sh + `grep`) |
| `langs/ts.yml` | `pre-commit` | ESLint `--fix` + Prettier `--write` via `lint-ts` and Prettier on JSON/CSS/MD via `format-ts` (suffixed — every language fragment carries a suffixed name in v2.0.0, so any combination composes) | `pnpm`, eslint, prettier |
| `langs/python.yml` | `pre-commit` | Ruff `check --fix` + `format` on staged Python via `lint-python`/`format-python` (suffixed so it composes with `langs/ts.yml`) | `uv` (`uvx ruff`) |
| `langs/go.yml` | `pre-commit` | `gofmt -w` + `goimports -w` on staged Go | `gofmt`, `goimports` |
| `langs/shell.yml` | `pre-commit` | `shfmt -w` + blocking `shellcheck` on staged shell scripts via `format-shell`/`lint-shell` (suffixed so it composes with `langs/ts.yml`) | `shfmt`, `shellcheck` |
| `langs/json.yml` | `pre-commit` | Check-only `jq --indent 2 .` on staged JSON via `check-json` (no Node.js needed; suffixed so it composes with `langs/ts.yml`) | `jq` |

Every fragment carries a top-of-file comment documenting how to consume it
and what it requires.

## Consuming fragments

Add or merge the following into your project's `lefthook.yml`:

```yaml
# lefthook.yml
remotes:
  - git_url: https://github.com/MartinCa/lefthook-configs
    ref: v2.0.0
    configs:
      - lefthook-shared.yml
      - commit-msg.yml
      - langs/ts.yml
      - langs/json.yml
```

- **Pin `ref:`** to a released tag (see [Versioning](#versioning)). Do not use a
  branch — tags are immutable and that is what makes consumer builds
  reproducible.
- **Merge order (verified on lefthook 2.1.12 with `lefthook dump`):**
  `remotes:` fragments merge **over** your `lefthook.yml` — for any command
  property the fragment sets (e.g. `run:`), your `lefthook.yml` value is
  discarded. A same-named `run:` in `lefthook.yml` therefore **loses** to the
  fragment. Your `lefthook.yml` only contributes keys the fragment does
  **not** set (e.g. `root:`), which is what makes the
  [monorepo subdirectory](#monorepo-subdirectory-override) pattern work.
  The `ts`/`python`/`shell`/`json` fragment collision on the `lint`/`format`
  command names is this same merge-order behavior — fixed in v2.0.0 by
  renaming the non-TS fragment commands to suffixed names (see
  [Migration v1 → v2](#migration-v1--v2)).
- **`configs:` ordering — list `lefthook-shared.yml` FIRST (v2.0.1 race fix):**
  hook-level keys are **last-writer-wins** across the `configs:` entries in
  listed order. `lefthook-shared.yml` sets `parallel: true`, while
  `langs/python.yml`/`langs/go.yml` set `parallel: false` to serialize the
  commands that rewrite the same staged files. If `lefthook-shared.yml` is
  listed **after** a language fragment, its `parallel: true` silently discards
  the fragment's `parallel: false` and the serialization race fix is lost —
  verified on lefthook 2.1.12: with shared listed last, 3/5 pre-commit runs
  dropped a fix while exiting 0 (`stage_fixed` re-staged the drifted file).
  Keep the shared fragments first and the language fragments after them (refs: #8).
- Caveat: `lefthook dump` does **not** fetch or sync `remotes:` configs — it
  merges only what lefthook has already fetched. Run `lefthook install -f`
  first; otherwise dump silently shows only the local, unmerged config.
- **`lefthook-local.yml` is the final merge layer and wins over everything,**
  including `remotes:` fragments. Same-named command properties placed there
  override the fragment, while fragment-only keys (`glob:`, `stage_fixed:`)
  are still inherited. Use it for team-wide overrides — see
  [Team-wide npm override](#team-wide-npm-override-npm-only-repos) and
  [Monorepo subdirectory override](#monorepo-subdirectory-override).
- Gotcha: `lefthook validate` does not merge `remotes:` fragments, so a
  partial local override (e.g. only `root:`) fails validation — on lefthook
  2.1.12 the output is `run: Value is null but should be string` and
  "validation failed for main config". Note validate *does* read
  `lefthook-local.yml`: lefthook merges that file into the main config, so it
  is not only inspecting `lefthook.yml`. The validation failure is expected —
  verify merged behavior with `lefthook dump` (after `lefthook install -f`)
  or `lefthook run <hook>`.

## Installing lefthook in a consumer project

Lefthook only *executes* hooks after `lefthook install` has registered them
via the corresponding git hook files (`.git/hooks/pre-commit`,
`.git/hooks/commit-msg`, ...). Two supported ways to get it:

**Node-based projects** (JS/TS repos) — add it as a devDependency so the
binary lands in the project and `prepare` runs automatically on install:

```jsonc
// package.json
{
  "scripts": { "prepare": "lefthook install" },
  "devDependencies": { "lefthook": "2.1.12" }
}
```

Installed via npm/pnpm, `pnpm install` runs `prepare` → `lefthook install` →
hooks are (re)installed on every fresh checkout. Keeping `lefthook` in
`devDependencies` also means the version is bumped by your dependency
manager/renovate like any other tool.

**Non-JS projects** (Python, Go, shell, ...) — install the standalone binary
and run `lefthook install` once per clone. For example, `uv` (`uvx`):

```console
$ uvx lefthook@2.1.12 install
```

or the official install script:

```console
$ curl -fsSL https://get.lefthook.io/install.sh | bash -s 2.1.12
```

## Team-wide npm override (npm-only repos)

`langs/ts.yml` invokes eslint/prettier through `pnpm` (the frontend-kit
convention). An npm-only repo has no pnpm: running `pnpm <cmd>` makes pnpm
resolve the tree and write a `pnpm-lock.yaml` next to `package-lock.json`,
or fails outright when pnpm is not on PATH. Because `remotes:` fragments win
over `lefthook.yml`, the fix lives in a committed `lefthook-local.yml` — the
one layer that overrides remotes — overriding `run:` with
`npx --no-install`:

```yaml
# lefthook-local.yml
pre-commit:
  commands:
    lint-ts:
      run: npx --no-install eslint --fix {staged_files} && npx --no-install prettier --write {staged_files}
    format-ts:
      run: npx --no-install prettier --write {staged_files}
```

This replaces the fragment's `run:` while inheriting its `glob`/`stage_fixed`.
Reference implementation: [frontend-kit's committed `lefthook-local.yml`](https://github.com/MartinCa/frontend-kit/blob/main/lefthook-local.yml)
(frontend-kit additionally excludes `test/fixtures/**` — repo-specific;
still pinned to v1.0.1, so its override keys are the old `lint`/`format` —
rename to `lint-ts`/`format-ts` when bumping to v2.0.0).
Note the lefthook convention: `lefthook-local.yml` is normally *personal and
untracked* (it is even gitignored in this repo). Teams that commit it for a
team-wide override should say so in their own docs — frontend-kit does, in
its `AGENTS.md`.

## Monorepo subdirectory override

Fragments deliberately carry no `root:`. A repo whose TS lives in a
subdirectory (e.g. `client/`, `frontend/`) scopes the hooks there with the
same mechanism — a committed `lefthook-local.yml` adding `root:` to the
`lint-ts`/`format-ts` commands:

```yaml
# lefthook-local.yml
pre-commit:
  commands:
    lint-ts:
      root: "client/"
    format-ts:
      root: "client/"
```

`root:` merges into the fragment's `lint-ts`/`format-ts` commands, so
ESLint/Prettier only see files under `client/` (lefthook re-bases the staged
paths relative to the command root) — verified via `lefthook dump`. The
`root:` keys survive the merge because the fragment does not set `root:`; a
same-named key like `run:` would not. If the subdirectory override must also
change `run:` (npm repos), combine both forms in the same `lefthook-local.yml`
as in the [Team-wide npm override](#team-wide-npm-override-npm-only-repos)
example above.

Real consumers of this committed root-override pattern:
[audiobook-manager](https://github.com/MartinCa/audiobook-manager) uses
`root: "client/"` ([PR #1427](https://github.com/MartinCa/audiobook-manager/pull/1427))
and [prowlarr-watcher](https://github.com/MartinCa/prowlarr-watcher) uses
`root: "frontend/"` ([PR #126](https://github.com/MartinCa/prowlarr-watcher/pull/126)),
both via a committed `lefthook-local.yml`.

## Versioning

Releases are SemVer tags pushed to this repo's `main`; consumers pin `ref:`:

- **`major`** — breaking change: altered behavior of an existing fragment
  (e.g. a previously advisory check starts failing commits, a command's
  semantics change). Consumers must re-review before upgrading.
- **`minor`** — a new fragment or an additive,
  backwards-compatible change (new glob, new optional command).
- **`patch`** — a non-breaking bugfix to an existing fragment: a broken or
  incorrect command, wrong documentation, or a false-positive fix. Consumers
  should bump promptly.

### Migration v1 → v2

v2.0.0 renames the collision-prone `langs/*.yml` pre-commit command names to
fix [issue #5](https://github.com/MartinCa/lefthook-configs/issues/5):
lefthook merges same-named commands across `configs:` entries key-by-key, so
consuming e.g. `langs/ts.yml` and `langs/python.yml` together silently
dropped the TS hooks (and `langs/json.yml`'s `format` inherited `stage_fixed`
from `langs/ts.yml`). All language fragments now use suffixed names:

- `langs/ts.yml`: `lint` → `lint-ts`, `format` → `format-ts`;
- `langs/python.yml`: `lint` → `lint-python`, `format` → `format-python`;
- `langs/shell.yml`: `lint` → `lint-shell`, `format` → `format-shell`;
- `langs/json.yml`: `format` → `check-json` (the command only checks — the
  old `format` name was misleading on top of colliding).

Because the TS names changed too, **every** consumer bumping `ref:` from
v1.0.x must update by-name references:

- local `lefthook run lint`/`lefthook run format`-style invocations and
  skips/overrides by name in `lefthook-local.yml` (e.g. `lint: {skip: ...}`)
  must use the new names: `lint-ts`/`format-ts`, `lint-python`/`format-python`,
  `lint-shell`/`format-shell`, `check-json`;
- committed `lefthook-local.yml` overrides keyed on `lint`/`format` for the
  TS hooks need their keys renamed. Known consumer:
  [frontend-kit](https://github.com/MartinCa/frontend-kit), whose committed
  `lefthook-local.yml` overrides `lint`/`format` — it must update those
  keys to `lint-ts`/`format-ts` when bumping;
- mixed-language repos that worked around the collision — by re-declaring the
  TS commands under distinct local names, or by maintaining their own
  suffixed overrides — can now consume any combination of `langs/*.yml`
  directly and drop those workarounds.

### Automated ref bumps (Renovate)

Consumers using Renovate can bump `ref:` automatically with a regex manager:

```jsonc
// renovate.json
{
  "regexManagers": [
    {
      "fileMatch": ["(^|/)lefthook\\.ya?ml$"],
      "matchStrings": ["ref: (?<currentValue>v[0-9]+\\.[0-9]+\\.[0-9]+)\\s*$"],
      "depNameTemplate": "lefthook-configs",
      "packageNameTemplate": "MartinCa/lefthook-configs",
      "datasourceTemplate": "github-tags"
    }
  ]
}
```

## Development

- Validate a fragment locally with any lefthook ≥ 2.1.12:
  `LEFTHOOK_CONFIG=langs/ts.yml lefthook validate`
- `langs/ts.yml` mirrors the inline config previously used by
  `frontend-kit`; the other fragments follow the same shape. New fragments
  must be valid standalone configs (own hook-group key) and should prefer
  zero-dependency defaults where reasonable (see `langs/json.yml` for the
  check-only, no-Node trade-off).
- CI runs zizmor on this repo's workflows and validates that every fragment
  still parses (`lefthook validate`) and that every YAML file is valid.