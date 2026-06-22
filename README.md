# Headroom client SDK for Workshop

This SDK routes a workshop's AI agent traffic through a
[Headroom](https://github.com/headroomlabs-ai/headroom) compression proxy running on the
host. It sets `ANTHROPIC_BASE_URL` and `OPENAI_BASE_URL` so every in-workshop LLM call
is compressed before leaving the container.

---

## Prerequisites

The Headroom proxy must be running on the **host** before the tunnel is useful:

```bash
# Install once on the host (build-essential required for hnswlib)
uv tool install "headroom-ai[proxy,code]"

# Persistent systemd user service
cat > ~/.config/systemd/user/headroom-proxy.service << 'EOF'
[Unit]
Description=Headroom compression proxy (host-wide, shared by Workshops)
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=%h/.local/bin/headroom proxy --host 127.0.0.1 --port 8787
Restart=always
RestartSec=3
Environment=HEADROOM_TELEMETRY=off

[Install]
WantedBy=default.target
EOF

loginctl enable-linger "$USER"
systemctl --user daemon-reload
systemctl --user enable --now headroom-proxy
curl -fsS http://127.0.0.1:8787/health
```

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
  - name: system
    slots:
      headroom:
        interface: tunnel
        endpoint: localhost:8787    # host headroom proxy
connections:
  - plug: headroom-client:headroom
    slot: system:headroom
actions:
  connect-headroom: |              # fallback if connections: is not auto-applied
    workshop connect my-project/headroom-client:headroom my-project/system:headroom
```

After `workshop launch`, all tools that honour `ANTHROPIC_BASE_URL`/`OPENAI_BASE_URL`
(omp, slupgrader, codex, OpenAI SDK, Anthropic SDK) will route through the proxy
automatically.

---

## What the SDK installs

The SDK writes two environment-injection artifacts inside the workshop:

| File | Content |
|------|---------|
| `/etc/profile.d/headroom.sh` | `export ANTHROPIC_BASE_URL=http://localhost:8787` and `OPENAI_BASE_URL` — picked up by every login shell |
| `/etc/environment` | Same two `KEY=VALUE` lines — picked up by PAM-based sessions and systemd units inside the workshop |

No binary is installed. The proxy, CCR cache, and cross-agent memory live on the host
and are shared across every workshop on the machine.

---

## Plugs (resources this SDK consumes)

### `headroom`

- Interface: `tunnel`
- Endpoint: `localhost:8787` (workshop-side listen address)
- Purpose: Forwards the workshop's port 8787 to the host's Headroom proxy at
  `127.0.0.1:8787`. Must be connected to a `system` SDK slot that points at the host
  proxy.

---

## Connecting the tunnel

Because the host-side slot uses stricter validation, the tunnel connection is not
auto-connected at launch. Verify with `workshop info <ws>` and connect manually if
needed:

```bash
workshop connect my-project/headroom-client:headroom my-project/system:headroom
```

Or add a `connect-headroom` action to your workshop definition and run:

```bash
workshop run my-project connect-headroom
```

---

## Verifying end-to-end

```bash
# Inside the workshop
workshop shell
env | grep -E 'ANTHROPIC_BASE_URL|OPENAI_BASE_URL'
curl -fsS http://localhost:8787/health    # proves the tunnel forwards to the host

# On the host, after running an LLM task:
curl -fsS http://127.0.0.1:8787/stats    # total_requests > 0 and tokens_saved > 0
```

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

---

## License and copyright

Copyright 2026 Canonical Ltd.

This SDK is released under the [MIT License](https://opensource.org/licenses/MIT).
