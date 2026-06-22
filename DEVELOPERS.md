# Developer guide

## Pipeline and branch model

See [ADR-0001](docs/adrs/0001-release-automation-pipeline.md) for the full
rationale and design of the branch-to-track/channel mapping, build/upload
pipeline, and the ruleset that gates merges.

Summary: each `track/<N>` branch maps to store channel `<N>/edge`. Unlike the
`omp-workshop-sdk`, there is no Renovate upstream-version manager here — the
SDK ships no binary, so `VERSION` tracks the SDK's own semver. Bump it manually
when the SDK gains new features.

On every push to a `track/<N>` branch the upload pipeline runs three jobs:

1. **snapshot** — records which revisions are currently in `<N>/edge`,
   `<N>/beta`, and `<N>/candidate` (pre-upload state). The store track is
   derived from the major in `VERSION` (`0.x.y → 0`), not from the branch name.
2. **build-and-upload** — builds both platforms and releases the new revisions
   to `<N>/edge`.
3. **promote** — cascades the pre-upload revisions one risk level down:
   `<N>/candidate → <N>/stable + latest/stable`, `<N>/beta → <N>/candidate`,
   `<N>/edge → <N>/beta`. Empty tiers are no-ops.

The promotion script is `.github/scripts/promote-pipeline.sh`; run its test
harness with `bash .github/scripts/promote-pipeline.test.sh`.

## Local development

Always work from a `track/*` branch:

```bash
git checkout track/0
sdkcraft try --verbose
# add try-headroom-client to your workshop definition + system slot + connection
workshop refresh
workshop connect <ws>/headroom-client:headroom <ws>/system:headroom
workshop shell
env | grep -E 'ANTHROPIC_BASE_URL|OPENAI_BASE_URL'
curl -fsS http://localhost:8787/health
workshop info   # runs check-health; should show status: okay
```

## Bumping the SDK version

`VERSION` is the SDK's own semver. To release a new SDK version:

```bash
echo "0.2.0" > VERSION
git add VERSION
git commit -m "chore: bump SDK to 0.2.0"
git push
```

The upload pipeline triggers on push to `track/0` and releases the new version
to `0/edge`.

## Bootstrapping a new track

When the SDK needs a major version bump (breaking change in hook behaviour,
tunnel interface change, etc.):

1. Create the new version branch from the current default:
   ```bash
   git checkout -b track/1 track/0
   ```

2. Set `VERSION` to the first 1.x release:
   ```bash
   echo "1.0.0" > VERSION
   ```

3. Update branch references in `build.yml` and `renovate.json`:
   ```bash
   # .github/workflows/build.yml  — pull_request branches: ["track/0"] → ["track/1"]
   # renovate.json — baseBranchPatterns: ["track/0"] → ["track/1"]
   # upload.yml needs no branch edit — track is derived from VERSION
   ```

4. Commit and push:
   ```bash
   git add -A && git commit -m "chore: configure 1/edge track"
   git push -u origin track/1
   ```

5. Roll the GitHub default branch:
   ```bash
   gh api repos/<owner>/headroom-workshop-sdk -X PATCH -f default_branch='track/1'
   ```

6. The `track/1` branch is gated automatically by the existing repository
   ruleset (`build-sdk-checks`) on the `refs/heads/track/*` pattern; no
   per-branch configuration is needed.

## On-demand release / dry-run

`.github/workflows/release-ondemand.yml` (manual `workflow_dispatch`) exercises
the whole release path — snapshot → build → upload → promote — without waiting
for a push to `track/<N>`. Two inputs:

- `mode`:
  - `release` (**default**) — builds, uploads the new revisions to `<N>/edge`,
    and cascades the promotion belt. **This mutates the (staging) store.**
  - `dry-run` — builds both platforms, prints the would-be
    `sdkcraft upload … --release <N>/edge` and `sdkcraft release …` commands,
    attaches the `.sdk` files as workflow artifacts, and writes nothing to the
    store. (The snapshot step still *reads* `sdkcraft revisions`.)
- `runner`: JSON array of runner labels. Default `["ubuntu-latest"]` runs on
  GitHub-hosted runners — an LXD **container** build that needs no KVM, so it
  works in a runner-less fork. Pass `["self-hosted","linux","jammy","x64","xlarge"]`
  to build on the production fleet.

```bash
# Build-only smoke test (no store writes), GitHub-hosted:
gh workflow run release-ondemand.yml -f mode=dry-run -f runner='["ubuntu-latest"]'

# Force a real release on the production fleet:
gh workflow run release-ondemand.yml -f mode=release \
  -f runner='["self-hosted","linux","jammy","x64","xlarge"]'
```

`workflow_dispatch` requires the workflow file to exist on the **default
branch** before it can be triggered. `tests/spread.yaml` (LXD `vm: true`, KVM)
is intentionally *not* run here — `sdkcraft pack` (containers) is the
GitHub-hosted-safe build; `sdkcraft test` is not.

## Provisioning checklist

For the automation to run green end-to-end, the production repository/org must
have all of the following (a runner-less fork satisfies only the last two, so
use the on-demand `dry-run` path there):

- [ ] Self-hosted runners labelled `self-hosted,linux,jammy,x64,xlarge` (used by
      the reusable `build.yml`/`upload.yml`), **or** override their `runs-on`.
      Without runners the PR `build` check and `upload.yml` queue indefinitely.
- [ ] Actions secret `SDKCRAFT_STORE_CREDENTIALS_STAGING` set. Without it,
      `snapshot` silently treats the belt as empty and uploads/promotes fail
      auth. Confirm staging is the intended publish target.
- [ ] Store tracks `0` and `latest`, plus the `latest/stable` guardrail,
      already exist (one-time operator action, outside this repo).
- [ ] Repository ruleset on `refs/heads/track/*` requiring the `build / build`
      status check (verify: `gh api repos/<owner>/headroom-workshop-sdk/rules/branches/track/0`).
- [ ] "Allow GitHub Actions to create and approve pull requests" enabled
      (Settings → Actions → General) so Renovate can auto-merge.
