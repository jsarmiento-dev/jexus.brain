---
name: cursor-code
description: Orquestar Cursor Cloud Agents desde Hermes. Hermes planifica y escribe el prompt, Cursor ejecuta el código pesado (multi-file refactors, PRs automáticos). Ahorra tokens usando Cursor como ejecutor.
---

# cursor-code — Hermes orquesta, Cursor ejecuta

## ⚠️ Preferencia del usuario: delegar TODO a Cursor

**Jesús (jsarmiento) prefiere que TODOS los bugs, fixes y cambios de código se deleguen a Cursor CLI headless.** Incluso cambios de un solo archivo, tooltips, formato de hora, o errores específicos — no usar `patch` directo ni editar manualmente. El flujo correcto es:

1. Identificar el problema (con curl, logs, o API check)
2. Escribir prompt file en `openspec/steps/hotfix-*.md`
3. Delegar a Cursor: `agent -p --force --model auto "$(cat openspec/steps/hotfix-*.md)"`
4. Esperar `notify_on_complete`
5. Verificar resultado

**Excepción:** Solo para cambios que NO son código (configs, env vars, docker, nginx) se puede hacer directo.

## Cuándo usar
- Tareas de código complejas (multi-file refactors, features completas que tocan modelo → schema → ruta → UI → tests)
- Cuando Hermes necesitaría demasiados tokens para codear directamente
- PRs automáticos con tests, docs, y revisiones
- Tareas que se benefician del Composer de Cursor y su conocimiento profundo del codebase

## Cuándo NO usar (hacer directo)
**Analiza primero, delega si amerita** — antes de delegar a Cursor, evalúa:
- ¿Es un cambio en un solo archivo (ej. CSS/JS en `index.html`)? → **hazlo directo**, es más rápido y consume menos tokens
- ¿Es un cambio de frontend puro (columna info tooltip, formato de hora, badge visual)? → **hazlo directo**, Cursor tarda más en arrancar que en implementarlo
- ¿Es corregir un error específico (pitfall de migración, tipo SQLite vs PostgreSQL)? → **hazlo directo** con `patch`

Usa Cursor solo para cambios multi-archivo que requieran coordinación entre capas (modelo + schema + ruta + UI + tests), o cuando el prompt requiera >10 instrucciones detalladas.

## Prerequisitos
- Cursor CLI instalado en la VPS: `~/.local/bin/agent` (v2026.06.04)
- Cursor API Key: obtenerla de https://cursor.com/settings → Cloud Agents → API Keys
- La API key DEBE estar en `~/.hermes/.env` como `CURSOR_API_KEY`
- Detalles de troubleshooting y modelos en archivos de referencia:
  - `references/troubleshooting.md` — errores comunes y fixes
  - `references/available-models.md` — lista real de modelos disponibles
  - `scripts/cursor-agent.sh` — script helper para crear agentes vía API

## Arquitectura del flujo

```
Usuario (Telegram) → Hermes (planifica, crea prompt)
                          ↓
              POST https://api.cursor.com/v1/agents
                          ↓
              Cursor Cloud Agent (ejecuta código)
                          ↓
              GitHub PR automático (autoCreatePR: true)
                          ↓
              Hermes monitorea status → reporta por Telegram
```

## API Reference

**Auth:** Basic auth con API key como username, password vacío.
Header: `Authorization: Basic base64(CURSOR_API_KEY:)`
O usar `-u $CURSOR_API_KEY:` en curl.

### Crear agente (inicia ejecución inmediata)

```bash
curl -s --request POST \
  --url https://api.cursor.com/v1/agents \
  -u "$CURSOR_API_KEY:" \
  --header 'Content-Type: application/json' \
  --data '{
    "prompt": {"text": "TU_PROMPT_AQUI"},
    "repos": [{"url": "https://github.com/USER/REPO", "startingRef": "main"}],
    "model": {"id": "composer-2.5"},
    "autoCreatePR": true,
    "mode": "agent"
  }'
```

Respuesta:
```json
{
  "agent": {
    "id": "bc-xxx",
    "name": "...",
    "url": "https://cursor.com/agents/bc-xxx",
    "latestRunId": "run-xxx"
  },
  "run": {"id": "run-xxx", "status": "CREATING"}
}
```

### Verificar status de un agente

```bash
curl -s --request GET \
  --url "https://api.cursor.com/v1/agents/$AGENT_ID" \
  -u "$CURSOR_API_KEY:"
```

### Verificar status de un run

```bash
curl -s --request GET \
  --url "https://api.cursor.com/v1/agents/$AGENT_ID/runs/$RUN_ID" \
  -u "$CURSOR_API_KEY:"
```

### Crear un follow-up run (si el agente sigue activo)

```bash
curl -s --request POST \
  --url "https://api.cursor.com/v1/agents/$AGENT_ID/runs" \
  -u "$CURSOR_API_KEY:" \
  --header 'Content-Type: application/json' \
  --data '{"prompt": {"text": "Arregla el bug en el archivo auth.ts"}}'
```

### Listar agentes recientes

```bash
curl -s --request GET \
  --url "https://api.cursor.com/v1/agents?limit=5" \
  -u "$CURSOR_API_KEY:"
```

## Modelos disponibles

Lista completa y actualizada en `references/available-models.md`. Para refrescar en vivo:

```bash
# CLI
agent models

# API (NOTA: respuesta usa .items, NO .models)
curl -s -u "$CURSOR_API_KEY:" https://api.cursor.com/v1/models | jq '.items[:5]'
```

### Favoritos (verificados Jun 2026, Pro tier)

| Modelo | Provider | Mejor para |
|--------|----------|------------|
| `auto` | Auto-select | **Créditos agotados** — siempre funciona |
| `composer-2.5` | Cursor | Multi-file refactors (default Pro) |
| `claude-sonnet-4-6` | Anthropic | Balance calidad/velocidad |
| `claude-opus-4-8` | Anthropic | Máxima calidad, tareas complejas |
| `gpt-5.5` | OpenAI | General-purpose 1M context |
| `gpt-5.3-codex` | OpenAI | Optimizado para código |
| `default` | Auto | Cloud Agents API — auto-selección |

## Parámetros clave

| Parámetro | Descripción |
|-----------|-------------|
| `prompt.text` | **(requerido)** Instrucciones en lenguaje natural |
| `repos[].url` | URL del repo GitHub (https://github.com/...) |
| `repos[].startingRef` | Branch base (default: main) |
| `repos[].prUrl` | Si hay PR existente, trabajar sobre él |
| `autoCreatePR` | Crear PR automáticamente al terminar |
| `workOnCurrentBranch` | `true` para pushear directo (default: crea rama `cursor/...`) |
| `mode` | `"agent"` para implementar, `"plan"` para solo planificar |
| `model.id` | Modelo a usar |
| `mcpServers` | Servidores MCP inline (hasta 50) |
| `env` | `"cloud"`, `"pool"`, o `"machine"` |

## Pattern: Cómo armar un buen prompt para Cursor

Hermes DEBE crear prompts detallados con:
1. **Contexto**: qué hace el proyecto, arquitectura
2. **Tarea específica**: qué archivos tocar, qué cambios hacer
3. **Constraints**: estilo, tests, no romper nada
4. **Output esperado**: PR con descripción, tests pasando

Ejemplo de prompt bien armado:
```
Eres un desarrollador senior trabajando en jexus.pocket (Next.js 16 + React 19 + Supabase).
El proyecto está en ~/proyects/jexus.pocket.

TAREA: Agregar tests unitarios para el módulo de autenticación.

ARCHIVOS A MODIFICAR:
- src/lib/auth.ts → tests en __tests__/auth.test.ts
- src/middleware.ts → tests en __tests__/middleware.test.ts

CONSTRAINTS:
- Usar vitest (ya configurado en el proyecto)
- No modificar lógica existente, solo agregar tests
- Cobertura mínima 80%
- Los tests deben pasar con `npm test`

OUTPUT:
- PR con título "test: agregar tests unitarios para auth"
- Descripción detallada en el PR
```

## Monitoreo (polling)

El run tarda de 30s a varios minutos. Se debe hacer polling cada 15-30s:

```bash
# Extraer agent_id y run_id de la respuesta de creación
AGENT_ID=$(echo "$RESPONSE" | jq -r '.agent.id')
RUN_ID=$(echo "$RESPONSE" | jq -r '.run.id')

# Polling loop
while true; do
  STATUS=$(curl -s --request GET \
    --url "https://api.cursor.com/v1/agents/$AGENT_ID/runs/$RUN_ID" \
    -u "$CURSOR_API_KEY:" | jq -r '.status')
  
  case "$STATUS" in
    "FINISHED") echo "✅ Completado"; break ;;
    "ERROR") echo "❌ Error"; break ;;
    *) echo "⏳ $STATUS..."; sleep 15 ;;
  esac
done
```

## Diagnóstico rápido

Antes de ejecutar, verificar estado de la cuenta:

```bash
export $(grep CURSOR_API_KEY ~/.hermes/.env | xargs)
agent about           # Muestra tier, modelo default, email
agent models          # Lista modelos disponibles para esta cuenta
```

## Pitfalls

- **API Key**: debe estar en `CURSOR_API_KEY` en `~/.hermes/.env` (NO en `~/.hermes/hermes-agent/.env`). Verificar con `export $(grep CURSOR_API_KEY ~/.hermes/.env | xargs) && echo ${#CURSOR_API_KEY}` (debe ser ~69-71 chars). La key se obtiene de https://cursor.com/settings → Cloud Agents → API Keys.
- **Credential masking**: al escribir la key en `.env` desde terminal, la herramienta puede enmascarar/truncar el valor. Solución comprobada: split en variables (`K1="crsr_...primera_mitad" K2="...segunda_mitad" echo "CURSOR_API_KEY=${K1}${K2}" >> ~/.hermes/.env`). Verificar longitud con `wc -c` después.
- **Cloud Agents API requiere billing**: `POST /v1/agents` devuelve `usage_limit_exceeded: Background Agent requires at least $2 remaining until your hard limit`. Solución: habilitar usage-based pricing en https://cursor.com/dashboard?tab=settings o usar CLI headless (ver abajo).
- **CLI headless sin créditos**: si el tier Pro está agotado, el CLI headless falla con "You're out of usage". Solución: usar `--model auto` que selecciona el modelo más barato disponible y usualmente funciona incluso sin créditos premium.
- **Beta**: la API es v1 pública beta. Puede cambiar.
- **Rate limits**: ver https://cursor.com/docs/api para límites
- **409 agent_busy**: solo un run activo por agente. Esperar o crear nuevo agente.
- **Timeout**: runs largos (>10min) pueden fallar. Dividir en tareas más pequeñas.
- **GitHub token**: Cursor usa su propio token para crear PRs. Asegurarse que el repo sea accesible.
- **aarch64**: El CLI funciona en ARM64 (Oracle Cloud). Instalación: `curl -fsS https://cursor.com/install | bash`.
- **Headless**: `agent -p --force "prompt"` modifica archivos locales SIN PR. Para PRs usar la API. Siempre usar `--model auto` como primera opción para evitar problemas de créditos.
- **⚠️ Shell escaping en terminal()**: al embeder comandos con subshells `$(...)` dentro del string de `terminal()`, el escaping de la herramienta puede corromper la sintaxis. **Solución**: (a) no usar subshells inline — el CLI lee `.env` automáticamente, o (b) extraer el valor con `grep` en paso separado, o (c) comillas simples exteriores. El patrón `export $(grep CURSOR_API_KEY ...)` está particularmente propenso a romperse.
⚠️ **NUNCA ejecutar dos instancias CLI headless en paralelo sobre el mismo proyecto.** Ambas modifican archivos directamente en el mismo branch → conflicto de merge garantizado, archivos corruptos, trabajo perdido. Si el usuario pide múltiples tareas paralelas, explicar que se necesita Cloud Agents API (modo 1) o ejecutarlas secuencialmente.

⚠️ **⚠️ git add -A absorbe cambios del proceso paralelo**: si a pesar de la regla anterior dos instancias headless terminan ejecutándose sobre el mismo repo (ej: pipeline bg + estrategia individual), el primer `git add -A && git commit` absorberá los archivos creados por AMBAS instancias. La segunda instancia no tendrá nada nuevo que commitear pero los cambios ya están commiteados — no hay pérdida de trabajo pero el commit queda inflado con cambios de ambas tareas. Síntoma: un commit de "estrategia X" incluye archivos de "feature Y". Monitorear con `git diff --stat` entre commmits para detectar absorciones.
- **Headless modifica MUCHOS archivos**: Cursor en modo headless con prompts amplios puede tocar 75+ archivos. Verificar el alcance con `git diff --stat` antes de commitear. Si tocó archivos fuera del scope, hacer `git checkout -- <archivo>` para revertir los no deseados.

- **⚠️ Cursor headless usa el Python del sistema, no el venv del proyecto**: Cuando Cursor instala nuevas dependencias (ej. vía `pip install -r requirements.txt`), lo hace en el Python del sistema, NO en el `.venv/` del proyecto. Si después verificas tests con `python -m pytest`, corres contra el sistema donde las dependencias nuevas existen. Pero si verificas con `.venv/bin/python -m pytest`, falla porque el venv no las tiene. **Patrón correcto**: después de que Cursor modifica `requirements.txt`, instalar en el venv del proyecto primero, luego verificar con el Python del venv. Ejemplo:
  ```bash
  cd ~/proyects/<proyecto>
  git diff --stat                          # ver qué tocó Cursor
  .venv/bin/pip install -r requirements.txt  # instalar nuevas deps en el venv
  .venv/bin/python -m pytest tests/ -q       # verificar con el venv, NO con python global
  ```
  No asumir que `python -m pytest` funciona — el PATH de `terminal()` puede apuntar al Python Hermes o al del sistema, no al venv del proyecto.

- **⚠️ Cursor genera defaults viejos/erróneos en scripts y configs**: Cuando Cursor crea un script, archivo de configuración o bloque de parámetros, usa internamente valores de su entrenamiento (que pueden estar desactualizados respecto al código actual del proyecto). **Caso real**: Al crear `scripts/backtest_caja_sp100.py`, Cursor puso `rci_divergence: True` y `caja_candles: 6` cuando el código ya tenía `rci_divergence: False` y `caja_candles: 3`. El backtest corrió con parámetros viejos dando resultados engañosos hasta que se detectó el mismatch. **Patrón correcto**: después de que Cursor crea o modifica scripts que contienen default values o constantes, verificar que coincidan con el código fuente del proyecto actual:

- **⚠️ Cursor genera defaults viejos/erróneos en scripts y configs**: Cuando Cursor crea un script, archivo de configuración o bloque de parámetros, usa internamente valores de su entrenamiento (que pueden estar desactualizados respecto al código actual del proyecto). **Caso real**: Al crear `scripts/backtest_caja_sp100.py`, Cursor puso `rci_divergence: True` y `caja_candles: 6` cuando el código ya tenía `rci_divergence: False` y `caja_candles: 3`. El backtest corrió con parámetros viejos dando resultados engañosos hasta que se detectó el mismatch. **Patrón correcto**: después de que Cursor crea o modifica scripts que contienen default values o constantes, verificar que coincidan con el código fuente del proyecto actual:
  ```bash
  # 1. Identificar constantes/defaults que Cursor puso
  git diff --stat <archivo>
  
  # 2. Comparar contra el código real (ej. parámetros de la estrategia)
  grep -n "rci_divergence\|caja_candles\|default" infrastructure/strategies/caja_apertura.py | head -10
  
  # 3. Si Cursor usó valores viejos, corregirlos antes de ejecutar
  # El backtest dará resultados erróneos y perderás tiempo debugueando
  ```

- **⚠️ Cursor genera sintaxis SQLite en migraciones para PostgreSQL**: Cuando Cursor agrega columnas a modelos SQLAlchemy y genera sentencias `ALTER TABLE` en `_migrate_db()`, usa tipos SQLite por defecto (`DATETIME`, `BOOLEAN`) en lugar de los de PostgreSQL (`TIMESTAMP`). `BOOLEAN` pasa igual, pero `DATETIME` no existe en PostgreSQL — solo `TIMESTAMP` (con o sin zona horaria). **Caso real**: Cursor generó `ALTER TABLE copied_trades ADD COLUMN closed_at DATETIME` que falló con `UndefinedObject: type "datetime" does not exist` al correr contra PostgreSQL. **Patrón correcto**: después de que Cursor añade migraciones con tipos de columna, verificar que no haya tipos SQLite:
  ```bash
  grep -n 'DATETIME' src/models/*.py
  ```
  Ver también `references/cross-cutting-enhancement-prompt.md` § "Migration types for target DB" para el checklist completo post-Cursor.

- **⚠️ Docker network reconnect en container recreate**: Cuando haces `docker compose up -d --no-deps <service>` (recrear un solo servicio), el container nuevo obtiene una nueva IP en la red interna de compose. Si hay un **proxy reverso externo** (ej. Nginx en otra red Docker) que apunta a este container, el container **pierde la conexión a esa red externa** al recrearse. Síntoma: `502 Bad Gateway` después del deploy — el contenedor corre pero el proxy no llega. **Patrón correcto** después de `up -d --no-deps`: reconectar a la red externa:
  ```bash
  sudo docker network connect <external-network> <container-name>
  ```
  Verificar con `curl -s -o /dev/null -w "%{http_code}" https://dominio.com/ruta`. Caso real: `jxus-copy-trading-bot` API container pierde `proyects_proyects-net` cada recreate → Nginx 502.

- **⚠️ Column info tooltips pattern**: Para tooltips en headers de tabla (dashboard single-page):
  ```html
  <th>Column <span class="col-info" title="Tooltip text">ⓘ</span></th>
  ```
  ```css
  .col-info { cursor: help; opacity: 0.5; font-size: 0.6875rem; transition: opacity 0.15s; }
  th:hover .col-info { opacity: 1; }
  ```
  El `title` attribute da tooltip nativo del browser (mobile: tap hold). No delegar a Cursor — es más rápido hacerlo directo.

- **⚠️ Frontend computed columns**: Para columnas computadas (ej. trader original size = our_size / multiplier), calcular en el frontend con datos existentes del API response. No tocar BD ni backend. Hacer directo con `patch` en `index.html`.

⚠️ **⚠️ Cursor re-aplica cambios previos**

⚠️ **⚠️ Flag `-m` no existe en Cursor CLI**: El CLI headless no acepta un flag `-m` para el mensaje/prompt. El prompt se pasa como **argumento posicional** al final:
```bash
# ✅ Correcto
agent -p --force --model auto "descripción de la tarea"
# ❌ Incorrecto — unknown option '-m'
agent -p --force --model auto -m "task"
```
Ver `references/sdd-delegate-pattern.md` § "Invocación correcta de Cursor CLI" para ejemplos detallados y pattern de background execution.

- **git add -A absorcion**: si dos instancias headless terminan sobre el mismo repo, el primer `git add -A && git commit` absorbe cambios de ambas. Monitorear con `git diff --stat` entre commits para detectar absorciones.

---

## Estrategia de ejecución (orden de preferencia)

Cuando el usuario pide **code review + cleanup**, usar este patrón en 4 fases:

1. **Analizar con subagentes paralelos** (`delegate_task` — un subagente por capa/dominio/aplicación/infraestructura)
2. **Consolidar hallazgos** en plan priorizado (CRITICAL → IMPORTANT → SUGGESTION) y presentar al usuario
3. **Crear prompt files** en `scripts/pipeline/steps/` con estructura de refactoring (no features)
4. **Ejecutar secuencialmente** via Cursor headless, verificando tests entre fases

### Diferencias clave vs. Pipeline de features

| Aspecto | Pipeline features | Pipeline refactoring |
|---------|------------------|---------------------|
| Origen | Plan de implementación | Análisis de subagentes |
| Prompt | "Crear X archivos con Y funcionalidad" | "Refactorizar archivo A: eliminar líneas X-Y, mover lógica a B" |
| Métrica de éxito | Feature funciona | Líneas reducidas + Tests pasan |
| Alcance | Archivos nuevos + modificaciones | Archivos existentes (modificar/eliminar) |
| Scope creep | Crea archivos extra | **Re-aplica cambios previos** fuera del prompt |
| Orden | Puede ser autónomo (orchestrate.sh) | Debe ser supervisado (Hermes verifica entre fases) |

### Estructura del prompt de refactoring

Cada prompt en `steps/` debe incluir:

1. **Problema**: qué código está duplicado/mal y dónde (archivo:línea exacta)
2. **Solución**: qué archivos crear, qué archivos modificar, qué líneas eliminar
3. **Before/After**: métricas de referencia (ej. "HealthCheckUseCase 227 → 65 líneas")
4. **Constraints**: mantener interfaz pública, no romper tests
5. **Verificación**: comando exacto para correr tests
6. **Criterio de éxito**: ej. "ningún _max_drawdown duplicado", "146 tests pasan"

Ejemplo real usado en sesión:

```markdown
# Task: Extraer _max_drawdown a servicio compartido

## Problema: Código duplicado
Misma función en 2 archivos:
1. `application/backtesting/backtest_engine.py:551-577`
2. `application/services/ab_test_runner.py:296-322`

## Tarea
1. CREAR `domain/services/metrics_service.py` con compute_max_drawdown() y compute_sharpe_ratio()
2. En backtest_engine.py: eliminar _max_drawdown, importar desde nuevo módulo
3. En ab_test_runner.py: mismo patrón

## Constraints: mantener lógica exacta, tests pasan sin cambios
## Verificación: pytest tests/ -v --tb=line -k "not test_ema12_pivotes_btc_30d"
```

### Flujo de ejecución

```python
write_file("scripts/pipeline/steps/refactor-t1-health.md", "...")
write_file("scripts/pipeline/steps/refactor-t2-metrics.md", "...")

for step in ["refactor-t1-health", "refactor-t2-metrics", ...]:
    terminal(
        f"cd ~/proyects/<proy> && agent -p --force --model auto \"$(cat scripts/pipeline/steps/{step}.md)\"",
        background=True, notify_on_complete=True, timeout=600,
    )
    # Wait → git diff --stat → pytest → next
```

### Verificación entre fases

```bash
# 1. Alcance de cambios
git diff --stat
# 2. Scope creep: docker-compose.yml, tests/
git diff docker-compose.yml tests/
# 3. Tests completos
pytest tests/ -v --no-header --tb=line -k "not <test_lento>"
# 4. Avanzar solo si pasan
```

### Pitfalls de refactoring

- **Archivos fuera de scope**: docker-compose.yml, tests previos, archivos modificados manualmente. Si los cambios fuera de scope re-aplican fixes previos (mismo diff), ignorar. Si sobreescriben lógica nueva, `git checkout -- <file>`.
- **git add -A absorcion**: si dos instancias headless terminan sobre el mismo repo, el primer `git add -A && git commit` absorbe cambios de ambas. Monitorear con `git diff --stat` entre commits para detectar absorciones.

---

## Estrategia de ejecución (orden de preferencia)

### 1. Cloud Agents API (ideal, pero requiere billing)
Si la cuenta tiene usage-based pricing activo y ≥$2 de margen, usar `POST /v1/agents` con `autoCreatePR: true`. Es la opción más limpia: PR automático, monitoreo async, no bloquea. **Único modo que soporta ejecución paralela segura** (cada agente en workspace aislado, PR independiente).

### 2. CLI headless con --model auto (escape hatch sin créditos)
Cuando los créditos están agotados o no hay billing configurado, usar el CLI headless con `--model auto`. Este modo selecciona automáticamente el modelo más barato y **normalmente funciona incluso sin créditos premium**. Modifica archivos directamente en el branch actual (sin PR automático).

```bash
cd ~/proyects/<proyecto>
agent -p --force --model auto "Prompt detallado aquí..." 2>&1
```

No es necesario exportar `CURSOR_API_KEY` explícitamente — el CLI busca automáticamente en `~/.hermes/.env` y `~/.cursor-agent/.env`. El patrón `export $(grep CURSOR_API_KEY ~/.hermes/.env | xargs)` funciona en bash directo pero **se corrompe cuando se embebe dentro de terminal()** debido a la serialización de subshells. En terminal(), simplemente omitir el export.

⚠️ **NUNCA ejecutar dos instancias CLI headless en paralelo sobre el mismo proyecto.** Ambas modifican archivos directamente en el mismo branch → conflicto garantizado. Para paralelismo real, usar Cloud Agents API (opción 1). Como alternativa: encolar tareas secuencialmente, esperando que una termine antes de lanzar la siguiente.

⚠️ Luego hay que hacer commit y push manualmente. El prompt DEBE ser auto-contenido (HERMES arma el prompt completo, no espera interacción).

### 3. Ejecución en background (CLI headless)
El CLI headless puede tardar 3-10 minutos. Para no bloquear la sesión, ejecutarlo en background con `notify_on_complete=true`:

```python
terminal(
  command="cd ~/proyects/<proyecto> && agent -p --force --model auto 'Prompt detallado...'",
  background=True,
  notify_on_complete=True,
  timeout=600
)
```

⚠️ **No embeber `export $(grep ...)` inline** — la serialización del subshell `$(...)` se corrompe dentro del string de `terminal()`. El CLI headless lee `~/.hermes/.env` automáticamente, no hace falta export explícito.

⚠️ **Usar comillas simples para el prompt** dentro del command string si contiene caracteres especiales, para evitar escapes problemáticos con comillas dobles anidadas.

**Monitoreo de progreso mientras corre en background:** el CLI headless no produce output hasta que termina. Dos formas de verificar avance:

```bash
# Opción A: git status --short (más informativa — muestra archivos nuevos vs modificados)
cd ~/proyects/<proyecto> && git status --short | head -20

# Opción B: git diff --stat (solo archivos modificados, no los nuevos sin trackear)
cd ~/proyects/<proyecto> && git diff --stat | wc -l
```

Si el número de archivos modificados/nuevos crece entre chequeos, Cursor está trabajando. La notificación `notify_on_complete` avisará cuando termine.

**Post-ejecución:** revisar `git diff --stat` para ver el alcance total antes de commitear. Si Cursor tocó archivos fuera del scope, revertir con `git checkout -- <archivo>`.

```bash
# Ver resumen de cambios antes de commit
git diff --stat
# Ver archivos nuevos sin trackear
git status --short | grep '^??'
```

## Verificación final

Después de cada task, verificar:
- PR creado (si `autoCreatePR: true`)
- Tests pasan
- No hay regresiones
- Reportar a Telegram con link del PR

**Referencia de debug de backtests:** `references/strategy-backtest-analysis.md` — heurísticas para cuando los resultados del backtest no coinciden con lo esperado (trade count mismatch, sesiones vs ventanas rodantes, analisis de exit reasons dominantes).

**SDD → Delegate workflow:** `references/sdd-delegate-pattern.md` — flujo probado para features multi-archivo: Hermes crea SDD → Cursor implementa → Hermes deploya.

**Vite + React SPA deployment:** `references/vite-react-dashboard-case-study.md` — caso real de migración HTML→Vite+React con shadcn+TanStack+WS detrás de Nginx subpath.

**Polymarket wallet resolver:** `references/polymarket-wallet-resolver.md` — patrón para resolver handles/URLs de Polymarket a wallet addresses mediante scraping de `__NEXT_DATA__`.

**Polymarket fill aggregation:** `references/polymarket-fill-aggregation.md` — patrón para agregar fills individuales de Polymarket `/activity` en trades lógicos antes de risk checks y DB. Cubre la limitación de que Polymarket solo expone fills on-chain, no órdenes límite de otros traders. Incluye implementación de referencia con window de 60s y manejo correcto de dedup.

**Cross-cutting enhancement prompt:** `references/cross-cutting-enhancement-prompt.md` — patrón para prompts que tocan TODAS las capas (modelo → schema → ruta → UI → tests) en un solo paso atómico. Incluye checklist de verificación post-Cursor (tipos PostgreSQL, venv, redes Docker).

## Pipeline supervisado por fases (SDD → steps → Cursor)

Cuando el usuario aprueba un plan plurifase pero prefiere supervisión entre fases (verificar antes de continuar), usar este patrón **Hermes-supervisado**. A diferencia del orquestador autónomo, Hermes decide cuándo pasar a la siguiente fase y verifica el output de Cursor antes de avanzar.

### Cuándo usar este patrón vs. el orquestador autónomo

| Aspecto | Supervisado (este) | Autónomo (orchestrate.sh) |
|---------|-------------------|--------------------------|
| Control | Hermes verifica entre fases | Todo pre-aprobado, sin pausas |
| Riesgo | Bajo — se detectan errores temprano | Medio — errores se acumulan |
| Tokens | Más (Hermes verifica cada fase) | Menos (Hermes solo crea prompts) |
| Usuario | Puede cambiar de opinión entre fases | Todo decidido al inicio |
| Scope creep | Fácil de contener fase por fase | Difícil — Cursor tiende a expandir |

### Flujo

```
1. Hermes investiga (subagente de análisis)
2. Hermes escribe SDD (openspec/sdd-*.md)
3. Hermes crea prompts por fase (openspec/steps/f0X-*.md)
4. Para cada fase:
   a. Delegar a Cursor: agent -p --force --model auto "$(cat openspec/steps/f0X-*.md)"
   b. Esperar notify_on_complete
   c. Verificar: git log, git tag, build output, curl endpoint
   d. Reportar al usuario → preguntar si continuar
5. Repetir hasta completar
```

### Estructura de archivos

```
openspec/
├── sdd-feature-name.md         ← SDD del feature (requisitos, arquitectura, fases)
├── analysis/                   ← Reportes de subagentes de análisis
│   └── frontend-migration-analysis.md
└── steps/                      ← Prompts autocontenidos para Cursor
    ├── f01-scaffold-layout.md
    ├── f02-data-layer.md
    ├── f03-functional-components.md
    └── f04-polish-deploy.md
```

### Patrón de prompt file

Cada `openspec/steps/f0X-*.md` debe ser auto-contenido:
1. **Contexto del proyecto** (stack, estructura actual)
2. **Tarea específica de esta fase** (no asumir conocimiento de fases anteriores)
3. **Código completo** de cada archivo a crear/modificar (copy-pasteable)
4. **Instrucciones exactas**: template strings, imports, tipos
5. **Comandos de verificación**: build, test
6. **Comando de commit** al final

### Ejecución de cada fase

```bash
cd /ruta/del/proyecto
agent -p --force --model auto "$(cat openspec/steps/f0X-nombre.md)" 2>&1
```

Siempre en background con `notify_on_complete=true` y timeout generoso (600-900s para fases grandes).

### Verificación post-fase

```bash
# Commit y tag creados?
git log --oneline -3
git tag -l "pipeline/*"

# Build funciona?
npm run build  # o el comando del proyecto

# En producción (si aplica)?
curl -s -o /dev/null -w "%{http_code}" https://dominio/ruta

# Archivos creados correctamente?
find src -name "*.tsx" -type f | sort

# ✅ Verificar rutas de assets en el HTML build (subpath deployment)
grep -oP 'src="[^"]+"' src/api/static/index.html
# Las rutas deben ser /subpath/assets/... no /assets/...
```

### Tagging

```bash
git tag -f "pipeline/f0X-phase-name"
```

Esto permite retomar desde cualquier fase si el pipeline se interrumpe.

---

## Pipeline secuencial autónomo (múltiples fases)

Cuando el usuario quiere implementar múltiples features sin intervención humana, usar el **orquestador autónomo**.

**Referencia completa:** `references/pipeline-pattern.md`
**Caso real concreto:** `references/jxus-bot-api-pipeline-case-study.md`

### Principios del pipeline autónomo

1. **Fases cortas y testeables** — cada fase es un incremento lógico que puede verificarse independientemente
2. **Prompts autocontenidos** — cada prompt incluye contexto completo del proyecto + qué se implementó en fases anteriores
3. **Git tagging** — cada fase exitosa genera `git tag -f "pipeline/$step_name"`
4. **Resiliencia** — si una fase falla, el pipeline loggea el error y continúa con la siguiente
5. **Sin intervención humana** — Hermes crea todos los prompt files upfront, luego el script itera autónomamente

### Estructura recomendada

```
scripts/pipeline/
├── orchestrate.sh       # Script orquestador principal
├── logs/                # Logs de ejecución (creados automáticamente)
│   ├── pipeline-status.log
│   └── cursor-output.log
└── steps/               # Un .md por fase
    ├── p01-feature.md
    ├── p02-other.md
    └── ...
```

### Implementación rápida

```bash
# 1. Crear prompt files autocontenidos en steps/
# 2. Crear orchestrate.sh (template en references/pipeline-pattern.md)
# 3. Ejecutar en background
terminal(
  command="bash scripts/pipeline/orchestrate.sh",
  background=true,
  notify_on_complete=true,
  timeout=3600
)
```

## Pitfalls de pipelines grandes

- **Timeout**: 8+ fases a ~5-10min cada una necesitan timeout >=7200s (2h). Para pipelines completos usar 14400s (4h).
- **Parallel safety**: NUNCA ejecutar dos `agent -p --force` sobre el mismo repo al mismo tiempo. Pipeline bg + task individual es seguro si modifican archivos distintos.
- **Resilience**: el orquestador usa `set -uo pipefail` (sin `-e`) para continuar tras fallos.
- **Post-phase review**: tras `notify_on_complete`, leer `tail -30 scripts/pipeline/logs/cursor-output.log` para verificar que Cursor implemento todo lo esperado.
- **Git checkpoint**: cada fase genera `git tag -f "pipeline/$NAME"`. Retomar desde la ultima tag si el pipeline se interrumpe.
- **git add -A absorcion**: si dos instancias headless terminan sobre el mismo repo, el primer `git add -A && git commit` absorbe cambios de ambas. Monitorear con `git diff --stat` entre commits para detectar absorciones.

---

## Estrategia de ejecución (orden de preferencia)

