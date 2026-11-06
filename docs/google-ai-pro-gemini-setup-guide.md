# CLIProxyAPI — Google AI Pro + Gemini via OpenAI-Compatible API

## Recommended Project: CLIProxyAPI (v8.0.51+)

**Repository:** https://github.com/router-for-me/CLIProxyAPI
**Latest release:** v8.0.51 (2026-11-01)
**Guides:** https://help.router-for.me/

CLIProxyAPI is the most mature and feature-complete solution for accessing Google
Gemini models (including those available through Google AI Pro subscriptions) via
an OpenAI-compatible API endpoint. It handles rate limiting, multi-account load
balancing, automatic retry/failover, request cloaking, and model aliasing — all
built-in.

## Why CLIProxyAPI

| Feature | Benefit for Google AI Pro Users |
|---|---|
| **Antigravity channel** | Direct OAuth integration with Google's Antigravity service (antigravity.google), providing Gemini models to Google AI Pro and Ultra subscribers |
| **Multi-account load balancing** | Round-robin across multiple Google accounts to maximize aggregate throughput |
| **Automatic retry on 429/5xx** | When one account hits a rate limit, requests failover to the next account |
| **Model-level cooling** | Rate-limited models are cooled per-model rather than blocking the entire account |
| **Model aliasing** | Map stable client names (e.g., `gemini-flash-lite`) to the latest upstream model |
| **Request cloaking** | Requests appear to come from legitimate Antigravity/Gemini CLI clients |
| **OpenAI-compatible API** | Any OpenAI SDK or client can connect using `/v1/chat/completions` or `/v1/responses` |
| **Session affinity** | Multi-turn conversations stay on the same account for KV-cache reuse |
| **Connection pooling** | Reuses HTTP connections to reduce latency and avoid throttling |

## Quick Start

### 1. Install

```bash
# Download latest release
gh release download v8.0.51 --repo router-for-me/CLIProxyAPI --pattern '*linux*amd64*'

# Or build from source (Go 1.26+)
git clone https://github.com/router-for-me/CLIProxyAPI.git
cd CLIProxyAPI && go build -o cli-proxy-api ./cmd/server
```

### 2. Configure

```bash
cp examples/google-ai-pro/config-google-ai-pro.yaml config.yaml
# Edit: set your API key under access.api-keys
```

### 3. Authenticate with Google (Antigravity)

```bash
./cli-proxy-api --config config.yaml --tui
```

In the TUI: **Antigravity -> Login** -> sign in with Google AI Pro account.
Repeat for each Google account you want to pool.

### 4. Start the Server

```bash
./cli-proxy-api --config config.yaml &
```

### 5. Connect from Any OpenAI Client

```bash
export OPENAI_BASE_URL=http://localhost:8317/v1
export OPENAI_API_KEY=sk-cpa-gemini-your-secret-key-here

curl $OPENAI_BASE_URL/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"gemini-flash-lite","messages":[{"role":"user","content":"Hello!"}],"stream":true}'
```

## Key Configuration Explained

### Model Aliases (`oauth.model-alias`)

Maps stable client names to actual upstream models:
- `gemini-flash-lite` -> `gemini-3.8-flash` (update `name` when Google releases a new Flash)
- `gemini-argon-latest` -> `gemini-4-argon`
- `gemini-pro-latest` -> `gemini-pro-agent`

With `force-mapping: true`, responses always show the alias name to clients.

### Retry and Failover (`routing.retry`)

- `request-retry: 3` — On failure, try up to 3 additional accounts
- Applies to HTTP 403, 408, 429, 500, 502, 503, 504
- `max-retry-interval: 30` — max wait between retries

### Model-Level Cooling (`oauth.providers.antigravity.model-level-cooling`)

When one model hits quota on an account, only that model is cooled — other
models on the same account remain available.

### Request Cloaking (`oauth.providers.antigravity.sensitive-words`)

Obfuscates words like "API", "proxy" in system instructions using zero-width
characters to reduce detection by upstream service.

### Session Affinity (`routing.session-affinity`)

Keeps multi-turn conversations on the same account for KV-cache reuse and
consistency. Subagents inherit the parent's account binding.

## Companion Tools

| Tool | Purpose |
|---|---|
| [CPA-Manager-Plus](https://github.com/seakev/CPA-Manager-Plus) | Web dashboard for accounts, quotas, usage |
| [Oh-My-CPA](https://github.com/WizisCool/oh-my-cpa) | Ant Design management console |
| [EasyCLIProxyAPI](https://github.com/router-for-me/EasyCLIProxyAPI) | Desktop GUI client |
| [CPA Usage Keeper](https://github.com/Willxup/cpa-usage-keeper) | Usage visualization |

## Troubleshooting

| Problem | Solution |
|---|---|
| 429 Too Many Requests | Add more Google accounts or increase `request-retry` |
| 403 Forbidden | Re-authenticate the account via TUI |
| Model not found | Check `oauth.model-alias`; verify model in Antigravity |
| High latency | Enable `connection-pool`; check proxy settings |
| Cooling too aggressive | Set `model-level-cooling: true`, adjust `transient-error-cooldown-seconds` |
| Response shows upstream model ID | Enable `force-mapping: true` on aliases |


## Running on Android Termux

### Feasibility: YES, with caveats

CLIProxyAPI can run on Android via Termux. Go cross-compiles for Android ARM64.
However, several Android-specific considerations apply.

### Requirements

- Android 10+, ARM64 (aarch64)
- Termux from F-Droid (NOT Google Play — outdated)
- 8GB RAM sufficient; CLIProxyAPI uses ~50-100MB idle
- ~50MB storage for binary + config + auth tokens

### 1. Install Termux and Dependencies

```bash
pkg update && pkg upgrade -y
pkg install -y golang git wget curl
```

### 2. Build (Static Binary Required)

```bash
# In Termux:
git clone https://github.com/router-for-me/CLIProxyAPI.git
cd CLIProxyAPI
CGO_ENABLED=0 go build -ldflags="-s -w" -o cli-proxy-api ./cmd/server
```

**Important:** `CGO_ENABLED=0` is required for a static binary.
Official release binaries use `CGO_ENABLED=1` (glibc) and may NOT work on Termux.

### 3. Configure

```bash
mkdir -p ~/.cli-proxy-api ~/.config/cliproxyapi
cp config.example.yaml ~/.config/cliproxyapi/config.yaml
# Edit: set server.host to "0.0.0.0" for LAN access
# Edit: set oauth.auth-dir to "~/.cli-proxy-api"
```

### 4. Run as Persistent Service

Termux has no systemd. Use one of:

**Option A: termux-services (recommended)**
```bash
pkg install -y termux-services
mkdir -p $PREFIX/var/service/cliproxyapi
cat > $PREFIX/var/service/cliproxyapi/run <<'EOF'
#!/data/data/com.termux/files/usr/bin/sh
exec 2>&1
exec $HOME/cli-proxy-api --config $HOME/.config/cliproxyapi/config.yaml
EOF
chmod +x $PREFIX/var/service/cliproxyapi/run
sv-enable cliproxyapi
sv up cliproxyapi
```

**Option B: tmux**
```bash
pkg install -y tmux
tmux new-session -d -s cliproxyapi \
  "$HOME/cli-proxy-api --config $HOME/.config/cliproxyapi/config.yaml"
```

**Option C: nohup (simplest)**
```bash
nohup $HOME/cli-proxy-api --config $HOME/.config/cliproxyapi/config.yaml \
  > $HOME/cliproxyapi.log 2>&1 &

### 5. Android-Specific Considerations

**Battery Optimization (CRITICAL):** Android Doze kills background processes.
- Settings -> Apps -> Termux -> Battery -> Unrestricted
- Xiaomi MIUI: Settings -> Apps -> Termux -> Battery saver -> No restrictions
- Run `termux-wake-lock` to keep CPU awake

**Auto-start on Boot:** Termux has no native autostart.
- Install Termux:Boot (F-Droid), create `~/.termux/boot/start.sh`:
  ```bash
  #!/data/data/com.termux/files/usr/bin/sh
  termux-wake-lock
  $HOME/cli-proxy-api --config $HOME/.config/cliproxyapi/config.yaml &
  ```

**OAuth on Android:** Use `--no-browser` flag and manually open the auth URL.
Install `termux-api` for browser integration: `pkg install -y termux-api`

**Network Access:** Set `server.host: "0.0.0.0"` in config. Access from LAN:
`http://<tablet-ip>:8317/v1/chat/completions`

### 6. Performance on 8GB Device

| Metric | Value |
|---|---|
| Binary size | ~30-40MB (stripped) |
| RAM (idle) | ~50-80MB |
| RAM (under load) | ~100-200MB |
| Startup time | ~1-2 seconds |
| Battery impact | Moderate |

### 7. Limitations on Android/Termux

| Limitation | Mitigation |
|---|---|
| No systemd | Use termux-services or tmux |
| Doze kills background apps | Battery whitelist + wake lock |
| No auto-start on boot | Termux:Boot app |
| OAuth callback port conflicts | Use `--oauth-callback-port` |
| Thermal throttling | Monitor temperature |
| Mobile data restrictions | Use Wi-Fi or Cloudflare Tunnel |

### 8. Recommended for "Always Available"

For maximum reliability:
1. Whitelist Termux from ALL battery optimizations
2. Use termux-services for process supervision (auto-restart on crash)
3. Use Termux:Boot for auto-start after reboot
4. Keep wake lock active: `termux-wake-lock`
5. Health check: `*/5 * * * * curl -sf http://localhost:8317/health || sv restart cliproxyapi`
6. Optional: Cloudflare Tunnel for external access without port forwarding

### Summary

| Aspect | Verdict |
|---|---|
| Can it run? | YES, with CGO_ENABLED=0 static build |
| Always available? | Possible with battery whitelist + wake lock + supervision |
| Reliable for agents? | YES if Doze disabled and service supervised |
| Performance | Adequate for 8GB RAM device |
| Recommendation | Use as supplementary/backup node; primary on Linux VPS is more reliable for "always available" production use |

**Bottom line:** Works well on Termux for development and light usage. For truly "always available" production use (OpenClaw agents, LLM services), a Linux VPS or dedicated server is more reliable. The Android tablet is best as a supplementary node or for testing.
```

