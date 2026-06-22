# ADR-0001: Release automation pipeline and branch-to-track/channel mapping

Date: 2026-06-22
Status: Accepted

## Context

`headroom-client` is a Workshop SDK connector that wires in-workshop LLM traffic
to a host-side Headroom proxy. It ships **no upstream binary** — only
environment-injection hooks and a tunnel plug declaration. Its version (`VERSION`)
is the SDK's own semver, not a tracking version of any upstream package.

The goals this design must satisfy:

- New SDK versions land in the store with minimal manual intervention.
- The store channel a build is released to is unambiguously determined by the
  source branch.
- Local iteration (`sdkcraft try`) works without special setup.
- The pipeline is structurally identical to `omp-workshop-sdk` so operators
  familiar with that repo can navigate this one immediately.

## Decision

### 1. One `track/<N>` branch per SDK major; no `main`

Each long-lived branch corresponds to exactly one SDK major version and one
store track:

| Branch      | Store channel | SDK version range |
|-------------|---------------|-------------------|
| `track/0`   | `0/edge`      | `0.x.y`           |
| `track/1`   | `1/edge`      | `1.x.y` (future)  |
| `track/<N>` | `<N>/edge`    | `<N>.x.y` (future)|

The GitHub default branch always points to the **currently active** track.
There is no `main`; `git clone` lands directly on the active track.

### 2. `VERSION` file as the single source of truth

A plain-text `VERSION` file (e.g. `0.1.0`) at the repo root is the only place
where the SDK version is recorded. `sdkcraft.yaml` reads it at build time via
`override-pull` to set the SDK version field. No version appears in workflow
files or elsewhere.

### 3. No Renovate upstream-version manager

Unlike `omp-workshop-sdk`, there is no upstream binary to track via
`github-releases`. Renovate is configured with a minimal `renovate.json`
(`config:recommended` + `baseBranchPatterns`) for GitHub Actions and lockfile
updates only. SDK version bumps are deliberate operator actions.

### 4. CI/CD pipeline: build check → upload → promote

```
operator bumps VERSION (or adds hook/plug changes)
        │
        ▼
(PR opened targeting track/<N>)
build.yml  (required check)
  canonical/sdkcraft-actions reusable workflow
  runs `sdkcraft build` for ubuntu@22.04:amd64 and ubuntu@24.04:amd64
        │  passes
        ▼
PR merged → push to track/<N>
        │
        ▼
upload.yml  (triggers on push to track/<N>)
  3-job pipeline: snapshot → build-and-upload → promote
  snapshot: capture pre-upload channel state
  build-and-upload: build + upload to <N>/edge (track derived from VERSION major)
  promote: cascade pre-upload revisions one tier down the belt
```

### 5. Release-promotion conveyor

`upload.yml` runs three sequential jobs:

```
snapshot        — record which revisions sit in edge/beta/candidate BEFORE upload
build-and-upload — build + release two new revisions to <N>/edge
promote         — shift the pre-upload revisions one tier down the belt:
                    candidate → <N>/stable AND latest/stable
                    beta      → <N>/candidate
                    edge      → <N>/beta
```

The conveyor logic lives in `.github/scripts/promote-pipeline.sh` (injectable
`$SDKCRAFT` for testing) with subcommands `snapshot`, `promote`, and
`channel-revs` (pure parser for test isolation). The script is identical to
the one in `omp-workshop-sdk` — it is parameterized by NAME/TRACK and is
fully reusable.

Values flow from `snapshot` to `promote` via GitHub Actions job outputs
referenced through env vars (not inline `${{ }}` expressions) to prevent shell
injection. The existing `concurrency` group serialises pipelines per branch, so
the read-then-shift sequence is race-free within a track.

### 6. Track derivation

The store track is derived at pipeline runtime from the major component of
`VERSION` (`cut -d. -f1 VERSION`) and passed explicitly to the upload job.
This avoids the `GITHUB_REF` pitfall where the branch name `track/0` would
be resolved as `0/edge` incorrectly by some tools.

### 7. Repository ruleset

A single GitHub repository ruleset (`build-sdk-checks`, `target: branch`,
`enforcement: active`) on the ref pattern `refs/heads/track/*` requires the
`build / build` status check to pass before a matching branch can be updated.
The same ruleset blocks branch deletion and non-fast-forward (force) pushes.
The `refs/heads/track/*` pattern covers every current and future track branch
automatically.

### 8. SDK major-version rollover is a manual operator action

When the SDK gains a breaking change (e.g. new hook interface, renamed plug):

1. `git checkout -b track/1 track/0`
2. Set `VERSION` to `1.0.0`.
3. Update branch references in `build.yml` and `renovate.json`
   (`track/0` → `track/1`; `upload.yml` needs no branch edit).
4. Push and set `track/1` as the GitHub default branch.

The ruleset covers `track/1` automatically via the existing
`refs/heads/track/*` pattern.

See `DEVELOPERS.md` for the full step-by-step commands.

## Consequences

**Positive:**

- Channel membership is statically determined by branch name; there is no
  ambiguity about where a build lands.
- Pipeline structure is identical to `omp-workshop-sdk`; operators need to
  learn only one pattern.
- The promotion conveyor populates `beta`, `candidate`, and `stable` channels
  automatically; no human intervention after the initial upload.
- The `refs/heads/track/*` ruleset requires no maintenance as new tracks
  are added.

**Negative / trade-offs:**

- SDK version bumps require a manual commit (no Renovate automation).
  This is acceptable because the SDK ships no binary — bump frequency is low.
- Each release requires three jobs instead of one; end-to-end wall time
  increases by one extra `snap install sdkcraft` + store API round-trip.
- Store tracks (`0` and `latest`) and the `latest/stable` guardrail must exist
  in the store before the first promotion; this is a one-time operator action
  outside this repository.
