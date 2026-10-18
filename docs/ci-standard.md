# azrtydxb CI standard

One place owns CI. Product repos are thin callers of the reusable workflows in
this repo (`azrtydxb/.github`). This replaces the per-repo copy-paste that left
us with 9 runner labels, 4 action-pinning styles and almost no shared caching.

## Runner labels (the only ones allowed)

| Use | Label |
| --- | --- |
| Linux CI, containers, **default** | `arc-azrtydxb` (arm64) |
| Linux CI that must be x86 | `arc-azrtydxb-amd64` |
| Build **and push** images to Nexus | `arc-azrtydxb-publish` (arm64) |
| Untrusted fork PRs / macOS/Windows | `ubuntu-latest`, `macos-latest`, `windows-latest` |

Everything we run is arm64 (kw and OKE are arm64), so **default to arm64**.
Do not request `linux/amd64` images unless a consumer actually needs them.

**Retired:** `qnap-nas`, `novanas-amd64`, `[self-hosted, zfs]`,
`[self-hosted, nvidia, gpu]`, `[self-hosted, novaflow-prod]`. They are not in the
org runner inventory; jobs that ask for them never start.

Fork-safe pattern for public repos (never hand a fork PR a privileged LAN runner):

```yaml
runs-on: ${{ (github.event_name != 'pull_request' ||
              github.event.pull_request.head.repo.full_name == github.repository)
             && 'arc-azrtydxb' || 'ubuntu-latest' }}
```

## Reusable workflows

| Workflow | Language | Key inputs |
| --- | --- | --- |
| `reusable-go.yml` | Go | `working-directory`, `tier`, `race`, `lint` |
| `reusable-node.yml` | Node/TS | `working-directory`, `tier`, `node-version`, `package-manager` |
| `reusable-rust.yml` | Rust | `working-directory`, `tier`, `clippy` |
| `reusable-python.yml` | Python | `working-directory`, `tier`, `python-version`, `install` |
| `reusable-docker-publish.yml` | images | `image`, `context`, `target`, `platforms` |
| `reusable-security.yml` | any | `runner` |
| `reusable-dependabot-automerge.yml` | any | `merge-method`, `allow-major` |

Reference them at `@main`:

```yaml
uses: azrtydxb/.github/.github/workflows/reusable-go.yml@main
```

## Test tiers

Every repo splits its tests into **three tiers by trigger**, not one big job on
every push. The PR gate must stay **under 5 minutes**; anything slower moves to a
nightly or release trigger.

| Tier | Trigger | What runs | Budget |
| --- | --- | --- | --- |
| `unit` | `pull_request`, `push` to main | fmt/lint/typecheck + **hermetic** unit tests. No containers, browsers, network, kind, or e2e. | **< 5 min** |
| `integration` | `schedule` (nightly) | unit + tests that need services (db, containers, kind, testcontainers). | < 30 min |
| `full` | `push` tag `v*` (release) | integration + e2e/playwright + race + coverage + full matrix. | no hard cap |

Caller files (copy from `templates/`):

- `ci.yml` → `tier: unit` (required `gate`)
- `nightly.yml` → `tier: integration` (`templates/nightly-integration.yml`)
- `release.yml` → `tier: full` (`templates/release-full.yml`)

### How tests are separated (per language)

The reusable workflows select the tier; the repo must actually keep the tiers
apart, or the PR run still drags the slow tests in.

| Language | unit | integration |
| --- | --- | --- |
| Go | default build; integration tests behind `//go:build integration` | `go test -tags integration ./...` |
| Node | `npm test` | `npm run test:integration`, e2e in `test:e2e` |
| Rust | `cargo test`; slow tests `#[ignore]` or behind the `integration` feature | `cargo test --all-features` |
| Python | `pytest -m "not integration and not e2e"` | `@pytest.mark.integration`, e2e `@pytest.mark.e2e` |

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
    runs-on: arc-azrtydxb
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

- Third-party and first-party actions are pinned **by commit SHA** with a
  `# vX.Y.Z` comment. Dependabot/Renovate keeps the comments current.
- The one exception is a moving toolchain channel: `dtolnay/rust-toolchain@stable`.
- `actions/create-github-app-token` bumps are pinned by SHA too.

## Merge policy

- Org ruleset on the default branch: require PR, require check `gate`, block
  force-push and deletion, linear history, squash-only.
- Auto-merge on for every repo; Dependabot patch/minor auto-merge via the
  reusable workflow; keep a merge queue only where CI is fast and no publish job
  is a required PR check.
