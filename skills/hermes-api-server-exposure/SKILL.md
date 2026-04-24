---
name: hermes-api-server-exposure
description: Expose Hermes gateway API Server (default 127.0.0.1:8642) on 0.0.0.0 for access from WSL host, Windows, LAN, or other agents (e.g. OpenClaw FLAC). Covers the specific config precedence that caught us — .env overrides systemd env, config.yaml gateway.api_server fields are ignored.
---

# Exposing Hermes API Server (OpenAI-compatible endpoint)

## When to use
- Another agent (e.g. OpenClaw's FLAC on Windows side) needs to call Hermes via HTTP
- You want to reach the Hermes gateway from Windows localhost / LAN / remote
- Default 127.0.0.1 binding is blocking access

## Hermes API Server basics
- Started by `hermes-gateway.service` (systemd)
- Default listen: `127.0.0.1:8642`
- OpenAI-compatible: `POST /v1/chat/completions`, `GET /v1/models`
- Records every request to session DB (queryable via Web UI, `hermes sessions list --platform api`)
- **Safety gate**: binding to non-loopback REQUIRES `API_SERVER_KEY` set (hard refusal if empty — see `gateway/platforms/api_server.py:2522`)

## Config precedence (what actually matters)

**This caught us:** Hermes has THREE places that could configure API Server host/key, but only ONE actually works.

| Source | Works? | Priority |
|---|---|---|
| `/root/.hermes/.env` (`API_SERVER_HOST=...`) | ✅ YES | **Highest — wins everything else** |
| `systemd` service `Environment=` (drop-in override) | ⚠️ Gets loaded into process env, but `.env` is read AFTER by hermes and overwrites | Middle |
| `config.yaml` `gateway.api_server.host` / `.key` | ❌ **Ignored** — Hermes reads only env vars (see `gateway/config.py:1046-1070`) | None |

**Lesson: always check `~/.hermes/.env` first.** The Web UI setup wizard writes `API_SERVER_HOST=127.0.0.1` there on first install, and nothing else can override it.

## Procedure

### 1. Generate an API key (never skip)
```bash
python3 -c "import secrets; print('hermes_' + secrets.token_urlsafe(32))"
```

### 2. Edit `~/.hermes/.env`
```bash
# Back up first
cp /root/.hermes/.env /root/.hermes/.env.bak

# Change these two lines in place
sed -i 's|^API_SERVER_HOST=.*|API_SERVER_HOST=0.0.0.0|' /root/.hermes/.env
sed -i 's|^API_SERVER_KEY=.*|API_SERVER_KEY=<PASTE_NEW_KEY>|' /root/.hermes/.env
```

Verify:
```bash
grep ^API_SERVER /root/.hermes/.env
# Should show:
#   API_SERVER_ENABLED=true
#   API_SERVER_HOST=0.0.0.0
#   API_SERVER_PORT=8642
#   API_SERVER_KEY=hermes_...
```

### 3. Restart gateway and verify binding
```bash
systemctl restart hermes-gateway
sleep 6
ss -tlnp | grep 8642
# MUST show: 0.0.0.0:8642 (not 127.0.0.1:8642)
```

If still 127.0.0.1 → the `.env` edit didn't stick, or there's a second `.env` being loaded. Check `HERMES_HOME` env var of the running process:
```bash
cat /proc/$(pgrep -f 'gateway run' | head -1)/environ | tr '\0' '\n' | grep -E "HERMES_HOME|API_SERVER"
```

### 4. Get the WSL IP (if exposing for Windows host)
```bash
hostname -I | awk '{print $1}'
# e.g. 172.21.217.245
```

WSL IP changes on reboot. For stable access from Windows, either:
- Use `$(wsl hostname -I)` dynamically in Windows scripts
- Set up Windows `netsh portproxy` pointing a fixed localhost port to current WSL IP (refresh on boot)
- Rely on WSL2 localhost-forwarding (unreliable — this is why we bind 0.0.0.0 in the first place)

### 5. Smoke test
```bash
WSL_IP=$(hostname -I | awk '{print $1}')
API_KEY=$(grep ^API_SERVER_KEY /root/.hermes/.env | cut -d= -f2)
curl -s -X POST http://$WSL_IP:8642/v1/chat/completions \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"hermes-agent","messages":[{"role":"user","content":"ping"}]}'
```

Should return a JSON response with the assistant reply.

## Pitfalls

1. **Don't bother editing `config.yaml gateway.api_server.host`** — Hermes doesn't read it. Wasted 10 minutes chasing this. The only config.yaml key that matters here is `gateway.api_server.enabled: true` (tells the gateway to load the platform at all).

2. **Don't rely on systemd `Environment=` override** — even when `/proc/PID/environ` shows the correct value, `.env` gets loaded later in Hermes startup and silently overrides. Verify by looking at what the running process ACTUALLY did (check `ss -tlnp`), not just env.

3. **`API_SERVER_KEY=""` (empty) + `API_SERVER_HOST=0.0.0.0`** = gateway refuses to start (silent in logs, just stays on 127.0.0.1). Always set a key.

4. **Don't bind 0.0.0.0 without a firewall** if the host is on a public network. For WSL2 this is fine (Windows NAT shields it), but a LAN-reachable Linux server needs `ufw` rules.

5. **After upgrading Hermes** the Web UI setup wizard may re-run and rewrite `.env` back to `API_SERVER_HOST=127.0.0.1`. Check `.env` after any upgrade.

## Multi-agent use case (OpenClaw FLAC → Hermes)

Once exposed, other agents can call Hermes as a standard OpenAI endpoint:

```python
# FLAC side (Windows)
import openai
client = openai.OpenAI(
    base_url="http://172.21.217.245:8642/v1",
    api_key="hermes_...",
)
resp = client.chat.completions.create(
    model="hermes-agent",
    messages=[{"role": "user", "content": "approved payment 20260424T...json — please execute"}]
)
```

Every such call creates a session in `~/.hermes/response_store.db`, visible in:
- Web UI session list (tagged with platform=api_server)
- `hermes sessions list --platform api`
- Asked from CLI: "what did FLAC ask me today?"

## Verification log (2026-04-24, real incident)

User asked to open API for FLAC (treasurer agent on Windows). Steps that FAILED:
1. Tried systemd drop-in `/etc/systemd/system/hermes-gateway.service.d/10-api-server.conf` with `Environment="API_SERVER_HOST=0.0.0.0"` — env injected into process (confirmed via `/proc/*/environ`), but bind remained 127.0.0.1.
2. Tried `config.yaml` adding `gateway.api_server: {host: 0.0.0.0, key: ...}` — Hermes ignored, still 127.0.0.1. Source check confirmed: `gateway/config.py:1046-1070` only reads env vars, never reads these YAML keys.
3. Finally checked `~/.hermes/.env`, found hardcoded `API_SERVER_HOST=127.0.0.1` from Web UI setup wizard. Edited → restart → bind changed to 0.0.0.0 immediately.

**Total time wasted on wrong paths: ~15 minutes.** Going forward, check `.env` FIRST.
