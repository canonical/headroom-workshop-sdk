# Headroom client SDK for Workshop

This SDK bakes a self-contained [Headroom](https://github.com/headroomlabs-ai/headroom)
compression proxy into its payload and runs it **inside the workshop** on
`127.0.0.1:8787` as a systemd user service. It sets `ANTHROPIC_BASE_URL` and
`OPENAI_BASE_URL` so every in-workshop LLM call is compressed locally before leaving
the container. No host-side proxy and no tunnel are required.

---

## No prerequisites

The engine ships in the SDK payload — a relocatable CPython interpreter with
`headroom-ai[proxy,code]` installed into its own site-packages. Nothing needs to be
installed or running on the host. Launch a workshop with the SDK and the proxy starts
automatically.

---

## Reference workshop

```yaml
# .workshop.yaml
name: my-project
base: ubuntu@24.04
sdks:
  - name: omp
    channel: 14/edge
  - name: headroom-client
    channel: 0/edge
```

After `workshop launch`, all tools that honour `ANTHROPIC_BASE_URL`/`OPENAI_BASE_URL`
(omp, slupgrader, codex, OpenAI SDK, Anthropic SDK) route through the workshop-local
proxy automatically. The `headroom-home` mount plug auto-connects at launch — no manual
connection step.

---

## What the SDK installs

| Artifact | Content |
|----------|---------|
| `$SDK/python/` | Relocatable CPython 3.13 + `headroom-ai[proxy,code]==0.35.0` in its site-packages |
| `~/.config/systemd/user/headroom-proxy.service` | User service running `headroom proxy --host 127.0.0.1 --port 8787` (written by `setup-project`) |
| `/etc/profile.d/headroom.sh` | `export ANTHROPIC_BASE_URL=http://localhost:8787` and `OPENAI_BASE_URL` — picked up by every login shell |
| `/etc/environment` | Same two `KEY=VALUE` lines — picked up by PAM-based sessions and systemd units inside the workshop |

The proxy, CCR cache, and cross-agent memory live in the workshop under
`/home/workshop/.headroom`, persisted across `workshop refresh` via the `headroom-home`
mount plug.

---

## Plugs (resources this SDK consumes)

### `headroom-home`

- Interface: `mount`
- Workshop target: `/home/workshop/.headroom`
- Purpose: Persists Headroom's workspace (CCR cache, savings ledger, cross-agent memory,
  logs) across workshop updates. Auto-connected to the `system` SDK's `mount` slot at
  launch.

---

## Verifying end-to-end

```bash
# Inside the workshop
workshop shell
env | grep -E 'ANTHROPIC_BASE_URL|OPENAI_BASE_URL'
systemctl --user is-active headroom-proxy        # -> active
curl -fsS http://localhost:8787/health           # local proxy serving (HTTP 200)
curl -fsS http://localhost:8787/stats            # total_requests > 0 after an LLM task
```

The proxy binds ~10 s after launch (uvicorn cold start); `check-health` reports the SDK
healthy as soon as the service is `active`, and `/health` follows shortly after.

---

## Disabling Headroom for a workshop

Remove the `headroom-client` SDK line from the workshop definition and run
`workshop refresh`. LLM calls will then route directly to the provider APIs.

---

## Documentation and guidance

- [Headroom documentation](https://github.com/headroomlabs-ai/headroom)
- [Workshop documentation](https://ubuntu.com/workshop/docs/)
- [DEVELOPERS.md](DEVELOPERS.md) — branch model, release automation, bootstrapping a new track
- [AGENTS.md](AGENTS.md) — quick-restart context for AI coding agents working in this repo
- [ADR-0002](docs/adrs/0002-self-hosted-engine.md) — the shift from host-tunnel to a workshop-local engine

---

## License and copyright

Copyright 2026 Canonical Ltd.

This SDK is released under the [MIT License](https://opensource.org/licenses/MIT).
