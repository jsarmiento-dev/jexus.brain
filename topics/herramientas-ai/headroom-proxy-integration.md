# Headroom Proxy Integration for Hermes Agent

> **Integración:** Headroom v0.26.0 + Hermes Agent + OpenCode Go
> **Fecha:** Junio 2026
> **Skill Hermes:** `headroom-proxy-integration`
> **Archivo fuente skill:** `~/.hermes/skills/devops/headroom-proxy-integration/SKILL.md`

---

## Arquitectura

```
Hermes Agent
  → model.base_url = http://localhost:8787/v1
    → Headroom Proxy (port 8787, systemd service)
      → OPENAI_TARGET_API_URL = https://opencode.ai/zen/go/v1
        → OpenCode Go
```

## Comandos de Setup

### 1. Instalar Headroom

```bash
uv venv -p 3.11 ~/.headroom-venv
source ~/.headroom-venv/bin/activate
uv pip install "headroom-ai[proxy]"
```

### 2. Systemd Service

Archivo: `~/.config/systemd/user/headroom-proxy.service`

```ini
[Unit]
Description=Headroom Context Compression Proxy
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/home/ubuntu/.headroom-venv/bin/headroom proxy --port 8787 --no-telemetry
Restart=always
RestartSec=5
Environment=OPENAI_TARGET_API_URL=https://opencode.ai/zen/go/v1
Environment=HEADROOM_CCR_TTL_SECONDS=3600
Environment=HEADROOM_OUTPUT_SHAPER=1
Environment=HEADROOM_VERBOSITY_LEVEL=2

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable headroom-proxy.service
systemctl --user start headroom-proxy.service
```

### 3. Configurar Hermes

```bash
hermes config set model.base_url http://localhost:8787/v1
```

## Detalles Técnicos

### Por qué native provider y no custom provider

El native `opencode-go` provider de Hermes lee la API key de `~/.hermes/.env` correctamente. Los custom providers NO resuelven la API key desde `.env` para el auth header. Por eso se usa:

```yaml
model:
  provider: opencode-go          # nativo, no custom
  base_url: http://localhost:8787/v1  # ← redirige al proxy
  model: opencode-go/deepseek-v4-flash
```

### Features activadas de Headroom

- **Output Shaper (L2):** Reduce respuestas del modelo ~22-65%
- **CCR (TTL=3600s):** Compresión reversible con 1h de caché
- **SmartCrusher:** Compresión JSON de tool outputs (60-92%)
- **CodeCompressor:** Compresión AST-aware de código (40-70%)
- **CacheAligner:** Estabiliza prefijos para hits de KV cache

### Endpoints disponibles

| Endpoint | Propósito |
|---|---|
| `http://localhost:8787/health` | Health check |
| `http://localhost:8787/stats` | Estadísticas de compresión |
| `http://localhost:8787/metrics` | Prometheus |

## Verificación

```bash
# Proxy vivo?
curl -s http://localhost:8787/health | python3 -c "import json,sys;d=json.load(sys.stdin);print(d['status'])"

# Hermes responde a través del proxy?
hermes chat -q "Responde solo: OK" -Q

# Stats de compresión?
curl -s http://localhost:8787/stats | python3 -c "import json,sys;d=json.load(sys.stdin);t=d['agent_usage']['totals'];print(f\"Requests: {t['requests']}, Saved: {t['tokens_saved']} ({t['savings_percent']}%)\")"
```

## Notas

- La compresión real se ve con tool outputs grandes (JSON, code search, logs), no con prompts pequeños de texto.
- Hermes tiene su propia compresión nativa (`context.engine: compressor`) que comprime el historial de conversación. Headroom es complementario: comprime tool outputs individuales antes de que entren al historial.
- Si el proxy se cae, Hermes no puede hacer requests. Verificar con `systemctl --user status headroom-proxy.service`.
