# xraph/workflows

Reusable GitHub Actions workflows for the xraph org. The Go track is complete;
Rust, Node/TypeScript and Dart/Flutter tracks land beside it, sharing the same
`semantic-release.yml` spine.

Consumers pin to the moving major tag. A caller must declare the permissions its
workflow needs — a called workflow can only *narrow* the caller's token, never
widen it — and should pass secrets explicitly rather than with `secrets: inherit`:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read
  security-events: write   # required for the gosec SARIF upload

jobs:
  ci:
    uses: xraph/workflows/.github/workflows/go-ci.yml@v1
    with:
      go-versions: '["1.25","1.26"]'   # must not list a version below your go.mod directive
    secrets:
      CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
```

## Workflows

| workflow | purpose |
|---|---|
| `go-ci.yml` | test matrix, lint, verify, security |
| `go-release.yml` | tag-push and manual-dispatch releases |
| `go-binary-release.yml` | GoReleaser cross-platform binaries |
| `codeql.yml` | CodeQL analysis for Go |
| `rust-ci.yml` | lint, security audit, build + test for Rust |
| `node-ci.yml` | one install, then the repo's own typecheck, lint, test and build scripts |
| `npm-publish.yml` | ordered npm publishing, with a pnpm workspace mode |
| `semantic-release.yml` | conventional-commit versioning and GitHub releases |

## Pinned tool versions

| tool | version |
|---|---|
| golangci-lint | v2.12.2 |
| gosec | v2.28.0 |
| govulncheck | v1.6.0 |
| goreleaser | v2.17.1 |
| actionlint | v1.7.12 |

Bumps are deliberate commits to this repository. See `CHANGELOG.md`.

## Module path for /v2+

`module-path` defaults to `github.com/<owner>/<repo>`, with no major-version
suffix. Repositories that have gone through a `/v2`, `/v3`, etc. major-version
bump must set `module-path` explicitly (e.g. `github.com/xraph/foo/v2`) — the
default will not add the suffix for you.

## Submodules input

`submodules` (on `go-release.yml`) is for **packages within the root module** —
directories that share the root `go.mod` but get their own install line and
proxy-warm call in the release notes, like `go-utils`'s `errs` and `log`. It is
**not** for repositories containing genuinely nested Go modules (their own
`go.mod`, e.g. `discovery/go.mod` in `farp`). Go resolves versions for a nested
module from a `<submodule>/vX.Y.Z` tag, and this workflow only ever creates the
bare `$VERSION` tag on the root module — so a nested module listed here gets an
install line and proxy warm that fail. Repositories with real nested modules
need their own per-submodule tagging step; this workflow does not implement
that.

## Recovering a failed dispatch release

On the `workflow_dispatch` path the tag is pushed before the GitHub release is
created. Tests and lint run first, so the common failures happen before anything
is mutated — but if release creation itself fails, the tag is already public.

To retry:

```bash
git push --delete origin v1.2.3
git fetch --prune --prune-tags
```

Then re-run the dispatch. A GitHub release requires its tag to exist, so
release-then-tag is not possible; this is a known trade-off.

## Caller permissions

A called (`workflow_call`) workflow can only **narrow** the permissions the
caller's token already has — it can never widen them. Job-level `permissions`
inside the reusable workflow are therefore a ceiling, not a grant: every
caller must declare, at the top level of its own workflow file, whatever
permissions its jobs actually need.

- `go-ci.yml` needs `contents: read` and `security-events: write` **declared by
  the caller**. Without the latter the gosec SARIF upload fails, since the
  default repo token is read-only.
- `go-release.yml` needs `contents: write` **declared by the caller**.
  Omitting it produces a failure at release-creation time, not at parse time.
- `codeql.yml` needs `actions: read`, `contents: read`, and
  `security-events: write` **declared by the caller**. Omitting it produces a
  `startup_failure` immediately, since the default repo token is read-only.
- `semantic-release.yml` needs `contents: write`, `issues: write` and
  `pull-requests: write`. Only the first is fatal if omitted; the other two
  degrade to a logged warning and skipped issue/PR comments.

GitHub validates the permissions of **every** nested job in a locally-called
reusable workflow at parse time, including jobs skipped by `if:`. An under-grant
does not skip a step — it fails the whole run with `startup_failure`.

## Squash-merging and release types

When you squash-merge, the **pull request title** becomes the commit message the
analyzer sees. Squashing genuine `fix:` or `feat:` commits under a `ci:` or
`chore:` title suppresses the release entirely — the individual commit types are
gone. This already happened once here, on `xraph/confy` PR #1. Title the PR with
the highest-impact type it contains.

## Binary track

`go-binary-release.yml` wraps GoReleaser. The consumer repository owns its
`.goreleaser.yml` — the config is genuinely per-project (build matrix, ldflags,
Homebrew and Scoop blocks) and is not something this repository should dictate.

Secrets, all optional: `HOMEBREW_TAP_TOKEN`, `SCOOP_BUCKET_TOKEN`,
`GPG_PRIVATE_KEY`. Pass them explicitly rather than with `secrets: inherit` so
the blast radius of a token stays visible in the caller.

### Docker, packages and cross-repo pushes

Opt-in, all defaulting off, so a Homebrew-only consumer is unaffected:

- `docker: true` sets up QEMU, Buildx and a registry login (`registry`,
  default `ghcr.io`). A `.goreleaser.yml` with `dockers` or `docker_manifests`
  **cannot** cross-build architectures without QEMU.
- `go-version-file: go.mod` reads the version from the module instead of
  duplicating it in the caller.
- Optional secrets: `GORELEASER_TOKEN` (preferred over `GITHUB_TOKEN` when
  GoReleaser pushes to a tap or bucket in another repo — it falls back to
  `GITHUB_TOKEN`), `FURY_TOKEN` (Gemfury, for `nfpms` deb/rpm), and `AUR_KEY`.

`forge` is the first consumer. `forgeui` and `smart-form` also carry a
`.goreleaser.yml` and are the obvious next ones.

## semantic-release

`semantic-release.yml` is language-agnostic. It reads conventional commits,
computes the next version, writes the changelog, and creates the GitHub release.
Anything ecosystem-specific belongs in the consumer's own `.releaserc.json`,
via `@semantic-release/exec`.

The pinned set is Node 22 with `semantic-release@25`. The current plugin
generation declares `engines.node: ^22.14.0 || >=24.10.0`, so Node 20 fails at
install. `spindle` and `farp` run `@23` on Node 20; they are unaffected.

### Caller permissions

```yaml
permissions:
  contents: write         # tag, and commit CHANGELOG.md via @semantic-release/git
  issues: write           # comment on resolved issues
  pull-requests: write    # and on the PRs that closed them
```

Omitting `issues:` or `pull-requests:` only logs a warning and skips the
comment. Omitting `contents: write` fails the release.

### Branch protection

`@semantic-release/git` commits `CHANGELOG.md` back to `main`. If `main` is
protected without an allowance for `github-actions[bot]`, that push fails and
the release aborts partway. Either exempt the bot, or drop
`@semantic-release/git` and let the changelog live only in the GitHub release.

## Go projects that need a JS toolchain

Some Go tests shell out to Node — generated-client suites that bundle an
emitted TypeScript client and execute it, for example. Two inputs cover that:

```yaml
node-version: '22'
npm-global-packages: 'typescript@5.8.2 esbuild@0.28.1'
```

Both default to empty, so nothing is installed unless asked.

## Private modules

`kgkit` is private. Anything that depends on it dies at `go mod download` with
"repository not found", which reads like a typo in the import path and isn't
one: Go asked proxy.golang.org, got a 404 back, and never got as far as trying
git, because nothing had told it that path was private to begin with.

Hand the workflow a token with read access and it sorts out the rest:

```yaml
jobs:
  ci:
    uses: xraph/workflows/.github/workflows/go-ci.yml@v1
    with:
      goprivate: github.com/xraph/kgkit
    secrets:
      XRAPH_REPO_TOKEN: ${{ secrets.XRAPH_REPO_TOKEN }}
```

Every job that downloads dependencies rewrites `https://github.com/xraph/`
to carry that token, then sets `GOPRIVATE` so those paths skip the proxy and go
straight to git. Leave the secret out and the step does nothing. A public-only
repo never has to think about it. The rewrite is scoped to the org and not to
all of github.com, so it won't stamp on a credential you already set up for
somewhere else.

Keep `goprivate` narrow. It sets `GONOPROXY` too, so a glob wider than your real
private modules drags perfectly public ones off the proxy and clones each of
them from scratch, in every job, on every run. foundry names a single module for
that reason: it pulls in twenty-two xraph modules and exactly one of them is
private. The default is `github.com/xraph/*`, which is the safe answer when you
do not know and the slow one when you do.

Read access on the private repos is all the token needs. A fine-grained PAT
works. So does a GitHub App installation token. The default `GITHUB_TOKEN` does
not, and cannot: it only ever reaches the repo the workflow is running in.

## Escaping the Makefile probe

`go-ci.yml` prefers a Makefile target when one matching the step exists. That
is usually what you want — but a repo whose `make test` sweeps every module in
a monorepo, while its CI deliberately tests only the root module, would get a
much broader (and slower) run than intended. Pass `prefer-makefile: false` to
force the plain `go` commands.

## Lint on a cold cache

golangci-lint typechecks the whole module while it loads packages. On a big
repo, with nothing in GOCACHE, that alone can outrun the deadline before a
single linter runs:

```
context loading failed: failed to load packages: context deadline exceeded
```

The lint job builds the module first so GOCACHE is populated and loading reuses
the compiled output. That step is deliberately non-fatal: a module that truly
does not compile should be reported by golangci-lint with a file and a line, not
as a bare build failure in a step you never asked for.

If linting is still slow once the cache is warm, raise the deadline:

```yaml
with:
  golangci-lint-timeout: '15m'
```

## Splitting gating from non-gating

`only-test: true` runs just the test matrix, skipping lint, verify and
security. A caller whose required check must not go red on an optional
platform calls the workflow twice: once with the defaults (blocking) and once
with `only-test` for the optional leg, leaving that one out of the gate's
`needs`.

## Security reporting

`go-ci.yml` runs gosec with `-no-fail -fmt sarif` and uploads the result, so
findings appear in the repository's Security tab with history, triage and
dismissal — not only in job logs. It then fails the build when
`gosec-fail-on-findings` (default `true`) is set. Pass `false` for
advisory-only scanning.

SARIF upload is skipped on pull requests from forks, which have no write token.
Those runs degrade to log-only output.

Generated files are excluded (`-exclude-generated`). gosec cannot be silenced
durably in generated code — a `#nosec` annotation is erased the next time the
generator runs — so scanning it only produces findings nobody can act on.
Anything carrying the standard `Code generated by ... DO NOT EDIT.` header is
skipped; fix such findings in the source template instead.

`codeql.yml` defaults to the `security-extended,security-and-quality` query
packs. Pass `queries: ''` for CodeQL's standard set.

### When an advisory has no fix

`govulncheck` reports only *reachable* vulnerabilities, so its findings are
real. But an advisory sometimes has no published patch — govulncheck prints
`Fixed in: N/A` — and then a repo's CI blocks indefinitely on something nobody
can act on.

`govulncheck-fail-on-findings: false` keeps the scan running and reporting
while letting the build pass. Name the specific advisory in a comment where you
set it, and flip it back when upstream ships a fix. `gosec-fail-on-findings`
behaves the same way, and both default to `true`.

## Rust track

`rust-ci.yml` runs lint (rustfmt + clippy), a security audit (`cargo-audit`),
and build + test. Two further jobs are opt-in and non-gating: `test-extended`
(an OS × toolchain matrix) and `docs`.

That split is deliberate — only the fast jobs gate merges, so the expensive
legs can run on `main` and a nightly cron without slowing per-PR feedback.
The caller decides when, by passing its own expression to `run-extended`.

Like `go-ci.yml`, commands are resolved **per Makefile target**: if the repo has
a Makefile defining `fmt`, `clippy`, `audit`, `build`, `test` or `docs`, that
target is used; otherwise the equivalent `cargo` command runs. Each job records
which it chose in the run summary.

**Callers need no `permissions:` block.** Nothing here uploads SARIF, so the
default read token suffices — unlike `go-ci.yml`, whose callers must grant
`security-events: write`.

**Library crates are audited too.** `cargo-audit` needs a `Cargo.lock`, which
library crates conventionally gitignore. The fallback generates one on the fly
rather than making every library consumer pass `skip-audit: true` and ship
unaudited.

**Splitting gating from non-gating.** `test-extended` and `docs` are non-gating
by design, but a caller whose required check depends on this workflow would go
red when a nightly-toolchain leg fails. Split into two calls: one with the
defaults (gating jobs only), and a second with `only-extended: true` that your
required check does not depend on.

**Calling this workflow twice in one repo** (a monorepo with two crates) needs a
distinct `docs-artifact-name` per call — artifact names must be unique within a
run, and cannot contain `/`, so `working-directory` cannot be used directly.

Rust crates frequently need native build dependencies. Pass them with
`system-deps-ubuntu` (apt) and `system-deps-macos` (brew); both default to empty.

### There is deliberately no Rust release workflow

`octopus` generates its release pipeline with cargo-dist from
`dist-workspace.toml`, and `farp-rust` is released by `farp`'s semantic-release
because its version is `farp`'s version. Both work; neither should be replaced
by a shared workflow.

## Node track

`node-ci.yml` installs your project once and runs the root package.json scripts
you list, in order. The default list is typecheck, lint, test and build. A
script your manifest doesn't define is skipped, and the job summary says so,
which lets one call serve a turbo monorepo and a single package with plain tsc.
If none of the listed scripts exist the job fails, because a green run that ran
nothing is nearly always a typo or the wrong `working-directory`.

The lockfile picks the package manager. `pnpm-lock.yaml` means pnpm, at the
version in your `packageManager` field. Anything else means npm: `npm ci` with a
`package-lock.json`, `npm install` without one.

```yaml
jobs:
  ci:
    uses: xraph/workflows/.github/workflows/node-ci.yml@v1
    with:
      scripts: '["typecheck","lint","test:ci","build"]'
      env-vars: |
        FORGE_DASHBOARD_URL=http://localhost:7901/dashboard
```

`env-vars` takes one `KEY=value` per line and gives it to every script. It ends
up in the job log, so keep secrets out of it.

### Publishing a pnpm workspace

By default `npm-publish.yml` installs each package with plain npm, and plain npm
can't resolve `workspace:*`. Set `package-manager: pnpm` and the workflow
installs the workspace once, stamps `version` on every listed package before it
packs any of them, and packs with pnpm. pnpm turns `workspace:*` into the
release version and `workspace:^` into a caret range on it, so give internal
peer dependencies `workspace:^` and they follow every release. Each tarball is
checked for a leftover `workspace:` range before `npm publish` sees it.

The workflow also refuses a list that leaves out a package another one needs. A
runtime, peer or optional dependency on an unlisted workspace package would pack
as a version nobody ever publishes. devDependencies don't count, since consumers
never install them.

`skip-existing: true` skips any `name@version` the registry already has, so you
can re-run a release that died half way through a long list. It's off by
default because it hides a forgotten version bump just as quietly.

`examples/node-library` has a CI caller and a tag-driven release that publishes
a pnpm workspace.
