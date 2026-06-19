# Headroom: Context Compression Layer para AI Agents

> **Repositorio:** [chopratejas/headroom](https://github.com/chopratejas/headroom)
> **Estrellas:** 35.3k ⭐ | **Forks:** 2.4k | **Licencia:** Apache 2.0
> **Lenguajes:** Python (78.7%), Rust (16.8%), TypeScript (2.4%)
> **Versión latest:** v0.26.0 | **Commits:** 1,640 | **Contribuidores:** 108
> **Análisis:** Junio 2026

---

## Índice

1. [Visión General](#1-visión-general)
2. [Problema que Resuelve](#2-problema-que-resuelve)
3. [Arquitectura y Pipeline](#3-arquitectura-y-pipeline)
4. [Modos de Despliegue](#4-modos-de-despliegue)
5. [Algoritmos de Compresión en Detalle](#5-algoritmos-de-compresión-en-detalle)
6. [Output Token Reduction](#6-output-token-reduction)
7. [CCR: Compress-Cache-Retrieve](#7-ccr-compress-cache-retrieve)
8. [Cross-Agent Memory](#8-cross-agent-memory)
9. [Headroom Learn](#9-headroom-learn)
10. [Métricas y Benchmarks](#10-métricas-y-benchmarks)
11. [Integración con Agentes](#11-integración-con-agentes)
12. [Rust Migration & Performance](#12-rust-migration--performance)
13. [Relevancia para Hermes / Nuestro Stack](#13-relevancia-para-hermes--nuestro-stack)
14. [Análisis Costo-Beneficio](#14-análisis-costo-beneficio)
15. [Recomendaciones Prácticas](#15-recomendaciones-prácticas)

---

## 1. Visión General

Headroom es una **capa de compresión de contexto local-first** que se interpone entre agentes de IA y proveedores LLM. Comprime **tool outputs, logs, archivos, RAG chunks, imágenes e historial de conversación** antes de que lleguen al LLM, manteniendo la fidelidad de las respuestas.

> **Claim principal:** "60–95% fewer tokens · library · proxy · MCP · 6 algorithms · local-first · reversible"

Su valor diferencial vs. soluciones caseras (truncamiento, resúmenes manuales) es que usa **compresión reversible (CCR)**: los originales se almacenan localmente y el LLM puede recuperarlos on-demand via tool call si necesita el detalle completo.

### Filosofía de Diseño

- **Local-first:** Todo corre en tu máquina, nada se envía a terceros
- **Reversible:** Nada se pierde permanentemente, el LLM puede pedir el original
- **Transparente:** Proxy drop-in, zero code changes para la mayoría de integraciones
- **Especializado:** No es compresión genérica (gzip), sino compresión semántica con conocimiento del dominio (AST-aware, JSON-aware, agentic-traces-aware)

---

## 2. Problema que Resuelve

### El Cuello de Botella del Contexto

Los agentes de IA modernos (Claude Code, Cursor, Codex, Copilot) consumen tokens de forma masiva:

| Actividad | Tokens | Costo aprox (Claude Sonnet) |
|---|---|---|
| Code search (100 results) | 17,765 | ~$0.05 |
| SRE incident debugging | 65,694 | ~$0.20 |
| GitHub issue triage | 54,174 | ~$0.16 |
| Codebase exploration | 78,502 | ~$0.24 |

Con **DeepSeek V3/V4 soportando 1M tokens** (contexto actualizado por Headroom v0.26), los límites de contexto son menos restrictivos, pero el **costo por sesión** y la **velocidad** siguen siendo problemas reales.

### Por qué no basta con truncar

El truncamiento simple pierde información crítica. Headroom usa **compresión inteligente**:

1. **Detección de tipo de contenido** → router selecciona compresor especializado
2. **Relevancia contextual** → el SmartCrusher puntúa ítems por relevancia a la query del usuario
3. **Compresión reversible** → si algo se perdió, el LLM puede recuperarlo

---

## 3. Arquitectura y Pipeline

### Pipeline de Vida

```
Setup → Pre-Start → Post-Start → Input Received → Input Cached →
Input Routed → Input Compressed → Input Remembered →
Pre-Send → Post-Send → Response Received
```

El pipeline tiene **hooks extensibles** via `on_pipeline_event(...)` donde plugins y extensiones pueden intervenir en cualquier etapa.

### Componentes Core

| Componente | Rol | Tipo de Contenido |
|---|---|---|
| **ContentRouter** | Detecta tipo de contenido y selecciona compresor | Enrutador universal |
| **SmartCrusher** | Compresión JSON universal (arrays, objetos anidados, tipos mixtos) | JSON / estructurado |
| **CodeCompressor** | Compresión AST-aware para lenguajes de programación | Código fuente |
| **Kompress-base** | Modelo HuggingFace entrenado en traces agentic | Texto general |
| **CacheAligner** | Estabiliza prefijos para maximizar hits en KV cache del provider | Prefijos de prompt |
| **IntelligentContext** | Score-based fitting con importancia aprendida | Importancia relativa |
| **Image Compressor** | Router ML entrenado para compresión de imágenes | Imágenes (40-90%) |
| **CCR** | Compress-Cache-Retrieve: almacena originales localmente | Reversibilidad |

### Flujo de Datos

```
Tool Output → CacheAligner (estabiliza prefijo KV)
           → ContentRouter
               ├─ JSON? → SmartCrusher
               ├─ Código? → CodeCompressor (AST)
               ├─ Imagen? → ImageCompressor
               └─ Texto? → Kompress-base
           → CCR (almacena original, emite hash retrievable)
           → IntelligentContext (score/ranking)
           → Prompt comprimido → LLM
```

---

## 4. Modos de Despliegue

| Modo | Comando | Uso | Mejor para |
|---|---|---|---|
| **Library** | `from headroom import compress` | Inline en cualquier app Python/TS | Integración programática |
| **Proxy** | `headroom proxy --port 8787` | Zero code changes, compatible OpenAI | Cualquier agente/CLI |
| **Agent Wrap** | `headroom wrap claude\|codex\|cursor\|aider\|copilot` | Wrapper automático | Coding agents |
| **MCP Server** | `headroom mcp install` | Tools: compress, retrieve, stats | Clientes MCP |
| **Cross-Agent Memory** | `headroom proxy --memory` | Store compartido entre agentes | Equipos multi-agente |

### Proxy Mode (el más relevante para nosotros)

El proxy es un servidor HTTP que implementa la API compatible con OpenAI `/v1/chat/completions` y `/v1/messages` (Anthropic).

```
headroom proxy --port 8787 --provider anthropic
# Luego apuntas tu agente a http://localhost:8787
```

Características del proxy:
- **Hot-reload** de variables de entorno via `POST /admin/runtime-env` (sin restart)
- **Output Shaping** integrado (reduce tokens de salida)
- **Compresión reversible** con CCR
- **Cache de prefijos** via CacheAligner
- **Métricas** en `/metrics` (Prometheus)
- **Health check** en `/health`

### Agent Wrap (para Claude Code específicamente)

```bash
headroom wrap claude
```

Esto:
1. Inicia el proxy local (o se conecta a uno existente)
2. Configura Claude Code para usar el proxy como intermediario
3. Aplica compresión en cada tool call
4. Inyecta las tools MCP de Headroom (compress, retrieve, stats)

Soporta:
- `--memory` → cross-agent memory
- `--code-graph` → grafo de dependencias de código
- Vertex AI → `CLAUDE_CODE_USE_VERTEX=1` detectado automáticamente

---

## 5. Algoritmos de Compresión en Detalle

### 5.1 ContentRouter

El router de contenido analiza el input y determina qué compresor usar basado en:

- **Estructura JSON** → SmartCrusher
- **Extensiones de archivo** → Python, JS, TS, Go, Rust, Java, C++ → CodeCompressor
- **Tipo MIME** → imágenes, texto plano, markdown
- **Heurísticas de contenido** → detección de tool outputs, logs, RAG chunks

Es el primer paso del pipeline y decide la ruta de compresión.

### 5.2 SmartCrusher (JSON)

El compresor universal para JSON, diseñado específicamente para tool outputs que son estructuras JSON complejas.

**Técnicas:**
- **Array summarization:** Arrays grandes (>N elementos) se resumen estadísticamente (min, max, avg, count, pattern detection)
- **Key elision:** Keys redundantes o predecibles se omiten
- **Nested flattening:** Objetos anidados se aplanan eliminando nesting innecesario
- **Type-aware formatting:** Números, strings, booleans se serializan en formato compacto
- **Deduplication:** Objetos repetidos se reemplazan con referencias
- **Relevance Scoring:** Puntúa cada ítem por relevancia a la query del usuario (`_extract_user_query()`)

**Caso de uso real:** 12 RAG chunks → SmartCrusher sin contexto perdió ítems clave. Con `_extract_user_query()` y context-aware scoring, mantuvo 3/4 términos clave vs 0/6.

### 5.3 CodeCompressor (AST-aware)

Compresión especializada para código fuente usando **Abstract Syntax Trees** (Tree-sitter).

**Técnicas:**
- **Function signature extraction:** Solo firma + docstring, cuerpo comprimido
- **Import collapsing:** `from x import a, b, c, d, e` → `from x import a, b, ...`
- **Type annotation stripping:** En contextos donde no se necesita
- **Dead code elimination:** Bloques `if False:`, funciones no llamadas
- **Whitespace normalization:** Mínimo espaciado legible
- **AST-aware minification:** Reduce identificadores largos manteniendo bindings

**Resultados:**
- Código Python: 45-70% reducción
- Código Go: 50-65% reducción
- Código Rust: 40-60% reducción
- Benchmarks: **0% pérdida de precisión** en GSM8K, BFCL

**Fix importante:** Tree-sitter thread safety (ThreadPoolExecutor panic fix) — necesario para entornos multi-thread.

### 5.4 Kompress-base (HF Model)

Modelo HuggingFace (`chopratejas/kompress-v2-base`) entrenado específicamente en **traces de agentes**:

- **Dataset:** miles de sesiones reales de Claude Code, Codex, Cursor
- **Arquitectura:** Transformer encoder-decoder para compresión textual
- **Entrenamiento:** Loss diseñada para preservar información crítica para decisión del agente
- **Uso:** Compresión de texto libre, logs, documentación

**Precisión preservada:**
- GSM8K: 0.870 → 0.870 (±0.000)
- TruthfulQA: 0.530 → 0.560 (+0.030)
- SQuAD v2: 97% @ 19% compresión

### 5.5 CacheAligner

Componente que **estabiliza prefijos** para maximizar hits en el KV cache del provider.

**Problema:** La compresión cambia el texto del prompt, lo que invalida el KV cache que el proveedor mantiene entre turns (prompt caching).

**Solución:** CacheAligner asegura que los primeros N tokens del prompt comprimido sean **idénticos entre turns** cuando sea posible, permitiendo que el provider reuse su KV cache y ahorre tiempo y dinero adicionalmente.

**Métrica:** Cache hit rate se reporta via `proxy_cache_hit_rate_per_session{provider}`.

### 5.6 Image Compression

Router ML entrenado que decide entre:

- **Skip** (imagen muy importante, no comprimir)
- **Redimensión** (reducir resolución manteniendo features clave)
- **Descripción textual** (extraer descripción y eliminar la imagen base64)
- **Eliminación** (imagen decorativa/no relevante)

**Ahorro:** 40-90% en tokens de imagen.
**Precisión:** No afecta benchmarks visuales standard.

### Comparativa de Compresores

| Compresor | Tipo | Savings | Precision | Ideal para |
|---|---|---|---|---|
| SmartCrusher | JSON | 60-92% | 97% | Tool outputs, API responses |
| CodeCompressor | AST | 40-70% | 100% | Source code |
| Kompress-base | ML | 30-60% | 99%+ | Logs, text, docs |
| CacheAligner | Prefix | 0% directo | N/A | Ahorro indirecto via KV cache |
| ImageCompressor | ML | 40-90% | Alta | Screenshots, diagrams |
| Output Shaper | Salida | 20-65% | 95%+ | Answers del modelo |

---

## 6. Output Token Reduction

Además de comprimir *entrada*, Headroom comprime la **salida** del modelo — lo que el modelo escribe de vuelta.

### Funcionamiento

```bash
export HEADROOM_OUTPUT_SHAPER=1
headroom proxy --port 8787
```

**Tres mecanismos:**

1. **Verbosity Steering** (niveles L1-L5)
   - Apéndice "be terse" al system prompt (cache-safe, no invalida prompt cache)
   - Nivel configurable via `HEADROOM_VERBOSITY_LEVEL`
   - L2: -22.7% tokens (code review ask)
   - L3: -65.8% tokens (code review ask, **mismos bugs encontrados**)

2. **Effort Routing**
   - Reduce thinking effort en turns mecánicos (tool returns, continuaciones)
   - Mantiene full effort en preguntas nuevas, errores, análisis
   - Aplica a modelos con extended thinking (Claude Sonnet/Opus)

3. **Auto-Learning**
   - `headroom learn --verbosity` mina sesiones pasadas buscando interrupts/fast-skips
   - Aprende el nivel de verbosidad óptimo para cada tipo de tarea
   - `--apply` persiste la configuración

### Medición Honesta

Headroom reporta output savings como **estimación con intervalo de confianza** (nunca un número inventado):

```bash
headroom output-savings
# Reduction: 31.7%  (95% CI 27.7% … 35.7%)   [estimated]
```

Para una medición *real* (no estimada), se reserva un grupo de control:
```bash
export HEADROOM_OUTPUT_HOLDOUT=0.1    # 10% de conversaciones sin shaping
```

### Hot-Reload (Runtime Env)

Las variables de output shaping son **hot-reloadables**: `headroom wrap` envía la configuración al proxy via `POST /admin/runtime-env` **sin reiniciar** — no hay cold start, no se pierden peticiones ni caches.

---

## 7. CCR: Compress-Cache-Retrieve

El sistema de compresión reversible de Headroom.

### Cómo Funciona

1. **Compress:** El input se comprime y se envía al LLM
2. **Cache:** El original se almacena localmente con un hash (TTL configurable, default 300s)
3. **Retrieve:** El LLM puede llamar `headroom_retrieve(hash)` para recuperar el original completo

### Tools MCP

```json
{
  "headroom_compress": { "messages": [...] },   // Comprime y cachea
  "headroom_retrieve": { "hash": "abc123" },    // Recupera original
  "headroom_stats": {}                           // Estadísticas
}
```

### Configuración

```bash
export HEADROOM_CCR_TTL_SECONDS=600   # Default: 300
```

### Stats Clave

- **compressions** — count de invocaciones de compress
- **tokens_removed** — suma de tokens removidos
- **retrievals** — count de retrieve calls; si crece linealmente con turns, el compresor está siendo muy agresivo

---

## 8. Cross-Agent Memory

Headroom implementa un **store de memoria compartido entre agentes**.

### Características

- **Agentes soportados:** Claude Code, Codex, Gemini (cualquiera que hable el protocolo MCP)
- **Deduplicación automática:** El mismo contexto compartido entre agentes no se duplica
- **Procedencia:** Cada entrada sabe qué agente la creó
- **Recuperación automática:** Contexto relevante se inyecta automáticamente basado en similitud

### Use Case

```bash
headroom proxy --port 8787 --memory
# En otra terminal:
headroom wrap claude    # Claude usa el store compartido
headroom wrap codex     # Codex usa el MISMO store
```

Ambos agentes comparten memoria de contexto: lo que Claude aprendió está disponible para Codex.

---

## 9. Headroom Learn

Sistema de **aprendizaje de fallos** que mina el historial de sesiones.

### Funcionamiento

```bash
headroom learn              # Analiza sesiones fallidas
headroom learn --apply      # Escribe correcciones a CLAUDE.md / AGENTS.md
```

**Qué detecta:**
- Turns donde el usuario hizo interrupt (canceló la respuesta)
- Turns donde el usuario pidió "se más breve" o similar
- Patrones de fast-skip en outputs
- Fallos de compresión (retrievals excesivos)

**Qué genera:**
- Correcciones conductuales en `CLAUDE.md` / `AGENTS.md`
- Configuración de verbosidad óptima
- Reglas de compresión personalizadas

---

## 10. Métricas y Benchmarks

### Savings en Workloads Reales

| Workload | Before | After | Savings |
|---|---|---|---|
| Code search (100 results) | 17,765 | 1,408 | **92%** |
| SRE incident debugging | 65,694 | 5,118 | **92%** |
| GitHub issue triage | 54,174 | 14,761 | **73%** |
| Codebase exploration | 78,502 | 41,254 | **47%** |
| Bedrock tool outputs | 51,961 | 25,658 | **50.6%** |
| RAG chunks (12 items) | 4,424 | 2,756 | **38%** |

### Benchmarks de Precisión

| Benchmark | Categoría | Baseline | Headroom | Delta |
|---|---|---|---|---|
| GSM8K | Math | 0.870 | 0.870 | **±0.000** |
| TruthfulQA | Factual | 0.530 | 0.560 | **+0.030** |
| SQuAD v2 | QA | — | **97%** | @ 19% compresión |
| BFCL | Function Calling | — | **97%** | @ 32% compresión |

### Output Savings (code review ask)

| Nivel | Tokens | vs Base |
|---|---|---|
| Sin shaping | 1,750 | — |
| L2 | 1,354 | -22.7% |
| L3 | 599 | **-65.8%** |

> Mismos bugs encontrados en los 3 niveles.

### Métricas de Observabilidad

Headroom expone métricas Prometheus en `/metrics`:

- `proxy_compression_ratio_by_strategy{strategy, content_type}`
- `proxy_cache_hit_rate_per_session{provider}`
- `proxy_rate_limit_remaining_{requests,tokens}{provider}`
- `proxy_passthrough_bytes_modified_total`
- `proxy_compression_rejected_by_token_check_total`
- `service_tier` (vocabulario acotado: auto, default, flex, on_demand, priority, scale)

---

## 11. Integración con Agentes

### Compatibilidad

| Agente | `headroom wrap` | Notas |
|---|---|---|
| Claude Code | ✅ | `--memory`, `--code-graph` |
| Codex | ✅ | Comparte memoria con Claude |
| Cursor | ✅ | Imprime config — pegar una vez |
| Aider | ✅ | Inicia proxy + lanza agente |
| Copilot CLI | ✅ | Inicia proxy + lanza, soporta Subscription |
| OpenClaw | ✅ | ContextEngine plugin |
| Cualquier OpenAI-compatible | ✅ vía proxy | `headroom proxy` |

### GitHub Copilot Subscription Mode

```bash
headroom copilot-auth login
headroom wrap copilot --subscription -- --model gpt-4o
```

Soporta autenticación Business/Enterprise Cloud via token-exchange.

### Vertex AI Claude Code

```bash
# Con Vertex configurado
headroom wrap claude   # Detecta CLAUDE_CODE_USE_VERTEX=1
```

Headroom no maneja credenciales — el cliente usa su propio token GCP ADC.

### AWS Bedrock

```bash
headroom proxy --port 8787 --bedrock-api-url <gateway-url>
```

⚠️ **Importante:** Headroom rewritea el body, lo que invalida la firma SigV4. Debe apuntarse a un gateway que re-firme o no verifique (LiteLLM, LocalStack). No apuntar directamente a AWS.

---

## 12. Rust Migration & Performance

Headroom está migrando su core a Rust (proxy-first).

### Estado Actual

- **Proxy core** → Rust (`crates/headroom-core/`)
- **Auth mode detection** → Rust + Python port
- **PyO3 extension** para performance en Python
- **Binary shipping:** `headroom-proxy` Rust binary se build y distribuye en imágenes Docker

### Benchmarks (Auth Mode Detection on M-series)

| Escenario | Tiempo |
|---|---|
| Empty request | 68 ns |
| Payg Anthropic | 75 ns |
| OAuth JWT | 182 ns |
| Subscription Claude Code | 81 ns |

### Docker Images

```bash
docker pull ghcr.io/chopratejas/headroom:latest
```

Tags disponibles: `debian` (full) y `distroless-slim` (minimal).

---

## 13. Relevancia para Hermes / Nuestro Stack

### ¿Dónde encaja Headroom?

Hermes Agent (nuestro stack actual) usa proveedores LLM via OpenCode Go, OpenRouter, etc. El problema que tenemos es:

> **Sesiones que se acortan por consumo excesivo de tokens** → Headroom puede comprimir tool outputs, logs y contexto antes de enviarlos al LLM.

### Puntos de Integración Posibles

**Opción A: Proxy Mode (recomendada)**
```
Hermes → Headroom Proxy (localhost:8787) → OpenCode Go / OpenRouter
```
- Zero cambios en código de Hermes
- Solo apuntar el provider a `http://localhost:8787/v1`
- Headroom se encarga de enrutar al provider real

**Opción B: MCP Server** (si Hermes implementa MCP)
- Tools `headroom_compress`, `headroom_retrieve`, `headroom_stats`
- Compresión on-demand en skills específicas

**Opción C: Library + inline**
- `from headroom import compress` en skills Python de Hermes
- Control granular sobre qué se comprime y cuándo

### Ahorro Esperado

Basado en los benchmarks, para nuestro stack:

| Componente | Sin compresión | Con Headroom | Saving estimado |
|---|---|---|---|
| Tool outputs (JSON) | 17,000 | 1,400 | **~92%** |
| Logs de debugging | 65,000 | 5,100 | **~92%** |
| Code search results | 17,000 | 1,400 | **~92%** |
| RAG context | 4,400 | 2,700 | **~38%** |

### Consideraciones

1. **Latencia:** Headroom añade latencia de procesamiento (ms) pero reduce latencia de red (menos tokens = menos tiempo de generación)
2. **Complejidad operativa:** Un proxy más en la cadena = un punto más de fallo
3. **CCR TTL:** Para sesiones largas, aumentar `HEADROOM_CCR_TTL_SECONDS` (default 300s es poco)
4. **Output Shaping:** Ideal para tareas mecánicas (tool returns), habilitar con `HEADROOM_OUTPUT_SHAPER=1`
5. **CacheAligner:** Crítico para mantener KV cache hits con Anthropic/OpenAI

---

## 14. Análisis Costo-Beneficio

### Beneficios Cuantitativos

| Métrica | Sin Headroom | Con Headroom | Mejora |
|---|---|---|---|
| Tokens por sesión (code search) | 17,765 | 1,408 | 12.6x más sesiones |
| Tokens por sesión (debugging) | 65,694 | 5,118 | 12.8x más sesiones |
| Costo por sesión | ~$0.20 | ~$0.015 | **93% menos** |
| Sesiones por $10 | ~50 | ~667 | **13x más** |

### Costos

- **Instalación:** `pip install headroom-ai[all]` (dependencias: torch si se usa ML)
- **CPU/RAM:** Proxy liviano (~50-100MB RAM, uso CPU moderado)
- **Latencia:** ~50-200ms adicionales por request (compresión)
- **Mantenimiento:** Actualizaciones periódicas (`headroom update`)

### Tradeoffs

| Aspecto | Ventaja | Desventaja |
|---|---|---|
| Tokens | Reducción 60-95% | Posible pérdida de matices en compresión agresiva |
| Sesiones | 10-12x más por mismo presupuesto | Mayor complejidad de infraestructura |
| Velocidad | Menos tokens = respuesta más rápida | Overhead de compresión (~ms) |
| Precisión | Benchmarks muestran 0% pérdida | Retrievals aumentan si compresión muy agresiva |

---

## 15. Recomendaciones Prácticas

### Para Implementar en Hermes

1. **Comenzar con Proxy Mode:**
   ```bash
   # Instalar
   pip install "headroom-ai[proxy]"
   
   # Iniciar proxy (hot-reload para ajustes)
   headroom proxy --port 8787
   
   # Configurar Hermes para usar el proxy
   # En hermes config: apuntar a http://localhost:8787/v1
   ```

2. **Habilitar Output Shaping gradualmente:**
   ```bash
   export HEADROOM_OUTPUT_SHAPER=1
   export HEADROOM_VERBOSITY_LEVEL=2    # Empezar conservador
   ```

3. **Ajustar CCR TTL para sesiones largas:**
   ```bash
   export HEADROOM_CCR_TTL_SECONDS=3600  # 1 hora
   ```

4. **Monitorear retrievals:**
   ```bash
   headroom output-savings  # Ver estimación de ahorro
   # Si retrievals crecen mucho, bajar nivel de compresión
   ```

5. **Auto-learn del comportamiento del usuario:**
   ```bash
   headroom learn --verbosity
   headroom learn --verbosity --apply
   ```

### Configuración Recomendada (Baseline)

```bash
# Proxy
headroom proxy --port 8787 \
  --provider openrouter \
  --ccr-ttl 3600 \
  --output-shaper 1 \
  --verbosity-level 2

# Env vars
export HEADROOM_OUTPUT_SHAPER=1
export HEADROOM_VERBOSITY_LEVEL=2
export HEADROOM_CCR_TTL_SECONDS=3600
export HEADROOM_OUTPUT_HOLDOUT=0.1  # grupo de control para medir
```

### Lo que NO hacer

- No apuntar Headroom directamente a AWS Bedrock sin gateway intermedio (invalida SigV4)
- No usar compresión máxima sin monitorear retrievals (pérdida de información)
- No olvidar que Output Savings son **estimados**, no medidos (usar `HEADROOM_OUTPUT_HOLDOUT`)
- No combinar con otros proxies sin probar compatibilidad

---

## Recursos

- **GitHub:** https://github.com/chopratejas/headroom
- **Docs:** https://headroom.docs.pyke.io (Fumadocs, MDX)
- **PyPI:** https://pypi.org/project/headroom-ai/
- **Docker:** `ghcr.io/chopratejas/headroom:latest`
- **Video:** https://www.youtube.com/watch?v=-WruUQhdN7c
- **Benchmarks:** `python -m headroom.evals suite --tier 1`
- **License:** Apache 2.0

---

> **Análisis preparado para Jexus Brain — Junio 2026**
> **Propósito:** Evaluar Headroom como capa de compresión para Hermes Agent, con objetivo de extender la duración de sesiones reduciendo tokens 60-95%.
