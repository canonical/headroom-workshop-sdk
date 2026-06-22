# headroom-workshop-sdk

Workshop SDK that routes a workshop's LLM traffic through a
[Headroom](https://github.com/headroomlabs-ai/headroom) compression proxy running on
the host.

## What this repo is

A Workshop SDK repo. It produces an SDK (`headroom-client`) that:

1. Declares a `tunnel` plug (`headroom`, `localhost:8787`) so the workshop's port 8787
   is forwarded to the host Headroom proxy at `127.0.0.1:8787`.
2. Writes `/etc/profile.d/headroom.sh` and appends to `/etc/environment` so every
   in-workshop login shell sees `ANTHROPIC_BASE_URL=http://localhost:8787` and
   `OPENAI_BASE_URL=http://localhost:8787/v1`.

The proxy itself, the CCR cache, and cross-agent memory all live on the host and are
shared by every workshop on the machine. No compression engine is installed in the
workshop.

## Repo structure

```
sdkcraft.yaml          SDK definition (plugs, nil version part)
hooks/setup-base       Writes /etc/profile.d/headroom.sh and /etc/environment (root)
hooks/check-health     Verifies the env file exists; workshopctl set-health okay (root)
VERSION                SDK's own semver (e.g. 0.1.0) — no upstream binary to track
renovate.json          Minimal Renovate config (no github-releases manager)
.github/workflows/
  build.yml            PR check: builds on PRs targeting track/0
  upload.yml           Release: 3-job pipeline (snapshot → build+upload → promote)
                       uploads to 0/edge, then cascades old revisions down the belt
  renovate.yml         Renovate bot schedule
  renovate-check.yml   Validates renovate.json on PRs
  release-ondemand.yml Manual workflow_dispatch for dry-run / forced release
.github/scripts/
  promote-pipeline.sh      Snapshot/promote/channel-revs helpers; $SDKCRAFT injectable
  promote-pipeline.test.sh Bash test harness for the promotion script
tests/
  spread.yaml              LXD vm spread config; stages try/headroom-client
  main/launch/
    workshop.yaml.in       Test workshop definition (envsubst ${BASE})
    task.yaml              Env-injection smoke test (no host proxy needed in CI)
docs/adrs/
  0001-release-automation-pipeline.md  Branch-to-track mapping and pipeline design
```

## Key design facts

- **No binary**: the SDK installs nothing executable — only environment-injection files.
  The `version` part uses `plugin: nil` with `override-pull` to set the SDK version
  from `VERSION`.
- **Multi-base**: `ubuntu@22.04:amd64` + `ubuntu@24.04:amd64` (no `build-base` field)
- **Track**: `0/edge` — branch `track/0`; track number derived from the major in `VERSION`
  (`cut -d. -f1 VERSION`) at release time, not hardcoded in upload.yml.
- **Tunnel plug**: `headroom` (interface: tunnel, endpoint: localhost:8787). Must be
  connected to a `system` SDK slot in the workshop definition. Host-side slots use strict
  validation and are **not** auto-connected at launch.
- **Health check**: lenient — only checks that `/etc/profile.d/headroom.sh` exists.
  Does **not** probe `http://localhost:8787/health` because the tunnel may not be
  connected during the first `check-health` run.
- **VERSION**: the SDK's own semver (starting `0.1.0`). There is no upstream binary to
  track. Renovate is configured for GitHub Actions / lockfile updates only.

## Branch/CI structure

- `track/0`: default branch — has VERSION, all workflows (build, upload, Renovate)
- No `main` branch; Renovate runs from the default branch

To bootstrap a new track branch (e.g., `track/1` for a future SDK major bump):
1. `git checkout -b track/1 track/0`
2. Update `VERSION` to the first 1.x release
3. Update `build.yml` and `renovate.json`: branch `"track/0"` → `"track/1"`
   (`upload.yml` needs no branch edit — track is derived from `VERSION`)
4. `git commit -m "chore: configure 1/edge track" && git push -u origin track/1`
5. `gh api repos/<owner>/headroom-workshop-sdk -X PATCH -f default_branch='track/1'`

## Host-side prerequisites

The Headroom proxy must be running on the host before the tunnel is useful. See
`README.md` for the install commands and the systemd user service definition.

## Iterate locally

```bash
cd ~/canonical/headroom-workshop-sdk
git checkout track/0
sdkcraft try --verbose        # → referenceable as try-headroom-client
# edit your .workshop.yaml to use try-headroom-client + system slot + connection
workshop refresh
workshop connect <ws>/headroom-client:headroom <ws>/system:headroom
workshop shell
env | grep ANTHROPIC_BASE_URL
curl -fsS http://localhost:8787/health
```
