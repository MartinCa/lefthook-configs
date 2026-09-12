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
| `langs/ts.yml` | `pre-commit` | ESLint `--fix` + Prettier `--write` on staged TS/JS and Prettier on JSON/CSS/MD | `pnpm`, eslint, prettier |
| `langs/python.yml` | `pre-commit` | Ruff `check --fix` + `format` on staged Python | `uv` (`uvx ruff`) |
| `langs/go.yml` | `pre-commit` | `gofmt -w` + `goimports -w` on staged Go | `gofmt`, `goimports` |
| `langs/shell.yml` | `pre-commit` | `shfmt -w` + blocking `shellcheck` on staged shell scripts | `shfmt`, `shellcheck` |
| `langs/json.yml` | `pre-commit` | Format-check staged JSON against `jq --indent 2 .` (no Node.js needed; check-only, deliberately) | `jq` |

Every fragment carries a top-of-file comment documenting how to consume it
and what it requires.

## Consuming fragments

Add or merge the following into your project's `lefthook.yml`:

```yaml
# lefthook.yml
remotes:
  - git_url: https://github.com/MartinCa/lefthook-configs
    ref: v1.0.0
    configs:
      - lefthook-shared.yml
      - commit-msg.yml
      - langs/ts.yml
      - langs/json.yml
```

- **Pin `ref:`** to a released tag (see [Versioning](#versioning)). Do not use a
  branch — tags are immutable and that is what makes consumer builds
  reproducible.
- Each `configs:` entry is merged as a separate config. Hook groups and
  commands of the same name merge **key-by-key** (deep merge) with your local
  `lefthook.yml`, so you can tweak a single setting — like `root:` — while
  keeping the fragment's `run`/`glob` (this is how the
  [audiobook-manager](#root-override-for-monorepo-subdirectories) pattern
  below works).
- Gotcha: `lefthook validate` only inspects the *local* `lefthook.yml` and
  does not merge `remotes:` configs. A local partial override (e.g. only
  `root:`) will therefore be reported as "missing `run`" by `lefthook
  validate`. That is expected — verify merged behavior with
  `lefthook run <hook>`.

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

## Root override for monorepo subdirectories

Fragments deliberately carry no `root:`. Consumers scoping a hook to a
subdirectory (monorepos) add `root:` locally. Example: `audiobook-manager`
has its app under `client/` and scopes the TS hooks there:

```yaml
# audiobook-manager/lefthook.yml
remotes:
  - git_url: https://github.com/MartinCa/lefthook-configs
    ref: v1.0.0
    configs:
      - lefthook-shared.yml
      - commit-msg.yml
      - langs/ts.yml

pre-commit:
  commands:
    lint:
      root: "client/"
    format:
      root: "client/"
```

The two `root:` keys merge into the fragments' `lint`/`format` commands, so
ESLint/Prettier only see files under `client/` (lefthook re-bases the staged
paths relative to the command root). Note that `lefthook validate` alone will
flag `lint`/`format` here as missing `run` because it validates the local file
without merging the remote configs — use `lefthook run pre-commit` to verify.

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