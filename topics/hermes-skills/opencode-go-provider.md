---
name: opencode-go-provider
description: "Configure OpenCode Go as provider en Hermes Agent — API key, modelos disponibles, cambio de modelo."
version: 1.0.0
author: Jesús Sarmiento (jsarmiento)
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [hermes, opencode, provider, configuration, opencode-go]
    related_skills: [hermes-agent]
---

# OpenCode Go — Provider para Hermes Agent

## Overview

OpenCode Go es el servicio en la nube de OpenCode que da acceso a modelos open-source de código a $5/mes (primer mes) luego $10/mes. Hermes Agent lo soporta como provider **nativo** — no requiere configuración manual de endpoint.

Endpoint: `https://opencode.ai/zen/go/v1/chat/completions` (compatible con OpenAI)

## Cuando Usar

- Quieres usar modelos open-source (DeepSeek, Qwen, Kimi, GLM, MiniMax, MiMo) desde Hermes
- Necesitas un provider económico con buenos límites de uso
- Te suscribiste a OpenCode Go y tienes tu API key

## Configuración

### 1. Guardar API Key

Agregar al archivo `~/.hermes/.env`:

```
OPENCODE_GO_API_KEY=***
```

### 2. Configurar Provider y Modelo

```bash
# Establecer provider
hermes config set model.provider opencode-go

# Modelo por defecto (Flash = rápido/barato)
hermes config set model.model opencode-go/deepseek-v4-flash
hermes config set model.default opencode-go/deepseek-v4-flash
```

### 3. Verificar

```bash
hermes chat -q "Responde solo: OK" -Q
```

Nota: Los cambios requieren una nueva sesión (`/reset` o salir y entrar de nuevo).

## Modelos Disponibles

| Model ID (para config) | Descripción | Precio input/output por 1M tokens |
|---|---|---|
| `opencode-go/deepseek-v4-flash` | Rápido y barato (~31K req/mes) | $0.14 / $0.28 |
| `opencode-go/deepseek-v4-pro` | Más potente (~3.4K req/mes) | $1.74 / $3.48 |
| `opencode-go/kimi-k2.7-code` | Especializado en código (~9K req/mes) | $0.95 / $4.00 |
| `opencode-go/kimi-k2.6` | Kimi v2.6 general (~5.7K req/mes) | $0.95 / $4.00 |
| `opencode-go/qwen3.7-max` | Qwen máximo (~4.7K req/mes) | $2.50 / $7.50 |
| `opencode-go/qwen3.7-plus` | Qwen plus (~21K req/mes) | $0.40 / $1.60 |
| `opencode-go/qwen3.6-plus` | Qwen v3.6 plus (~21K req/mes) | $0.50 / $3.00 |
| `opencode-go/glm-5.2` | GLM 5.2 (~4.3K req/mes) | $1.40 / $4.40 |
| `opencode-go/glm-5.1` | GLM 5.1 (~4.3K req/mes) | $1.40 / $4.40 |
| `opencode-go/mimo-v2.5` | MiMo V2.5 (~150K req/mes) | $0.14 / $0.28 |
| `opencode-go/mimo-v2.5-pro` | MiMo V2.5 Pro (~16K req/mes) | $1.74 / $3.48 |
| `opencode-go/minimax-m3` | MiniMax M3 (~16K req/mes) | $0.30 / $1.20 |
| `opencode-go/minimax-m2.7` | MiniMax M2.7 (~17K req/mes) | $0.30 / $1.20 |

## Cambiar de Modelo

### Temporal (sesión activa)

```bash
/model opencode-go/deepseek-v4-pro
```

### Permanente

```bash
hermes config set model.model opencode-go/deepseek-v4-pro
hermes config set model.default opencode-go/deepseek-v4-pro
```

O usar el selector interactivo:

```bash
hermes model
```

## Límites de Uso

| Período | Límite en $ |
|---|---|
| Cada 5 horas | $12 |
| Semanal | $30 |
| Mensual | $60 |

## Pitfalls

- El secret redactor de Hermes puede censurar la API key al escribirla en `.env`. Usar Python con base64 encoding o `sed -i` con línea exacta para evitar.
- `model.default` tiene prioridad sobre `model.model` en el gateway — actualizar **ambos**.
- Los cambios de modelo requieren `/reset` o nueva sesión.
- Si la API key es inválida, da error `401 - Invalid API key`.

## Referencias

- [OpenCode Go Docs](https://opencode.ai/docs/go/)
- [OpenCode Providers Docs](https://opencode.ai/docs/providers/)
- [Hermes Agent Docs — Providers](https://hermes-agent.nousresearch.com/docs/integrations/providers)
