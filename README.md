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

## Private Go module dependencies

A repo whose go.mod requires a private repo in this org needs the module
proxy authenticated before any `go` command touches the network, and a
reusable workflow's caller cannot inject a step into its jobs to do that
itself. `go-ci.yml` covers it directly: pass the org secret through, and a
"Configure Go module auth" step runs in `test`, `lint`, `verify` and
`security`, before that job's `Set up Go` step, so it takes effect for
`go mod download`, `golangci-lint`, `go vet`, `gosec` and `govulncheck` alike.

```yaml
secrets:
  CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
  XRAPH_REPO_TOKEN: ${{ secrets.XRAPH_REPO_TOKEN }}
```

The secret is optional and defaults to unset, so every existing caller is
unaffected. If your workflow uses `secrets: inherit`, you already get this
for free and there is nothing to change. If you pass secrets explicitly, as
recommended above, add the `XRAPH_REPO_TOKEN` line yourself alongside
`CODECOV_TOKEN`.

The step sets `GOPRIVATE` to `github.com/xraph/*` by default. Override it
with the `private-module-pattern` input if your module paths need a
different pattern, though that should be rare inside this org.

## Escaping the Makefile probe

`go-ci.yml` prefers a Makefile target when one matching the step exists. That
is usually what you want — but a repo whose `make test` sweeps every module in
a monorepo, while its CI deliberately tests only the root module, would get a
much broader (and slower) run than intended. Pass `prefer-makefile: false` to
force the plain `go` commands.

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
