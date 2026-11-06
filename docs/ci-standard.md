# azrtydxb CI standard

One place owns CI. Product repos are thin callers of the reusable workflows in
this repo (`azrtydxb/.github`). This replaces the per-repo copy-paste that left
us with 9 runner labels, 4 action-pinning styles and almost no shared caching.

## Runner labels (the only ones allowed)

Decided 2026-11-06: the runner follows the repo's visibility.

| Repo                      | Use                                         | Label                                              |
| ------------------------- | ------------------------------------------- | -------------------------------------------------- |
| **Public**                | Linux CI, image build and push              | `ubuntu-latest` (GitHub-hosted, amd64)             |
| **Public**                | Native arm64 tests (not image builds)       | `ubuntu-24.04-arm` (GitHub-hosted)                 |
| **Private**               | Everything on Linux                         | `arc-azrtydxb-amd64` (ARC on quadronix, amd64)     |
| Either                    | macOS / Windows jobs that need those OSes   | `macos-latest`, `windows-latest`                   |

Public repos never use a self-hosted label. Private repos never use `ubuntu-*` or the
arm64 OKE set `arc-azrtydxb`. The reusable workflows pick the label for you from the
caller's visibility when their `runner` input is empty (the default); pass `runner` only
to override.

**arm64 is cross-compiled, never emulated.** No `docker/setup-qemu-action`, no
`tonistiigi/binfmt`, no `qemu-user-static`. Build `linux/amd64,linux/arm64` together in
one job on the amd64 runner:

- every Dockerfile stage that has a `RUN` is `FROM --platform=$BUILDPLATFORM ...` with
  `ARG TARGETOS TARGETARCH` (Go: `GOOS=$TARGETOS GOARCH=$TARGETARCH CGO_ENABLED=0`; Rust:
  `cargo zigbuild` or a rustup target plus a cross linker, not `cross`, which uses QEMU);
- the final (target-platform) stage has **no `RUN`**: only FROM/COPY/ENV/ARG/USER/WORKDIR/
  EXPOSE/LABEL/ENTRYPOINT/CMD. Do apk/apt/adduser/mkdir/chmod in a `$BUILDPLATFORM`
  stage and `COPY` the result;
- arm64 tests cannot run on amd64 without emulation: private repos run amd64 tests and
  keep only cross builds and vet (`GOARCH=arm64 go build ./...`); public repos may run
  arm64 tests natively on `ubuntu-24.04-arm`.

`actions/setup-builder` creates a per-arch native builder pod in the runner's cluster.
Its arm64 builder pod template no longer exists on OKE, so **do not use it to get arm64**
any more: cross-build instead. The action is left in place for amd64-only use on the ARC
runners.

**Retired:** `arc-azrtydxb` (arm64, OKE) for CI, `arc-azrtydxb-publish`,
`arc-azrtydxb-amd64-publish`, `qnap-nas`, `novanas-amd64`, `[self-hosted, zfs]`,
`[self-hosted, nvidia, gpu]`, `[self-hosted, novaflow-prod]`. Jobs that ask for them
never start or sit on the wrong architecture.

Fork PRs on public repos are safe by construction: they only ever run on GitHub-hosted
runners, so the old fork-safe `runs-on` expression is no longer needed.

## Package and image mirrors (Kuvryn Ant)

Runners pull through Kuvryn Ant (`https://ant.azrty.com`), not Nexus. dind mirrors
Docker Hub through Ant; other registries are pulled by prefixed name
(`ant.azrty.com/ghcr/...`, `ant.azrty.com/dockerhub/...`). Go, npm/pnpm, cargo and pip/uv are configured per job by `azrtydxb/ant-setup-action`, which exchanges the job's
GitHub OIDC token for a short-lived Ant token, so no secret is stored:

```yaml
permissions:
  contents: read
  id-token: write
steps:
  - uses: actions/checkout@v7
  - uses: azrtydxb/ant-setup-action@v1  # (docker-login: "true" to push to ant.azrty.com/oci-hosted/...)
```

`main`, `master` and `v*` tags get publish on `oci-hosted`; every other ref is read-only.

The reusable Go, Node, Python and Rust workflows already run the action, so their callers
must grant `id-token: write` (next to `contents: read`) or the called workflow will not start.
Maven, NuGet, Composer, RubyGems and SwiftPM are not configured by the action.

## Reusable workflows

| Workflow                            | Language | Key inputs                                                     |
| ----------------------------------- | -------- | -------------------------------------------------------------- |
| `reusable-go.yml`                   | Go       | `working-directory`, `tier`, `race`, `lint`                    |
| `reusable-node.yml`                 | Node/TS  | `working-directory`, `tier`, `node-version`, `package-manager` |
| `reusable-rust.yml`                 | Rust     | `working-directory`, `tier`, `clippy`                          |
| `reusable-python.yml`               | Python   | `working-directory`, `tier`, `python-version`, `install`       |
| `reusable-docker-publish.yml`       | images   | `image`, `context`, `target`, `platforms`                      |
| `reusable-security.yml`             | any      | `runner`                                                       |
| `reusable-dependabot-automerge.yml` | any      | `merge-method`, `allow-major`                                  |

Reference them at `@main`:

```yaml
uses: azrtydxb/.github/.github/workflows/reusable-go.yml@main
```

## Test tiers

Every repo splits its tests into **three tiers by trigger**, not one big job on
every push. The PR gate must stay **under 5 minutes**; anything slower moves to a
nightly or release trigger.

| Tier          | Trigger                        | What runs                                                                                     | Budget      |
| ------------- | ------------------------------ | --------------------------------------------------------------------------------------------- | ----------- |
| `unit`        | `pull_request`, `push` to main | fmt/lint/typecheck + **hermetic** unit tests. No containers, browsers, network, kind, or e2e. | **< 5 min** |
| `integration` | `schedule` (nightly)           | unit + tests that need services (db, containers, kind, testcontainers).                       | < 30 min    |
| `full`        | `push` tag `v*` (release)      | integration + e2e/playwright + race + coverage + full matrix.                                 | no hard cap |

Caller files (copy from `templates/`):

- `ci.yml` → `tier: unit` (required `gate`)
- `nightly.yml` → `tier: integration` (`templates/nightly-integration.yml`)
- `release.yml` → `tier: full` (`templates/release-full.yml`)

### How tests are separated (per language)

The reusable workflows select the tier; the repo must actually keep the tiers
apart, or the PR run still drags the slow tests in.

| Language | unit                                                                     | integration                                        |
| -------- | ------------------------------------------------------------------------ | -------------------------------------------------- |
| Go       | default build; integration tests behind `//go:build integration`         | `go test -tags integration ./...`                  |
| Node     | `npm test`                                                               | `npm run test:integration`, e2e in `test:e2e`      |
| Rust     | `cargo test`; slow tests `#[ignore]` or behind the `integration` feature | `cargo test --all-features`                        |
| Python   | `pytest -m "not integration and not e2e"`                                | `@pytest.mark.integration`, e2e `@pytest.mark.e2e` |

**Rule:** if a PR job exceeds 5 minutes, either (a) split it into parallel jobs,
(b) move the slow part down a tier, or (c) both. Never gate merges on tests that
need hardware, external services or a browser.

## Caller shape

```yaml
name: ci
on:
  pull_request:
  push:
    branches: [main]

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  go:
    uses: azrtydxb/.github/.github/workflows/reusable-go.yml@main
    with:
      working-directory: .

  # The single required status check. Never require the job names themselves —
  # require this one, so adding/removing jobs never strands a PR.
  gate:
    if: always()
    needs: [go]
    runs-on: ${{ github.event.repository.private && 'arc-azrtydxb-amd64' || 'ubuntu-latest' }}
    steps:
      - run: |
          [ "${{ needs.go.result }}" = "success" ] || exit 1
```

## The `gate` convention

Branch protection must require **one** check named `gate`. It `needs:` every real
job, runs `if: always()`, and exits non-zero if any dependency did not succeed.
This is already how `novamail`, `procoder` and `NovaFlow` work.

> Why: `novamem` made `package (npm)` a required check, then had to keep a no-op
> job alive forever just to satisfy protection. One `gate` avoids that.

## Pinning policy

- Actions are referenced **by version tag**, third-party and first-party alike: the
  project's major tag (`actions/checkout@v7`, `azrtydxb/ant-setup-action@v1`), or its
  newest full release tag where it publishes no major tag. Minor and patch releases
  then apply without a PR; Dependabot raises a PR for a new major.
- No commit hashes, no branches. Exceptions: the org's own shared workflows and
  actions in `azrtydxb/.github` (no release tags) use `@main`, and a moving
  toolchain channel stays a channel (`dtolnay/rust-toolchain@stable`).
- Decided 2026-11-06, replacing hash pinning: one release of a first-party action
  meant a bump PR in every repo.

## Merge policy

- Org ruleset on the default branch: require PR, require check `gate`, block
  force-push and deletion, linear history, squash-only.
- Auto-merge on for every repo; Dependabot patch/minor auto-merge via the
  reusable workflow; keep a merge queue only where CI is fast and no publish job
  is a required PR check.
