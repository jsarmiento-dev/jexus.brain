# 🎬 Me PIDIERON que muestre CÓMO trabajo con IA → Acá está TODO

> **Video:** `https://www.youtube.com/watch?v=c5Gwx0RcxNE`
> **Creador:** [Gentleman Programming](https://www.youtube.com/@gentlemanprogramming)
> **Transcripción:** Original obtenida vía TurboScribe
> **Análisis:** Basado en transcripción completa del video

---

## 📋 Ficha Técnica

| Campo | Valor |
|:------|:------|
| **Título** | Me PIDIERON que muestre CÓMO trabajo con IA → Acá está TODO \| Engram + Agent Teams + SDD |
| **Creador** | Gentleman Programming |
| **Duración** | ~90 min (en vivo / grabación extendida) |
| **Nivel** | Avanzado |
| **Categoría** | Arquitectura de Agentes IA + Workflows |
| **Stack principal** | Go, OpenClaw/OpenCode, Bubble Tea, SQLite (FTS5), Ngram, GGA, Agent Teams Lite |
| **Repositorios clave** | [Ngram](https://github.com/gentlemanprogramming/ngram), [GGA](https://github.com/gentlemanprogramming/gga), [Agent Teams Lite](https://github.com/gentlemanprogramming/agent-teams-lite) |

---

## 1. Resumen

El video es una demostración en vivo del ecosistema completo que Gentleman Programming ha construido para trabajar con IA a nivel profesional. Muestra **3 herramientas principales** que funcionan juntas, y un **4to proyecto revelado al final** como el "gran unificador".

### Las 3 Herramientas + 1

| # | Herramienta | Propósito | Estrellas (al momento) |
|:-:|:-----------|:----------|:-----------------------|
| 1 | **Ngram** | Memoria persistente para cualquier agente IA | ~525 ★ |
| 2 | **GGA** (Gentleman Guardian Angel) | Code review automatizado con IA | ~625 ★ |
| 3 | **Agent Teams Lite** | Framework SDD con sub-agentes orquestados | Nueva (~8 commits, 5 releases) |
| 4 | **AI Gentle Stack** *(revelado)* | Instalador unificado de todo el ecosistema | No publicado aún |

---

## 2. Ngram — Memoria Persistente para Agentes

### ¿Qué problema resuelve?

Los agentes IA tienen contexto limitado. Cuando el contexto se llena, ocurre una **compaction** (resumen comprimido) y se pierde información. Soluciones existentes como **ClaudeMem** requieren Python, Node, bases de datos vectoriales y están atadas a la nube.

### La solución de Ngram

> **Un solo binario. Cero dependencias. Go. SQLite. FTS5.**

```
┌────────────────────────────────────────────────┐
│                 NGRAM (Go)                      │
│                                                  │
│  ┌──────────────────┐   ┌──────────────────┐    │
│  │    SQLite DB      │   │      FTS5         │    │
│  │  (datos locales)  │   │  (full-text search)│   │
│  └──────────────────┘   └──────────────────┘    │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │           MCP Interface                   │   │
│  │  (cualquier agente → HTTP/MCP)            │   │
│  └──────────────────────────────────────────┘   │
└────────────────────────────────────────────────┘
```

### Cómo funciona

1. **Cada agente** (OpenClaw, Codex, Gemini CLI, VSCode, Anti-Gravity) recibe un **system prompt**:
   > *"Cada vez que aprendas algo nuevo — arquitectura, decisión, bug fix, release — guárdalo en la base de datos."*

2. **Cuando un nuevo agente/sesión comienza**, lo primero que hace es consultar Ngram:
   > *"¿Hay información sobre lo que voy a hacer?"*

3. **Búsqueda ultra rápida** via FTS5 — ranking de ocurrencias sin necesidad de vector store.

### Diferencia clave con ClaudeMem

| Aspecto | ClaudeMem | Ngram |
|:--------|:----------|:------|
| **Dependencias** | Python + Node + 2 DBs | Go → 1 binario |
| **Infraestructura** | Cloud | Local (SQLite) |
| **Búsqueda** | Vectorial | FTS5 (texto completo) |
| **Complejidad** | Alta | Mínima |
| **Costo** | Cloud (pago) | Local (gratis) |
| **Agentes** | Solo Claude | Cualquiera (MCP) |

---

## 3. GGA — Gentleman Guardian Angel

### ¿Qué es?

Una alternativa open-source a **Rabbit Code** (herramienta paga de code review con IA). GGA permite revisar código usando **cualquier CLI de IA** (OpenCode, Codex, Gemini, Claude) con **configuraciones personalizables**.

### Modos de ejecución

```
┌─────────────────────────────────────────────────┐
│              GGA (Go)                             │
│                                                    │
│  1. 🔧 Pre-commit hook                            │
│     → Revisa antes de hacer commit                 │
│                                                    │
│  2. 📤 Pre-push hook                              │
│     → Revisa antes de pushear                      │
│                                                    │
│  3. 🤖 CI/CD (GitHub Actions)                     │
│     → Revisa en la pipeline de integración        │
└─────────────────────────────────────────────────┘
```

### Smart Caching

El cache es la clave para que no queme tokens:

```
Archivos a commitear:
├── src/auth.ts       → Ya pasó revisión → ✅ Cache hit → No revisar
├── src/api.ts        → Cambió → ❌ Cache miss → Revisar
└── src/utils.ts      → No cambió desde última revisión → ✅ Saltar
```

Esto significa que solo revisa **lo que realmente cambió**, ahorrando tokens masivamente.

### Estado del proyecto

- **625+ estrellas** en GitHub
- **63+ forks**
- Multiples contribuciones de la comunidad
- Documentación extensa incluida en el repositorio

---

## 4. Agent Teams Lite — El Framework SDD

### El problema original

Los sistemas multi-agente existentes (como el nativo de OpenCode/Claude Code) tienen 3 problemas:

1. **Solo funcionan para su plataforma** — no son portables
2. **Queman tokens** — el orquestador se llena de contexto
3. **No hay comunicación entre sub-agentes** — cada uno trabaja en "lienzo en blanco"

### La arquitectura real

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ORQUESTADOR (main agent)                      │
│                       Contexto: ~46K tokens                          │
│                       NUNCA escribe código                           │
│                              │                                       │
│       ┌──────────────────────┼──────────────────────────┐            │
│       │                      │                          │            │
│  ┌────▼──────┐    ┌─────────▼─────────┐    ┌──────────▼──────┐     │
│  │ PROPOSE   │    │     EXPLORE       │    │     SPECS       │     │
│  │ (skill)   │    │     (skill)       │    │     (skill)     │     │
│  │ Propuesta │    │  Investigar repo  │    │  Especificación │     │
│  │ inicial   │    │  y ecosistema     │    │  técnica        │     │
│  └────┬──────┘    └─────────┬─────────┘    └──────────┬──────┘     │
│       └──────────────────┬──┴──────────────┬───────────┘           │
│                          │                 │                        │
│                   ┌──────▼──────┐   ┌──────▼──────────┐            │
│                   │   DESIGN    │   │     APPLY       │            │
│                   │   (skill)   │   │     (skill)     │            │
│                   │ Arquitectura│   │  Implementación │            │
│                   └──────┬──────┘   └──────┬──────────┘            │
│                          │                 │                        │
│                          └────────┬────────┘                        │
│                                   │                                 │
│                            ┌──────▼──────┐                         │
│                            │    NGRAM     │                         │
│                            │  (SQLite DB) │ ← Source of Truth       │
│                            └─────────────┘                         │
└─────────────────────────────────────────────────────────────────────┘
```

### Regla de oro del orquestador

> **"El orquestador NUNCA implementa código. Siempre delega a sub-agentes."**

¿Por qué? Porque mantener el contexto del orquestador limpio es la prioridad #1. En la demo, el orquestador se mantuvo en **~46K tokens** durante toda la sesión mientras los sub-agentes hacían el trabajo pesado con contextos grandes.

### ¿Por qué skills?

Cada sub-agente en Agent Teams Lite es implementado como una **skill**. Esto porque:

- Las skills solo se cargan **cuando se necesitan** → no queman contexto innecesario
- Funcionan en **cualquier plataforma** (OpenClaw, OpenCode, Codex, Gemini CLI)
- Son **agnósticas al lenguaje** de programación
- Se pueden compartir y versionar

### El flujo completo

```
1. IDEA
   ↓
2. ORQUESTADOR lee PRD
   ↓
3. Sub-agent PROPOSE → Propuesta de solución
   ↓ (guarda en Ngram)
4. Sub-agent EXPLORE → Investiga repos, dependencias, ecosistema
   ↓ (guarda en Ngram)
5. Sub-agent SPECS → Especificaciones técnicas detalladas
   ↓ (guarda en Ngram)
6. Sub-agent DESIGN → Arquitectura, componentes, estructura de carpetas
   ↓ (guarda en Ngram)
7. Sub-agent APPLY → Implementa el código (¡el único que codea!)
   ↓
8. RESULTADO → Código, tests, documentación
```

### Multimodelo demostrado

Durante la demo, el orquestador cambió de modelo en vivo:
- Inició con un modelo
- Cambió a **Gemini Codex** por restricciones de sesión
- Luego a **GPT** sin ningún problema
- La arquitectura es **multimodelo nativa**

---

## 5. La Demo en Vivo: AI Gentle Stack

### El proyecto

El video culmina con la revelación de un proyecto que estaban construyendo **en vivo durante la transmisión**:

**AI Gentle Stack** — un **instalador unificado** con TUI que:

1. Detecta el **sistema operativo** (macOS Apple Silicon, Intel, Linux, Windows)
2. Presenta una **interfaz TUI** para elegir agentes
3. **Resuelve dependencias** automáticamente
4. **Configura cada agente** usando todo el ecosistema Gentleman
5. **Verifica** que todo funcione

### Stack técnico del instalador

| Componente | Tecnología |
|:-----------|:----------|
| **Lenguaje** | Go |
| **TUI** | Bubble Tea + Lipgloss |
| **CLI** | Cobra (modo interactivo y no interactivo) |
| **Screens TUI** | 9 pantallas planificadas |
| **Arquitectura** | 8 paquetes internos (system, agents, components, presentations, backup, installer, etc.) |
| **Instalación** | Cross-platform (Win/ Mac/ Linux) |
| **Ngram** | Integrado para persistencia |
| **SDD** | PRD-driven development |
| **GGA** | Code review integrado |
| **Backup** | Sistema de backup de configuraciones existentes |
| **Plugin system** | Interfaz de agente → add new = implementar interfaz |

### El PRD

El proyecto comenzó con un **PRD (Product Requirements Document)** completo que definía:

- Reglas de negocio
- Requerimientos funcionales
- Experiencia de usuario
- Especificaciones técnicas

Todo gobernado por **Spec-Driven Development**: primero las especificaciones, luego los agentes implementan.

### Resultado de la demo

En ~45 minutos de desarrollo en vivo, el sistema:
- Leyó el PRD completo
- Exploró el ecosistema existente
- Generó especificaciones técnicas detalladas
- Diseñó la arquitectura completa
- Creó las tasks de implementación divididas en **8 fases, 9 batches de apply**
- El orquestador nunca pasó de ~46K tokens

---

## 6. Lecciones Clave

### 6.1 Gestión de Contexto

El problema #1 de los sistemas multi-agente no es la calidad del código, es **el contexto**. Tres estrategias:

1. **Ngram** → Persistencia externa (no depende del contexto del agente)
2. **Sub-agentes** → El trabajo pesado va a sesiones hijas
3. **Orquestador limpio** → Nunca codea, solo orquesta

### 6.2 Skills como Bloques

Cada skill es un sub-agente. No se cargan hasta que se necesitan. Esto es **lazy loading para agentes**.

### 6.3 No reinventes el State Management

> Gentleman menciona que lleva años haciendo sus propios state managers: Redux → alternativa propia, NGX Store → versión propia, Signals → versión agnóstica. SDD + Ngram es su "state manager" para agentes IA.

### 6.4 El Instalador es el Gateway

El AI Gentle Stack no es solo un instalador — es la **puerta de entrada** al ecosistema. Resuelve el mayor problema de adopción: "esto es un quilombo para instalar".

---

## 7. Comparativa con el Blueprint Original

| Aspecto | Blueprint Original (adivinado) | Real (transcripción) |
|:--------|:-------------------------------|:---------------------|
| **Engram** | Cloud service (vector store) | Local SQLite + FTS5, Go binary |
| **Agent Teams** | Architect/Coder/Reviewer | Propose/Explore/Specs/Design/Apply |
| **Skill system** | Mencionado genéricamente | Cada sub-agente = skill (lazy loaded) |
| **Stack** | OpenClaw + Node.js | Go + OpenClaw/Codex/Gemini (agnóstico) |
| **GGA** | No mencionado | Herramienta central (~625★) |
| **Estado** | Recomendaba subir | Mantener ambos (diferentes perspectivas) |

> Ambos blueprints son válidos — el original describe una arquitectura funcional usando OpenClaw; este describe la arquitectura REAL que Gentleman Programming implementó.

---

## 8. Knowledge Snippets

### KS-01: La regla del 46K
El orquestador nunca debe escribir código. Mantén su contexto bajo (~46K tokens) delegando toda implementación a sub-agentes. Cada sub-agente quema su propio contexto, el orquestador se mantiene limpio.

### KS-02: Ngram > Vector Store
Para equipos pequeños, FTS5 sobre SQLite es más que suficiente. No necesitas una base vectorial hasta que tengas millones de registros. Simple > Complejo.

### KS-03: Smart Caching en GGA
No revises lo que ya pasó. GGA cachea resultados de revisión por archivo. Si un archivo no cambió entre commits, no se revisa de nuevo. Esto ahorra ~60-80% de tokens.

### KS-04: Skills como sub-agentes
Cada skill es un sub-agente perezoso. No se carga en contexto hasta que se necesita. Esto escala mejor que tener todos los skills precargados.

### KS-05: Cross-platform desde el día 1
El instalador (AI Gentle Stack) detecta SO, arquitectura, paquetes disponibles. No esperes a "después" para hacerlo portable — hazlo cross-platform desde el primer commit.

### KS-06: PRD primero, código después
Spec-Driven Development: escribe el PRD, luego los specs técnicos, luego el diseño, y SOLO entonces el código. Los agentes siguen este orden automáticamente.

### KS-07: Ngram es el bus de comunicación
Cada sub-agente guarda en Ngram lo que aprende. Cuando otro sub-agente necesita saber algo, consulta Ngram. Es el "source of truth" compartido entre sesiones.

### KS-08: Cambiar de modelo es gratis
La arquitectura es multimodelo nativa. El orquestador puede cambiar de Claude a GPT a Gemini sin reconfigurar nada. El modelo es un parámetro, no un acoplamiento.

### KS-09: 9 pantallas TUI es ambicioso
Bubble Tea con 9 screens diferentes requiere planificación cuidadosa de navegación. Gentleman lo diseñó con 8 paquetes internos para mantenerlo modular.

### KS-10: El ecosistema tiene 3 repositorios
Ngram (525★), GGA (625★), Agent Teams Lite (nuevo). No son proyectos separados — son un ecosistema. Cada uno resuelve una parte del problema.

---

## 9. Checklist de Implementación

- [ ] Entender Ngram: Go binary, SQLite + FTS5, MCP interface, sin dependencias
- [ ] Entender GGA: pre-commit/pre-push/CI hook, smart caching, multi-CLI
- [ ] Entender Agent Teams Lite: 5 sub-agentes, skills lazy-loaded, orquestador limpio
- [ ] Ver demo del AI Gentle Stack: Bubble Tea TUI, Go, cross-platform installer
- [ ] Analizar PRD-driven development en la práctica
- [ ] Comparar con mi stack actual (OpenClaw + system-design skill)
- [ ] Decidir qué incorporar al flujo actual

---

## 10. Recursos

- **Ngram:** `https://github.com/gentlemanprogramming/ngram`
- **GGA:** `https://github.com/gentlemanprogramming/gga`
- **Agent Teams Lite:** `https://github.com/gentlemanprogramming/agent-teams-lite`
- **YouTube:** `https://www.youtube.com/watch?v=c5Gwx0RcxNE`
- **OpenSpec.dev:** `https://openspec.dev` (mencionado como alternativa SDD)
- **Bubble Tea:** `https://github.com/charmbracelet/bubbletea`
- **Lipgloss:** `https://github.com/charmbracelet/lipgloss`
- **Go:** `https://go.dev`

---

*Análisis generado el 15 de mayo de 2026 basado en transcripción completa del video.*
*Blueprint anterior mantenido como referencia de arquitectura alternativa (OpenClaw-based).*
