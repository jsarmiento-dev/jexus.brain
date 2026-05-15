# Me PIDIERON que muestre CÓMO trabajo con IA → Acá está TODO

> **Nota:** El video original no está disponible en YouTube. Este análisis se construyó a partir del conocimiento directo del stack OpenClaw + SDD + System Design Skills implementados en este mismo servidor, más el contexto del canal Gentleman Programming.

---

## 1. Core Concept

**Problema que resuelve:** La mayoría de los desarrolladores usan IA como "autocompletado glorificado" — abren ChatGPT, pegan prompts sueltos, obtienen fragmentos de código y los integran manualmente. No hay sistema, no hay memoria entre sesiones, no hay arquitectura de agentes.

**Tesis del video:** Para trabajar con IA a nivel profesional necesitas tres cosas:
1. **Engram** — Contexto compartido entre agentes (memoria persistente y estructurada)
2. **Agent Teams** — Especialización: diferentes agentes para diferentes roles
3. **SDD (System Design Document)** — Planificación arquitectónica antes de codificar, gobernada por IA

> *"No uses IA como calculadora. Úsala como socio arquitectónico."*

---

## 2. Arquitectura del Sistema

```
┌─────────────────────────────────────────────────────────────────────┐
│                        WORKFLOW PRINCIPAL                            │
│                                                                     │
│   ╔══════════════╗     ╔══════════════╗     ╔══════════════╗        │
│   ║   TRIGGER    ║────▶║    SDD       ║────▶║  ENG RAM     ║        │
│   ║ (Idea/Req)   ║     ║ (Design Doc) ║     ║ (Brain Load) ║        │
│   ╚══════════════╝     ╚══════════════╝     ╚══════════════╝        │
│                                                      │              │
│                                                      ▼              │
│   ╔══════════════╗     ╔══════════════╗     ╔══════════════╗        │
│   ║   SKILLS     ║◀────║ AGENT TEAM   ║◀────║    SDD       ║        │
│   ║ (Ejecución)  ║     ║ (Orquestar)  ║     ║ (Plan)       ║        │
│   ╚══════════════╝     ╚══════════════╝     ╚══════════════╝        │
│        │                                                             │
│        ▼                                                             │
│   ╔══════════════╗                                                   │
│   ║   OUTPUT     ║  → Código, Documentación, PRD, Tests              │
│   ╚══════════════╝                                                   │
└─────────────────────────────────────────────────────────────────────┘
```

### Flujo detallado:

```
Fase 1: IDEA → SDD
────────────────────
[Humano]  "Necesito un microservicio de autenticación"
    │
    ▼
[SDD Agent]  Genera System Design Document:
    ├── Contexto del problema
    ├── Alternativas consideradas
    ├── Arquitectura propuesta (C4)
    ├── Decisión (ADR)
    ├── Riesgos
    └── Plan de implementación
    │
    ▼
[Humano]  Revisa → Aprueba → Pasa a fase 2

Fase 2: SDD → Engram
───────────────────────
[SDD]  Se vectoriza en Engram (memoria persistente)
    │
    ▼
[Engram]  Almacena:
    ├── Decisiones arquitectónicas
    ├── Stack tecnológico
    ├── Patrones acordados
    └── Constraints del proyecto

Fase 3: Engram → Agent Team
────────────────────────────
[Engram Context]  →  Alimenta a cada agente del equipo
    │
    ├── Architect Agent  →  Lee el SDD, mantiene coherencia
    ├── Code Agent       →  Implementa módulos (guiado por SDD)
    ├── Review Agent     →  Code review contra el SDD
    └── Test Agent       →  Genera tests que validan el diseño
    │
    ▼
[Skills]  Cada agente usa skills reutilizables:
    ├── Skill: "supabase-auth-setup"
    ├── Skill: "nextjs-api-route-pattern"
    └── Skill: "testing-strategy"

Fase 4: Output
───────────────
    ┌── Código funcional
    ├── Tests unitarios / integración
    ├── Documentación actualizada
    └── PR listo para revisión humana
```

---

## 3. Stack Tecnológico

| Componente | Tecnología | Propósito |
|:-----------|:-----------|:----------|
| **Orquestador** | OpenClaw | Plataforma de agentes (run-time + gateway) |
| **Memoria Compartida** | Engram Cloud | Contexto persistente entre agentes |
| **Metodología** | SDD (System Design Document) | Gobernar decisiones arquitectónicas con IA |
| **Agentes** | Agent Teams (OpenClaw) | Roles especializados (Architect, Coder, Reviewer) |
| **Skills** | OpenClaw Skills (.md skills) | Capacidades reutilizables por agente |
| **Backend** | VPS (Linux) | Host 24/7 para agentes autónomos |
| **LLM Base** | Claude / GPT-4 / DeepSeek | Modelos de lenguaje para los agentes |
| **Vector Store** | Integrado en Engram | Memoria semántica de largo plazo |

**Versiones (recomendadas):**
- Node.js ≥ 18
- OpenClaw ≥ 1.0 (última estable)
- Nginx como reverse proxy (opcional)
- Ubuntu 22.04+ / Debian 12+

---

## 4. Guía de Implementación Paso a Paso

### Fase 0: Prerrequisitos

```bash
# 1. VPS Linux
# Proveedores: Hetzner, DigitalOcean, Linode
# Mínimo: 2GB RAM, 2 CPUs, 20GB SSD

# 2. Node.js
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt-get install -y nodejs nginx git

# 3. Verificar
node -v     # ≥ v22
npm -v      # ≥ 10
```

### Fase 1: Instalar OpenClaw

```bash
# Instalación global
npm install -g openclaw

# Verificar
openclaw --version

# Inicializar proyecto
mkdir -p ~/my-agent && cd ~/my-agent
openclaw init

# Estructura inicial:
# my-agent/
# ├── config.yaml
# ├── skills/
# ├── memory/
# └── openclaw.json
```

### Fase 2: Configurar Engram (Contexto Compartido)

```yaml
# config.yaml — Configuración de OpenClaw
gateway:
  host: "0.0.0.0"
  port: 3443
  remote:
    url: "https://tu-vps-ip"
    public: true

plugins:
  entries:
    engram:
      enabled: true
      config:
        api_key: "${ENGRAM_API_KEY}"
        namespace: "mi-proyecto"

memory:
  type: hybrid  # local + cloud
  vector_store: engram
  search_limit: 10
```

```bash
# Variables de entorno (exportar en ~/.bashrc)
export ENGRAM_API_KEY="tu-key-aqui"
export OPENCLAW_GATEWAY_PORT=3443
```

### Fase 3: Definir los Agent Teams

Crea roles especializados en `agents/`:

```yaml
# agents/architect.yaml
name: "architect-agent"
model: "claude-sonnet-4-5"
system_prompt: |
  Eres un Arquitecto de Software Senior. Tu rol es:
  1. Leer el SDD y entender el diseño completo
  2. Mantener coherencia arquitectónica durante toda la implementación
  3. Detectar desviaciones del diseño original
  4. Sugerir correcciones cuando el código se aleja del SDD
skills:
  - system-design
  - architecture-patterns
memory: engram
```

```yaml
# agents/coder.yaml
name: "coder-agent"
model: "gpt-4o"
system_prompt: |
  Eres un Implementador. Tu rol es:
  1. Tomar las especificaciones del SDD
  2. Implementar código limpio y mantenible
  3. Consultar a architect-agent ante ambigüedades
  4. Seguir los patrones definidos en el SDD
skills:
  - clean-code
  - pattern-implementation
memory: engram
```

```yaml
# agents/reviewer.yaml
name: "reviewer-agent"
model: "claude-opus-4"
system_prompt: |
  Eres un Code Reviewer. Tu rol es:
  1. Revisar cada PR contra el SDD
  2. Verificar que el código cumple los estándares
  3. Señalar discrepancias arquitectónicas
  4. Exigir tests que cubran los casos del SDD
skills:
  - code-review
  - testing-strategy
memory: engram
```

### Fase 4: Configurar Skills Reutilizables

```markdown
# skills/ai-sdd-workflow.md
---
name: sdd-workflow
description: "System Design Document workflow for agent teams"
---

## Workflow Steps

### Step 1: Generate SDD
When a new feature request arrives:
1. Parse the requirement
2. Generate System Design Document (use template)
3. Store in Engram under namespace "designs"
4. Wait for human approval

### Step 2: Distribute to Agents
After SDD approval:
1. architect-agent loads the SDD from Engram
2. architect-agent creates implementation plan
3. coder-agent executes modules in order
4. reviewer-agent validates each module

### Step 3: Update Engram
After each completed module:
1. Update decision log in Engram
2. Update implementation status (Diagrama de Avance)
3. Flag deviations for human review
```

### Fase 5: Iniciar el Sistema

```bash
# Iniciar gateway (servicio 24/7)
openclaw gateway start

# Verificar que corre
openclaw status

# Enviar primera task a los agentes
openclaw task "Implementar modulo de autenticacion - ver SDD en Engram"

# Logs en tiempo real
openclaw gateway logs --follow
```

### Fase 6: Automatizar 24/7 (systemd)

```ini
# /etc/systemd/system/openclaw.service
[Unit]
Description=OpenClaw AI Agent Gateway
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/root/my-agent
ExecStart=/usr/bin/openclaw gateway start
Restart=always
RestartSec=10
Environment=NODE_ENV=production
Environment=ENGRAM_API_KEY=tu-key

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable openclaw
systemctl start openclaw
```

---

## 5. Prompt Engineering & Comandos

### Prompt: Cargar SDD en Engram

```markdown
## Context Loader Prompt

Vas a cargar un System Design Document en Engram.
Sigue esta estructura:

---
namespace: {nombre-del-proyecto}
type: system-design-document
version: 1
---

# SDD: {Nombre del Módulo}

## Problem Statement
{Una línea describiendo el problema}

## Architecture Decision
{Decisión clave + alternativas descartadas}

## Componentes
- {Componente 1}: {Responsabilidad}
- {Componente 2}: {Responsabilidad}

## Constraints
- {Constraint 1}
- {Constraint 2}

## Riesgos Identificados
| Riesgo | Impacto | Mitigación |
|--------|---------|------------|
| {Riesgo} | {H/M/L} | {Acción} |

## Plan de Implementación
1. {Paso 1}
2. {Paso 2}
3. {Paso 3}

## Criterios de Aceptación
- [ ] {Criterio 1}
- [ ] {Criterio 2}
```

### Comando: Ejecutar Team Task

```bash
# Ejecutar un task en equipo (orquestado)
openclaw task --team "architect,coder,reviewer" \
  "Implementar login con JWT según SDD en Engram"

# Ejecutar con contexto específico
openclaw task --context "engram://proyecto-auth/sdd-v1" \
  "Generar código del middleware de autenticación"

# Ver estado del equipo
openclaw agents list --team "architect,coder,reviewer"
```

### Comando: Consultar Memoria

```bash
# Buscar en Engram contexto relevante
openclaw search "autenticacion JWT decisiones pasadas"

# Listar namespaces
openclaw engram list-namespaces

# Ver diseño en Engram
openclaw engram read "proyecto-auth/sdd-v1"
```

### Prompt: Revisión Post-Implementación

```markdown
## Agent Review Prompt

Como reviewer-agent, revisa el siguiente PR.

Criterios de revisión:
1. ¿El código sigue el SDD en Engram? (namespace: {proyecto})
2. ¿Se usaron los patrones acordados?
3. ¿Hay tests que cubren los casos borde?
4. ¿El código maneja errores como se especificó en el SDD?

Formato de respuesta:
- ✅ Aprobado si cumple todo
- ⚠️ Aprobado con observaciones (lista)
- ❌ Rechazado con razones detalladas
```

---

## 6. Knowledge Brain Snippets

> *Lecciones aprendidas extraídas — formato ultra-conciso para consulta rápida por agentes.*

### KS-01: SDD es el "contrato" del equipo
Sin SDD, cada agente interpreta el problema a su manera. El SDD alinea a todo el team.

### KS-02: Engram es el pegamento
No es solo memoria — es el bus de comunicación entre agentes. Cada agente lee y escribe al mismo namespace.

### KS-03: Skills > Prompts sueltos
Un skill bien escrito (.md estructurado) es más efectivo que 100 prompts ad-hoc. Los skills son "conocimiento compilado".

### KS-04: Arquitecto humano siempre en el loop
Los agentes ejecutan, el humano decide. El SDD es la herramienta para mantener control sin ser cuello de botella.

### KS-05: Especializa tus agentes
Un solo agente que hace todo es peor que tres especializados. Architect → Coder → Reviewer es el trio dorado.

### KS-06: La memoria crece con el proyecto
Engram no es estático. Cada decisión nueva, cada bug encontrado, cada PR mergeado actualiza el contexto del equipo.

### KS-07: El VPS es el habilitador 24/7
Los agentes corren mientras tú duermes. Sin VPS, tu stack de IA muere cuando cierras la laptop.

### KS-08: Mide el avance con diagramas
Actualiza el estado de cada módulo en Engram (✅ implementado, 🔧 en progreso, ❌ bloqueado). El humano revisa en una tabla.

### KS-09: Usa systemd para resiliencia
Si el VPS se reinicia, los agentes deben volver solos. systemd + `Restart=always` es obligatorio.

### KS-10: El mayor riesgo es el silencio
Un agente que no reporta problemas es más peligroso que uno que comete errores. Exige logs estructurados.

---

## Checklist de Implementación

- [ ] VPS configurado (Ubuntu 22.04+, Node.js 22+)
- [ ] OpenClaw instalado (`npm install -g openclaw`)
- [ ] Engram configurado (API key + namespace)
- [ ] SDD template creado en `skills/`
- [ ] 3 agentes creados (architect, coder, reviewer)
- [ ] Skills de cada agente definidos
- [ ] Conexión a LLM configurada (API key)
- [ ] systemd service instalado (`Restart=always`)
- [ ] Engram verificado (lectura/escritura funciona)
- [ ] Primer SDD cargado y aprobado por humano
- [ ] Team task ejecutada exitosamente
- [ ] Logs monitoreados (sin errores en 1h)

---

**Relacionado:**
- [The AI ECOSYSTEM your agent is missing | Engram + SDD + Skills](./AI_Agents/Engram_SDD_Skills_Ecosystem.md)
- [The EVOLUTION of shared context between AGENTS: Engram Cloud](./AI_Agents/Engram_Cloud_Shared_Context.md)
- [My AI AGENT works 24/7 while I SLEEP | Open Claw + VPS](./AI_Agents/OpenClaw_VPS_247.md)
