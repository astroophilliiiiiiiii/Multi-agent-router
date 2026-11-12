<div align="center">

# 🌐 OmniRouter

### The Intelligent LLM Gateway

**One interface. Every model. Automatic failover.**

[![Live Demo](https://img.shields.io/badge/🚀_Live_Demo-multi--agent--router--phi.vercel.app-000?style=for-the-badge)](https://multi-agent-router-phi.vercel.app/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.7-3178C6?style=flat-square&logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![Next.js](https://img.shields.io/badge/Next.js-15-000?style=flat-square&logo=next.js)](https://nextjs.org/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-17-4169E1?style=flat-square&logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Tests](https://img.shields.io/badge/Tests-31_passing-brightgreen?style=flat-square)]()
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)]()

<br />

[Live Demo](https://multi-agent-router-phi.vercel.app/) · [Features](#-features) · [Architecture](#-architecture) · [Quick Start](#-quick-start) · [API Reference](#-openai-compatible-api)

</div>

---

## 📌 What is OmniRouter?

OmniRouter is a **self-hosted LLM gateway** that lets you chat with models from **Anthropic, OpenAI, and Google** through a single unified interface. Bring your own API keys (BYOK), and the router automatically selects the best model for each request, fails over mid-stream if a provider goes down, and tracks your costs — all while exposing an **OpenAI-compatible API** so any existing SDK works out of the box.

> **TL;DR** — Think of it as a smart reverse-proxy for LLMs: plug in your keys, and it handles routing, failover, rate limiting, and cost tracking for you.

---

## ✨ Features

| Category | What it does |
|---|---|
| **Multi-Provider Routing** | Routes across **4 provider adapters** (Anthropic, OpenAI, Google, OpenAI-compatible) and **13 pre-cataloged models** |
| **Cascade Failover** | Classifies **10 error classes** and automatically fails over across up to **3 fallback models** — including **mid-stream handoff** with continuation prompts |
| **Circuit Breakers** | Per-model health tracking with exponential backoff (**30 s → 15 min**), EMA-based latency/success metrics, and `closed → open → half_open` state machine |
| **Intelligent Scoring** | Extracts **6 task families** (code, reasoning, vision, writing, summarization, general) and ranks on a **4-dimensional score** (quality, cost, latency, reliability) |
| **Envelope Encryption** | User API keys secured with **AES-256-GCM** envelope encryption (per-key DEK + master KEK) with SHA-256 fingerprinting |
| **SSRF Protection** | Multi-layer guard blocking **13+ private IP/host patterns**, DNS re-binding, cloud metadata, and redirect chains |
| **Rate Limiting** | Tiered limits: guest **10**, authenticated **30**, API **60 req/min** — with monthly USD spending caps |
| **Built-in Tools** | `web_search`, `fetch_url`, `get_current_time` — with per-tool **20 s timeout** and SSRF-safe execution |
| **Context Compaction** | Auto-drops oldest turns to fit context windows, retries before cascading |
| **OpenAI-Compatible API** | Drop-in `/v1/chat/completions` and `/v1/models` endpoints with SSE streaming |
| **Google OAuth** | Full authentication via Better Auth with guest fallback mode |
| **Usage Analytics** | Per-request cost tracking, daily rollups per provider, and audit logging |

---

## 🏗 Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client Layer                            │
│   Web UI (Next.js 15 + React 19)  ·  OpenAI SDK  ·  curl       │
└──────────────┬──────────────────────────────┬───────────────────┘
               │  /api/chat (SSE)             │  /v1/chat/completions
               ▼                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Gateway Layer (Next.js API Routes)         │
│  Auth · Rate Limiter · Budget Gate · Request Validation (Zod)   │
└──────────────────────────────┬──────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Router Core                              │
│                                                                 │
│  ┌──────────┐   ┌──────────────┐   ┌──────────────────────┐    │
│  │  Task     │   │  Candidate   │   │  Cascade Engine      │    │
│  │  Detector │──▶│  Scorer      │──▶│  (failover + handoff)│    │
│  │  (6 types)│   │  (4-dim)     │   │                      │    │
│  └──────────┘   └──────────────┘   └──────────┬───────────┘    │
│                                                │                │
│  ┌──────────────────┐   ┌─────────────────┐   │                │
│  │  Circuit Breaker  │   │  Context        │   │                │
│  │  (per-model EMA)  │   │  Compactor      │◀──┘                │
│  └──────────────────┘   └─────────────────┘                    │
└──────────────────────────────┬──────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Provider Adapters                            │
│                                                                 │
│   ┌───────────┐  ┌────────┐  ┌────────┐  ┌──────────────────┐  │
│   │ Anthropic  │  │ OpenAI │  │ Google │  │ OpenAI-Compatible│  │
│   │ Claude 5.5 │  │ GPT-4.1│  │ Gemini │  │ (Ollama, vLLM)  │  │
│   │ Opus/Sonnet│  │ o4-mini│  │ 2.5 Pro│  │                  │  │
│   └───────────┘  └────────┘  └────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Data Layer                                  │
│                                                                 │
│   PostgreSQL (Drizzle ORM)         Crypto Vault                 │
│   ├── 20 tables, 3 migrations      AES-256-GCM envelope        │
│   ├── users, sessions, orgs         encryption for all          │
│   ├── conversations, messages        stored API keys            │
│   ├── runs, feedback, audit_log                                 │
│   └── usage_daily (cost rollups)                                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🛠 Tech Stack

| Layer | Technology |
|---|---|
| **Framework** | Next.js 15 (App Router, Turbopack) |
| **Language** | TypeScript 5.7 (strict) |
| **Frontend** | React 19, Tailwind CSS 4, Lucide Icons |
| **Auth** | Better Auth + Google OAuth |
| **Database** | PostgreSQL 17 + Drizzle ORM |
| **Encryption** | AES-256-GCM (Web Crypto API) |
| **Validation** | Zod |
| **Monorepo** | Turborepo (7 packages) |
| **Testing** | Vitest (31 unit tests) |
| **Deployment** | Vercel (Edge + Serverless) |
| **Search** | Tavily API (optional) |

---

## 📦 Project Structure

```
omnirouter/
├── apps/
│   └── web/                          # Next.js 15 application
│       └── src/
│           ├── app/
│           │   ├── api/              # 9 REST API routes
│           │   │   ├── auth/         # Better Auth handler
│           │   │   ├── chat/         # SSE chat streaming
│           │   │   ├── conversations/
│           │   │   ├── credentials/  # BYOK key management
│           │   │   ├── models/       # Model catalog
│           │   │   ├── router-keys/  # API key CRUD
│           │   │   └── usage/        # Spend tracking
│           │   └── v1/               # OpenAI-compatible surface
│           │       ├── chat/completions/
│           │       └── models/
│           ├── components/           # React UI components
│           ├── hooks/                # Custom React hooks
│           └── lib/                  # Core business logic
│               ├── router.ts         # Streaming orchestrator
│               ├── limits.ts         # Rate limiter + budget
│               ├── credentials.ts    # Key vault integration
│               └── store/            # Memory / Postgres toggle
│
├── packages/
│   ├── canonical/                    # Shared types, SSE codec, error taxonomy
│   ├── crypto/                       # AES-256-GCM envelope encryption vault
│   ├── db/                           # Drizzle schema (20 tables) + migrations
│   ├── providers/                    # Anthropic, OpenAI, Google adapters
│   ├── router-core/                  # Scoring, cascade, circuit breaker
│   └── tools/                        # web_search, fetch_url, SSRF guard
│
├── turbo.json                        # Turborepo pipeline config
├── package.json                      # Workspace root
└── .env.example                      # All environment variables
```

---

## 🚀 Quick Start

### Prerequisites

- **Node.js** ≥ 22
- **PostgreSQL** 17 (or [Neon](https://neon.tech) free tier) — *optional, runs in-memory without it*

### 1. Clone & Install

```bash
git clone https://github.com/astroophilliiiiiiiii/Multi-agent-router.git
cd Multi-agent-router
npm install
```

### 2. Configure Environment

```bash
cp .env.example apps/web/.env.local
```

Edit `apps/web/.env.local` with your values:

```env
# ── Required ──
DATABASE_URL=postgresql://user:pass@host/dbname?sslmode=require
OMNIROUTER_MASTER_KEY=          # 64 hex chars: node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"

# ── Google Sign-In ──
BETTER_AUTH_SECRET=              # another 64 hex chars
BETTER_AUTH_URL=http://localhost:3000
GOOGLE_CLIENT_ID=your-client-id
GOOGLE_CLIENT_SECRET=your-secret

# ── Optional ──
TAVILY_API_KEY=                  # enables web_search tool
ALLOW_PRIVATE_BASE_URLS=false    # true only for local Ollama/vLLM
```

> **Note:** User LLM API keys (Anthropic / OpenAI / Google) are **not** set here — users paste them into Settings in the web UI. That's the whole BYOK idea.

### 3. Set Up the Database

```bash
npm run db:migrate
```

### 4. Run

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000) — paste your API keys in **Settings**, pick a model (or leave it on **Auto**), and start chatting.

---

## 🔌 OpenAI-Compatible API

OmniRouter exposes a drop-in replacement for the OpenAI Chat API. Any SDK or tool that speaks the OpenAI protocol works with **zero code changes**.

### Generate a Router Key

Go to **Settings → Router API Keys** in the web UI and create a key.

### Usage

```bash
# Chat completion (streaming)
curl https://multi-agent-router-phi.vercel.app/v1/chat/completions \
  -H "Authorization: Bearer omni-your-key-here" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "auto",
    "stream": true,
    "messages": [{ "role": "user", "content": "Explain quicksort in 3 sentences." }]
  }'

# List available models
curl https://multi-agent-router-phi.vercel.app/v1/models \
  -H "Authorization: Bearer omni-your-key-here"
```

### With the OpenAI Python SDK

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://multi-agent-router-phi.vercel.app/v1",
    api_key="omni-your-key-here",
)

response = client.chat.completions.create(
    model="auto",          # or "auto:cheap", "auto:fast", or a specific model
    messages=[{"role": "user", "content": "Hello!"}],
)
print(response.choices[0].message.content)
```

### Routing Policies

| Model Value | Behavior |
|---|---|
| `auto` | Balanced: best quality-cost-latency trade-off |
| `auto:cheap` | Prefer cheaper models (↑ cost weight) |
| `auto:fast` | Prefer faster models (↑ latency weight) |
| `anthropic/claude-sonnet-5-5` | Pin to a specific model (with fallbacks) |

---

## 🛡 Security

| Feature | Implementation |
|---|---|
| **Key Encryption** | AES-256-GCM envelope encryption: per-key DEK → master KEK wrapping. Keys are never stored in plaintext. |
| **SSRF Guard** | Blocks private IPv4/v6 ranges, cloud metadata (169.254.x.x), DNS re-binding, `.internal`/`.local` domains, and validates redirect chains (≤ 5 hops). |
| **Auth** | Google OAuth via Better Auth with encrypted tokens. Guest mode via signed `omni_sid` cookies. |
| **API Keys** | SHA-256 hashed router keys with scoped permissions and per-key spend caps. |
| **Rate Limiting** | Per-identity sliding windows: 10 / 30 / 60 req/min by tier. |
| **Input Validation** | Every API route validated with Zod schemas. |
| **Audit Log** | Every credential change and admin action logged with IP + timestamp. |

---

## 🧪 Testing

```bash
npm test
```

**31 unit tests** across 3 test suites:

| Suite | Tests | Covers |
|---|---|---|
| `cascade.test.ts` | 18 | Happy path, silent failover, mid-stream handoff, fatal errors, context compaction, circuit breaker state machine |
| `vault.test.ts` | 5 | Encrypt/decrypt round-trip, wrong-key rejection, tamper detection |
| `tools.test.ts` | 8 | SSRF guard (private IPs, dangerous URLs), tool execution, schema validation |

---

## 🗄 Database Schema

20 normalized PostgreSQL tables managed by Drizzle ORM with 3 versioned migrations:

| Table Group | Tables | Purpose |
|---|---|---|
| **Identity** | `users`, `sessions`, `accounts`, `verification`, `guest_sessions` | Auth & session management |
| **Tenancy** | `organizations`, `members` | Multi-tenant org support |
| **Vault** | `provider_credentials` | Encrypted BYOK API keys |
| **Chat** | `conversations`, `messages` | Conversation persistence |
| **Telemetry** | `runs`, `provider_health`, `feedback`, `quality_scores` | Per-attempt logging & model quality tracking |
| **API** | `router_api_keys`, `routing_policies` | OpenAI-compatible key management |
| **Billing** | `usage_daily` | Daily cost rollups per provider |
| **Tools** | `tool_invocations` | Tool execution audit trail |
| **Catalog** | `models` | Model metadata & pricing |
| **Audit** | `audit_log` | Security audit trail |

---

## 📡 Supported Models

<table>
<tr><th>Provider</th><th>Models</th></tr>
<tr>
<td><strong>Anthropic</strong></td>
<td>Claude Opus 5.5 · Claude Sonnet 5.5 · Claude Haiku 5.5 · Claude Opus 4.8 · Claude Sonnet 4.6</td>
</tr>
<tr>
<td><strong>OpenAI</strong></td>
<td>GPT-4.1 · GPT-4.1 mini · GPT-4.1 nano · o4-mini</td>
</tr>
<tr>
<td><strong>Google</strong></td>
<td>Gemini 2.5 Pro · Gemini 2.5 Flash · Gemini 2.5 Flash-Lite</td>
</tr>
<tr>
<td><strong>Compatible</strong></td>
<td>Any OpenAI-compatible endpoint (Ollama, LM Studio, vLLM, etc.)</td>
</tr>
</table>

> New models can be added by extending the seed catalog in `packages/providers/src/catalog.ts`.

---

## 🚢 Deployment

### Vercel (Recommended)

1. Push to GitHub
2. Import the repo in [Vercel](https://vercel.com)
3. Set **Root Directory** to `apps/web`
4. Add all environment variables from `.env.example`
5. Deploy 🚀

### Docker (Self-hosted)

```bash
docker run -p 5432:5432 -e POSTGRES_PASSWORD=secret postgres:17
# Then set DATABASE_URL and run the app
npm run db:migrate
npm run build
npm start
```

---

## 🤝 Contributing

1. Fork the repo
2. Create a feature branch (`git checkout -b feat/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feat/amazing-feature`)
5. Open a Pull Request

---

## 📄 License

This project is licensed under the MIT License.

---

<div align="center">

**[🚀 Try the Live Demo](https://multi-agent-router-phi.vercel.app/)**

Built with ❤️ using TypeScript, Next.js, and PostgreSQL

</div>

