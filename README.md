# 🧠 Jexus Brain

**Investigación y documentación sobre las mejores prácticas para trabajar con Inteligencia Artificial.**

Repositorio público de investigación: analizamos videos, extraemos conocimiento y construimos una base de aprendizaje estructurada sobre IA para desarrolladores.

---

## 🗂️ Estructura del Conocimiento

```
jexus-brain/
├── creators/                      # Análisis por creador de contenido
│   ├── gentleman-programming/     # @gentlemanprogramming
│   │   ├── AI_GentlemanProgramming_Index.md   # Índice maestro
│   │   └── analisis/              # Análisis detallados por video
│   ├── midudev/                   # (futuro)
│   └── dotnetdo/                  # (futuro)
│
├── topics/                        # Vistas transversales por tema
│   ├── llm-agents/                # Agentes, subagentes, memoria
│   ├── automatizacion/            # n8n, workflows, AI pipelines
│   ├── herramientas-ai/           # Claude, Copilot, Cursor, Windsurf
│   ├── ingenieria-prompts/        # Prompting, skills, system design
│   ├── arquitectura-software/     # Clean Architecture, patrones
│   └── cloud-devops/              # Infra, VPS, deploy
│
├── meta/                          # Metadatos del sistema
│   ├── templates/                 # Templates reutilizables
│   │   └── video-analysis.md      # Template para análisis de video
│   ├── glosario/                  # Términos y conceptos
│   └── metodologia/               # ¿Cómo investigamos y catalogamos?
│
├── notes/                         # Notas generales de investigación
├── resources/                     # Enlaces, papers, referencias
└── README.md
```

## 📋 Metodología de Análisis

Cada video se analiza siguiendo el template en `meta/templates/video-analysis.md` con estos campos:

| Campo | Descripción |
| :--- | :--- |
| **Título** | Nombre exacto del video |
| **Categoría** | Sub-tema (LLMs, Agentes, Automatización, Herramientas...) |
| **Duración** | [HH:MM:SS] |
| **Sinopsis** | Resumen ejecutivo de conceptos clave |
| **Enlace** | URL directa al video |
| **Complejidad** | Básico / Intermedio / Avanzado |

## 🚀 Próximos Pasos

- [x] Escaneo inicial: **Gentleman Programming** (43 videos de IA catalogados)
- [ ] Analizar en profundidad videos seleccionados
- [ ] Agregar siguiente creador (MiduDev, DotNetDo...)
- [ ] Construir vistas transversales por tema
- [ ] Crear glosario de términos
