<div align="center">

# ⚡ ControlIA

**SaaS de ventas conversacional para PYMEs — un sistema multi-agente que opera un negocio completo desde Telegram y WhatsApp.**

[![Built with VoltAgent](https://img.shields.io/badge/built%20with-VoltAgent-blue)](https://voltagent.dev)
[![Node](https://img.shields.io/badge/node-%3E%3D22-brightgreen)](https://nodejs.org)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.x-3178c6)](https://www.typescriptlang.org/)
[![PostgreSQL + pgvector](https://img.shields.io/badge/PostgreSQL-pgvector-336791)](https://github.com/pgvector/pgvector)
[![Prisma](https://img.shields.io/badge/ORM-Prisma-2D3748)](https://www.prisma.io/)

</div>

---

## ¿Qué es ControlIA?

ControlIA es un **asistente de negocio conversacional multi-tenant** para PYMEs (nace en el sector de distribución de lubricantes/aceites, pero es genérico). El usuario habla en lenguaje natural —por texto, **voz** o **foto**— desde **Telegram** o **WhatsApp**, y un sistema de **11 sub-agentes de IA** coordinados por un **supervisor** ejecuta la operación: CRM, cotizaciones, inventario, finanzas, cobranzas, rutas de prospección, generación de documentos y contenido para redes.

Además de responder, el sistema es **autónomo**: aprende capacidades nuevas bajo demanda (Architect Agent) y se auto-repara ante errores (self-healing).

El repositorio es un **monorepo** con dos cerebros complementarios y una webapp:

| Componente | Rol | Stack |
|---|---|---|
| **`controlia-agent/`** | Núcleo de IA: sistema multi-agente (11 sub-agentes + supervisor) | VoltAgent · Node 22 · TypeScript · Prisma · PostgreSQL + pgvector · Whisper |
| **Bot Python (raíz)** | Bot de Telegram multi-tenant con suscripción, onboarding y motor de rutas | Python 3.11 · pyTelegramBotAPI · psycopg3 · scikit-learn · reportlab |
| **`webapp/`** | Mini panel web estático | HTML/CSS/JS |

---

## Arquitectura de los 11 sub-agentes

Un **Supervisor Agent** recibe cada mensaje, entiende la intención y delega en el sub-agente adecuado (patrón orquestador con VoltAgent). Los sub-agentes comparten memoria semántica (pgvector) y una base de datos PostgreSQL multi-tenant con aislamiento por `tenant`.

```
                        ┌───────────────────────────┐
        Telegram  ────► │      SUPERVISOR AGENT      │ ◄────  WhatsApp
        (voz/foto)      │  (routing + orquestación)  │        (voz/texto)
                        └─────────────┬─────────────┘
                                      │ delega
   ┌──────────┬──────────┬───────────┼───────────┬──────────┬──────────┐
   ▼          ▼          ▼           ▼           ▼          ▼          ▼
┌──────┐  ┌───────┐  ┌─────────┐ ┌─────────┐ ┌────────┐ ┌────────┐ ┌──────────┐
│ CRM  │  │ Sales │  │Inventory│ │ Finance │ │Document│ │Content │ │ Context  │
└──────┘  └───────┘  └─────────┘ └─────────┘ └────────┘ └────────┘ └──────────┘
   ┌──────────────┬──────────────┬──────────────────┐
   ▼              ▼              ▼                  ▼
┌──────────┐ ┌──────────┐ ┌──────────────┐  ┌──────────────────────┐
│ Research │ │ Routing  │ │Notifications │  │      ARCHITECT       │
└──────────┘ └──────────┘ └──────────────┘  │ auto-aprendizaje +   │
                                            │ auto-curación (self- │
                                            │ healing) + visión    │
                                            └──────────────────────┘
```

| # | Sub-agente | Responsabilidad |
|---|------------|-----------------|
| 1 | **CRM** | Clientes, notas de visita, pipeline comercial |
| 2 | **Sales** | Pedidos, cotizaciones, márgenes, cobranzas |
| 3 | **Inventory** | Stock, alertas, catálogo y valorización de inventario |
| 4 | **Finance** | Estado de resultados, flujo de caja, aging / cuentas por cobrar |
| 5 | **Document** | Generación de PDFs (cotizaciones, remisiones) y Excel |
| 6 | **Content** | Posts para redes sociales, newsletters, copy |
| 7 | **Context** | Memoria semántica (embeddings + pgvector), búsqueda y consolidación de memoria |
| 8 | **Research** | Búsqueda web / investigación (Tavily MCP) para enriquecer respuestas |
| 9 | **Routing** | Geocodificación y optimización de rutas de prospección/entrega |
| 10 | **Notifications** | Recordatorios, clientes en mora, pronóstico de demanda |
| 11 | **Architect** | Auto-aprendizaje (genera herramientas nuevas), auto-curación y procesamiento de imágenes |

> El **Supervisor** no es un sub-agente de negocio: es el orquestador que enruta hacia los 11 anteriores.

---

## Stack técnico

- **VoltAgent** (`@voltagent/core`, `@voltagent/postgres`, `@voltagent/server-hono`) — framework multi-agente.
- **Node.js 22 + TypeScript**, build con `tsdown`, lint con Biome.
- **Vercel AI SDK** con múltiples proveedores: OpenAI (GPT-4o), Anthropic (Claude), Google (Gemini) y DeepSeek — conmutables por configuración.
- **PostgreSQL + Prisma** como capa de datos multi-tenant (esquemas para Postgres/Render y SQLite para desarrollo).
- **pgvector** (`vector(1536)`) para memoria semántica y RAG (`services/embeddings.ts`, migración `add_pgvector_embedding`).
- **Whisper** (`whisper-1` vía OpenAI) para transcribir notas de voz entrantes (`services/voice-transcription.ts`).
- **Canales**: adapters de **Telegram** (`adapters/telegram.ts`) y **WhatsApp** (`adapters/whatsapp.ts`).
- **Docker** listo para despliegue; el bot Python se despliega en **Render** (`render.yaml`).

---

## ¿Cómo se usa? (Telegram / WhatsApp)

El usuario final interactúa 100% por chat. Ejemplos:

- 🗣️ *(nota de voz)* "Registra un pedido de 5 canecas de aceite 15W40 para Ferretería El Roble" → Whisper transcribe → **Sales** crea el pedido.
- 📸 *(foto de una factura)* → **Architect** (visión) la lee → **Finance** la registra.
- 💬 "¿Qué clientes están en mora?" → **Notifications/Finance** responde con el aging.
- 💬 "Genérame la cotización en PDF" → **Document** devuelve el PDF.
- 💬 "Dame la ruta de prospección de hoy" → **Routing** optimiza y exporta la ruta.
- 💬 "Aprende a leer códigos QR" → **Architect** investiga, genera `tools/vision/qr-reader.ts` y confirma la nueva capacidad.

---

## Estructura del repositorio

```
ControlIA/
├── controlia-agent/          # 🧠 Núcleo IA (VoltAgent / TypeScript)
│   ├── src/
│   │   ├── agents/           # 11 sub-agentes + supervisor
│   │   ├── adapters/         # telegram.ts · whatsapp.ts
│   │   ├── tools/            # herramientas por dominio (finance, charts, routing, vision, ...)
│   │   ├── services/         # embeddings, voice-transcription, self-healing, ...
│   │   ├── guardrails/       # aislamiento por tenant + control de suscripción
│   │   ├── prompts/          # prompts de los agentes
│   │   └── index.ts
│   └── prisma/               # schema.prisma (+ variantes postgres/sqlite/render)
├── bot.py                    # 🐍 Bot Telegram multi-tenant (Python)
├── handlers/                 # admin, crm, sales, finance, logistics, onboarding, support
├── routing_engine.py         # Motor de rutas (VROOM / scikit-learn)
├── webapp/                   # Panel web estático
├── render.yaml · Procfile    # Despliegue del bot Python en Render
└── requirements.txt
```

---

## Instalación

### Núcleo IA (`controlia-agent/`)

**Requisitos:** Node.js 22+, PostgreSQL con extensión `pgvector`, API keys (OpenAI y/o Anthropic/Google/DeepSeek), token de bot de Telegram.

```bash
cd controlia-agent
npm install
cp .env.example .env          # editar credenciales (DATABASE_URL, OPENAI_API_KEY, TELEGRAM_BOT_TOKEN, ...)
npx prisma generate
npm run build
npm start                     # producción
# o
npm run dev                   # desarrollo con hot reload
```

**Docker:**

```bash
cd controlia-agent
docker build -t controlia-agent .
docker run -p 3141:3141 --env-file .env controlia-agent
```

### Bot Python (raíz)

**Requisitos:** Python 3.11, PostgreSQL (Supabase), `TELEGRAM_BOT_TOKEN`, opcional `GOOGLE_API_KEY`.

```bash
pip install -r requirements.txt
python bot.py
```

---

## Scripts útiles (`controlia-agent/`)

| Comando | Acción |
|---|---|
| `npm run dev` | Desarrollo con recarga en caliente |
| `npm run build` | Compila con `tsdown` |
| `npm start` | Ejecuta la build de producción |
| `npm run typecheck` | Chequeo de tipos (`tsc --noEmit`) |
| `npm test` | Tests con Vitest |
| `npm run backup` | Respaldo de base de datos |

---

## Seguridad y multi-tenancy

- **Aislamiento por tenant** (`guardrails/tenant-isolation.ts`) en cada consulta.
- **Control de suscripción** (`guardrails/subscription-check.ts`): trial y precio mensual configurables.
- Validación de inputs y auditoría de los cambios que genera el Architect Agent.
- Sin exposición de secretos en logs.

---

<div align="center">
  <sub>Construido con ❤️ usando <a href="https://voltagent.dev">VoltAgent</a> · 🎓 Auto-aprendizaje · 🩺 Auto-curación · 🤖 11 sub-agentes</sub>
</div>
