# headroom-workshop-sdk

Workshop SDK that bakes a self-contained
[Headroom](https://github.com/headroomlabs-ai/headroom) compression proxy into its
payload and runs it inside each workshop container.

## What this repo is

A Workshop SDK repo. It produces an SDK (`headroom-client`) that:

1. Bakes a relocatable CPython interpreter with `headroom-ai[proxy,code]` into the
   payload at pack time (the `headroom` part) and runs `headroom proxy` on
   `127.0.0.1:8787` inside the container as a systemd user service (`hooks/setup-project`).
2. Writes `/etc/profile.d/headroom.sh` and appends to `/etc/environment` so every
   in-workshop login shell sees `ANTHROPIC_BASE_URL=http://localhost:8787` and
   `OPENAI_BASE_URL=http://localhost:8787/v1`.
3. Persists the proxy's workspace (`/home/workshop/.headroom`: CCR cache, savings
   ledger, cross-agent memory) across `workshop refresh` via the `headroom-home` mount
   plug.

The compression engine runs entirely inside the workshop; there is no host proxy and
no tunnel.

## Repo structure

```
sdkcraft.yaml          SDK definition (headroom payload part, version part, mount plug)
hooks/setup-base       Writes /etc/profile.d/headroom.sh and /etc/environment (root)
hooks/setup-project    Writes + enables the headroom-proxy systemd user service (workshop user)
hooks/check-health     Probes localhost:8787/health, else checks the service is active (root)
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
    task.yaml              Env-injection + local-proxy /health smoke test
docs/adrs/
  0001-release-automation-pipeline.md  Branch-to-track mapping and pipeline design
  0002-self-hosted-engine.md            Shift from host-tunnel to workshop-local engine
```

## Key design facts

- **Baked engine**: the `headroom` part (`plugin: nil`, `override-build`) fetches a
  relocatable standalone CPython 3.13 via `uv`, installs `headroom-ai[proxy,code]==0.35.0`
  into its site-packages (`--system --break-system-packages`, no venv — venvs break under
  payload relocation), and copies the whole prefix to `$SDK/python`. The service invokes
  `$SDK/python/bin/python3 $SDK/python/bin/headroom` (explicit interpreter, ignores the
  build-path shebang) so it is relocation-safe. The `version` part stays `plugin: nil` +
  `override-pull` setting the SDK version from `VERSION`.
- **Local proxy**: `hooks/setup-project` (runs as the `workshop` user) writes
  `~/.config/systemd/user/headroom-proxy.service` with `$SDK` resolved to absolute paths
  and runs `systemctl --user enable --now headroom-proxy` → `headroom proxy --host
  127.0.0.1 --port 8787`. `HEADROOM_WORKSPACE_DIR=/home/workshop/.headroom`.
- **Multi-base**: `ubuntu@22.04:amd64` + `ubuntu@24.04:amd64` (no `build-base` field).
  Single linux-x86_64 standalone build covers both.
- **Track**: `0/edge` — branch `track/0`; track number derived from the major in `VERSION`
  (`cut -d. -f1 VERSION`) at release time, not hardcoded in upload.yml.
- **Mount plug**: `headroom-home` (interface: mount, workshop-target
  `/home/workshop/.headroom`). Auto-connected to the `system` SDK's mount slot at launch;
  persists CCR cache, savings ledger, and cross-agent memory across `workshop refresh`.
- **Health check**: probes `http://localhost:8787/health`; if that is not up yet
  (uvicorn cold start ~10s exceeds the retry window), it treats an `active`
  `headroom-proxy` service as `okay`. The launch smoke test's `--retry` curl is the
  load-bearing proof the proxy actually serves.
- **VERSION**: the SDK's own semver (`0.2.0` after the self-hosted-engine change). The
  `headroom-ai` pin is bumped manually; Renovate is GitHub Actions / lockfile only.

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

None. The compression engine is baked into the SDK payload and runs inside the
workshop. Nothing needs to be installed or running on the host.

## Iterate locally

```bash
cd ~/canonical/headroom-workshop-sdk
git checkout track/0
sdkcraft try --verbose        # → referenceable as try-headroom-client
# add try-headroom-client to your .workshop.yaml (no slot/connection needed)
workshop refresh
workshop shell
env | grep ANTHROPIC_BASE_URL
systemctl --user is-active headroom-proxy   # -> active
curl -fsS http://localhost:8787/health      # local proxy serving
```
