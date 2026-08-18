# ADR-0002: Self-hosted compression engine (workshop-local proxy)

Date: 2026-08-18
Status: Accepted
Supersedes the architecture (not the release pipeline) of ADR-0001.

## Context

The original `headroom-client` (ADR-0001, `0.1.x`) shipped **no engine**. It declared a
`tunnel` plug forwarding the workshop's port 8787 to a Headroom proxy the operator had
to install and run on the host, and wrote `ANTHROPIC_BASE_URL`/`OPENAI_BASE_URL` env
files pointing at `localhost:8787`.

This coupled every workshop to host-side setup: the operator had to `uv tool install
headroom-ai[proxy,code]`, run a systemd user service, enable lingering, and manually
connect the tunnel (host-side slots use strict validation and are not auto-connected).
A workshop was not self-contained — moving it to another machine, or onboarding a new
operator, required replicating the host install.

## Decision

Make each workshop self-contained: **bake the engine into the SDK payload and run it
inside the container.**

### 1. Bake a relocatable Headroom install at pack time

A new `headroom` part (`plugin: nil`, `override-build`) fetches a python-build-standalone
CPython 3.13 via `uv`, installs `headroom-ai[proxy,code]==0.35.0` into that interpreter's
site-packages, and copies the whole prefix into the payload at `$SDK/python`.

- **No venv.** A venv's `pyvenv.cfg` `home` is an absolute path to the base interpreter
  and breaks when the payload is relocated from the build dir to the runtime `$SDK`
  prefix (uv's `--relocatable` only fixes bin/ shebangs). Standalone interpreters resolve
  their stdlib relative to the executable, so copying the whole prefix is relocation-safe.
- **`--system --break-system-packages`.** The standalone interpreter carries an
  `EXTERNALLY-MANAGED` marker (uv-managed); the flags let `uv pip install` write into its
  site-packages directly.
- Relocation-safety is verified: extracting the packed `.sdk` to an arbitrary path and
  running `$path/python/bin/python3 -c "import headroom"` + `… headroom --version`
  succeeds.

Only amd64 platforms exist (`ubuntu@22.04:amd64`, `ubuntu@24.04:amd64`, build-on ==
build-for), so a single linux-x86_64 standalone build covers both bases.

### 2. Run the proxy as a systemd user service

A new `hooks/setup-project` hook (runs as the `workshop` user with `$HOME`,
`$XDG_RUNTIME_DIR`, `$DBUS_SESSION_BUS_ADDRESS`, `$SDK` set) writes
`~/.config/systemd/user/headroom-proxy.service` with `$SDK` resolved to absolute paths
and runs `systemctl --user enable --now headroom-proxy`. The unit's `ExecStart` invokes
`$SDK/python/bin/python3 $SDK/python/bin/headroom proxy --host 127.0.0.1 --port 8787` —
the explicit-interpreter form ignores the console script's build-path shebang, which is
the relocation-proof entry point.

### 3. Persist the workspace via a mount plug

The `tunnel` plug is removed. A `headroom-home` mount plug
(`workshop-target: /home/workshop/.headroom`, matching `HEADROOM_WORKSPACE_DIR`)
persists the CCR cache, savings ledger, cross-agent memory, and logs across
`workshop refresh`. It auto-connects to the `system` SDK's mount slot at launch — no
manual connection step, unlike the old strict host-side tunnel slot.

### 4. Health check probes the local proxy

`hooks/check-health` probes `http://localhost:8787/health`. Because uvicorn cold start
(~10 s) exceeds the check-health retry window, an `active` `headroom-proxy` service is
treated as `okay`; the launch smoke test's `--retry --retry-connrefused` curl to
`/health` is the load-bearing proof the proxy actually serves.

### 5. Env injection is unchanged

`hooks/setup-base` still writes `/etc/profile.d/headroom.sh` and `/etc/environment` with
`ANTHROPIC_BASE_URL=http://localhost:8787` / `OPENAI_BASE_URL=http://localhost:8787/v1`.
Clients still target `localhost:8787`, now served locally instead of tunneled — the
client-facing contract is identical.

### 6. Version and release pipeline

`VERSION` bumps `0.1.0 → 0.2.0`. Removing the tunnel plug is a breaking change, but `0.x`
semver permits it without a new track, so the work stays on `track/0` (`0/edge`). The
ADR-0001 release pipeline (branch-to-track mapping, snapshot → build → promote conveyor)
is unchanged. The `headroom-ai` pin is bumped manually, consistent with ADR-0001's
"no upstream-version manager" decision.

## Consequences

**Positive:**

- Workshops are self-contained: no host install, no lingering, no manual tunnel connect.
  Launch the SDK and the engine runs.
- Portable: the `.sdk` carries the whole engine; a fresh machine needs nothing extra.
- The mount plug auto-connects, removing the strict host-side-slot connection friction.

**Negative / trade-offs:**

- Payload size grows from an env-file-only SDK to ~150 MB (CPython + headroom + deps).
- Each workshop runs its own proxy process (~350 MB RSS) instead of one shared host
  proxy; CCR cache and cross-agent memory are per-workshop, not machine-wide.
- The Kompress ONNX model is not prefetched; the first *compression* request may download
  it from HuggingFace (network). `/health` and `/proxy` do not depend on it. Prefetching
  into `HEADROOM_WORKSPACE_DIR` at build time is a future option if fully-offline
  compression is required.
- The `headroom-ai` version is pinned in `sdkcraft.yaml` and bumped by hand.
