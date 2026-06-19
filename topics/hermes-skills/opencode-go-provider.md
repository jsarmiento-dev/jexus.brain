---
name: opencode-go-provider
description: "Configure OpenCode Go as provider en Hermes Agent — API key, modelos disponibles, cambio de modelo."
version: 1.1.0
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

## Modelos Disponibles (ordenados por capacidad)

Los primeros 20 modelos detectados vía API. ID para config: `opencode-go/<model-id>`.

### 🏆 Tier 1 — Frontier (Código complejo + razonamiento profundo)

| # | Modelo | Precio I/O /1M | Capacidad | Límite/mes | Ideal para |
|---|--------|---------------|-----------|-----------|------------|
| 1 | `deepseek-v4-pro` | $1.74 / $3.48 | **80.6% SWE-bench**. Top coding complejo. Comparable a Claude Opus. | ~17K req | Código complejo, agentes autónomos, refactors grandes |
| 2 | `glm-5.2` | $1.40 / $4.40 | **1M contexto**, beats GPT-5.5 en long-horizon coding. #1 Design Arena. | ~4.3K req | Proyectos enormes, contexto completo del codebase |
| 3 | `kimi-k2.7-code` | $0.95 / $4.00 | **81.1 Kimi Code Bench**. Especialista puro coding. 30% menos thinking tokens que K2.6. | ~9.2K req | Programar, debugging, código agente |

### 🥈 Tier 2 — Fuertes (Muy capaces, versátiles)

| # | Modelo | Precio I/O /1M | Capacidad | Límite/mes | Ideal para |
|---|--------|---------------|-----------|-----------|------------|
| 4 | `qwen3.7-max` | $2.50 / $7.50 | Flagship Alibaba. Coding + debugging + workflows + ofimática agente. | ~4.7K req | Tareas variadas, máximo rendimiento general |
| 5 | `minimax-m3` | $0.30 / $1.20 | **59% SWE-Bench Pro**, 83.5 BrowseComp. Junio 2026. Buen costo/performance. | ~16K req | Coding sólido a bajo precio |
| 6 | `deepseek-v4-flash` | $0.14 / $0.28 | **79% SWE-bench**. Casi tan bueno como Pro a 1/12 del precio. | ~158K req | **Uso diario** — mejor relación calidad/precio |
| 7 | `glm-5.1` | $1.40 / $4.40 | Predecesor GLM-5.2. Sólido all-rounder con 1M contexto. | ~4.3K req | Contexto largo sin pagar 5.2 |
| 8 | `kimi-k2.6` | $0.95 / $4.00 | Multimodal (txt+img). Coding + agente. Predecesor del K2.7. | ~5.7K req | Multimodal + código |

### 🥉 Tier 3 — Buenos (Calidad/precio sólido)

| # | Modelo | Precio I/O /1M | Capacidad | Límite/mes | Ideal para |
|---|--------|---------------|-----------|-----------|------------|
| 9 | `mimo-v2.5-pro` | $1.74 / $3.48 | Xiaomi frontier. 42 AA Index. Thinking mode coding. | ~16K req | Coding con razonamiento profundo |
| 10 | `qwen3.7-plus` | $0.40 / $1.60 | Multimodal (txt+video+img). Mid-size, buena relación. | ~21K req | Tareas multimodales, workflows generales |
| 11 | `hy3-preview` | Gratis 2 sem | Tencent **295B (A21B)**, 256K contexto, reasoning + agente. | ? | Probar, experimentar gratis |
| 12 | `minimax-m2.7` | $0.30 / $1.20 | Predecesor M3. Coding sólido a bajo costo. | ~17K req | Backend económico |
| 13 | `qwen3.6-plus` | $0.50 / $3.00 | Prev-gen Qwen, aún competente. | ~16K req | Tareas generales |

### 🔹 Tier 4 — Económicos (Para volumen, tareas simples)

| # | Modelo | Precio I/O /1M | Capacidad | Límite/mes | Ideal para |
|---|--------|---------------|-----------|-----------|------------|
| 14 | `mimo-v2.5` | $0.14 / $0.28 | Barato con **150K req/mes**. | ~150K req | Bulk, tareas repetitivas, batch |
| 15 | `mimo-v2-pro` | ? | MiMo mid-tier con thinking mode. | ? | Coding ligero con razonamiento |
| 16 | `qwen3.5-plus` | $0.50 / $3.00 | Qwen generación anterior. | ~16K req | Tareas básicas |
| 17 | `glm-5` | $1.40 / $4.40 | Entry GLM, 1M contexto. | ~4.3K req | Contexto largo básico |
| 18 | `mimo-v2-omni` | ? | Multimodal MiMo V2 (txt+img). | ? | Multimodal económico |
| 19 | `kimi-k2.5` | $0.95 / $4.00 | Kimi generación anterior. | ~5.7K req | Backup código |
| 20 | `minimax-m2.5` | $0.30 / $1.20 | Entry MiniMax. El más básico. | ~17K req | Tareas muy simples |

### Recomendación rápida

| Para esto | Usa |
|-----------|------|
| **Uso diario** 🏆 | `deepseek-v4-flash` — 79% SWE con 158K req/mes |
| **Código complejo** 🔥 | `deepseek-v4-pro`, `glm-5.2` o `kimi-k2.7-code` |
| **Proyecto enorme** 📦 | `glm-5.2` — 1M contexto |
| **Barato + volumen** 💰 | `mimo-v2.5` — $0.14/$0.28, 150K req |
| **Gratis** 🆓 | `hy3-preview` — 2 semanas gratis |
| **Multimodal** 🖼️ | `qwen3.7-plus` o `kimi-k2.6` |

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
