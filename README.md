# CodePilot Engineer

> AI-powered codebase assistant — index any Git repository and ask natural language questions about your code.

CodePilot helps software engineers understand unfamiliar codebases instantly. It indexes your repositories using semantic embeddings, then answers questions like *"where is the auth middleware?"*, *"how does the indexing pipeline work?"*, or *"what calls this function?"* — with exact file paths and line numbers cited from your actual code.

<img width="1856" height="888" alt="image" src="https://github.com/user-attachments/assets/b3352456-9f80-4e77-b68c-cbb26cf9c563" />


---

## Table of Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [Architecture Overview](#architecture-overview)
- [Project Structure](#project-structure)
- [How It Works](#how-it-works)
  - [Indexing Pipeline](#indexing-pipeline)
  - [Query Pipeline (RAG)](#query-pipeline-rag)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Installation](#installation)
  - [Environment Variables](#environment-variables)
  - [Running with Docker](#running-with-docker)
  - [Running without Docker](#running-without-docker)
- [API Reference](#api-reference)
  - [Authentication](#authentication-endpoints)
  - [Repositories](#repository-endpoints)
  - [Queries & Conversations](#query-endpoints)
  - [Webhooks](#webhook-endpoints)
- [WebSocket Events](#websocket-events)
- [Database Schema](#database-schema)
- [Subscription Plans](#subscription-plans)
- [Deployment](#deployment)
- [Available Scripts](#available-scripts)
- [Contributing](#contributing)
- [License](#license)

---

## Features

- **Natural language Q&A** over any indexed codebase
- **Semantic search** — finds relevant code by meaning, not keyword matching
- **Streaming responses** — answers appear word-by-word like ChatGPT
- **Exact code references** — every answer cites `file:line` locations
- **Real-time indexing progress** — live status via WebSocket
- **Multi-repository support** — query across multiple repos
- **Conversation history** — persistent chat sessions with full context
- **Multi-tenant** — organization-scoped data isolation
- **GitHub OAuth** — one-click login
- **Subscription billing** — Stripe-powered plans
- **12 programming languages** — TypeScript, JavaScript, Python, Go, Rust, Java, C++, C, Ruby, PHP, Swift, Kotlin

---

## Tech Stack

### Frontend
| Technology | Purpose |
|-----------|---------|
| React 18 | UI framework |
| Vite | Build tool & dev server |
| React Router | Client-side routing (SPA) |
| TailwindCSS | Utility-first styling |
| Socket.io Client | Real-time WebSocket communication |
| React Query (@tanstack) | Server state management & caching |
| Zustand | Client-side local UI state |
| React Markdown | Render LLM responses with syntax highlighting |
| Lucide Icons | Icon library |
| Sonner | Toast notifications |

### Backend (API)
| Technology | Purpose |
|-----------|---------|
| Node.js + Express | REST API server |
| Socket.io | WebSocket server for real-time streaming |
| MongoDB + Mongoose | Primary database (users, orgs, repos, conversations) |
| Redis (IORedis) | Job queue backend |
| BullMQ | Job queue client — dispatch indexing jobs |
| JWT + bcryptjs | Stateless auth + password hashing |
| GitHub OAuth | Social login |
| Groq API (Llama-3.3-70b) | LLM for generating answers |
| Pinecone | Vector database for semantic search |
| FlagEmbed (BGE-base-en-v1.5) | Local embedding model (768 dimensions) |
| Stripe | Subscription billing |
| Resend | Transactional email |
| Helmet + express-rate-limit | HTTP security |
| Zod + express-validator | Input validation |
| Winston | Structured logging |

### Worker (Background Jobs)
| Technology | Purpose |
|-----------|---------|
| BullMQ | Job processor — picks up and runs queued jobs |
| Simple-git | Clone Git repositories to disk |
| Tree-sitter | Parse source code into AST |
| FlagEmbed (BGE-base-en-v1.5) | Generate 768-dim local embeddings |
| Pinecone SDK | Upsert vectors to vector database |
| Tiktoken | Count tokens before LLM calls |

### Shared Package
| Technology | Purpose |
|-----------|---------|
| TypeScript interfaces | Shared types across all packages |
| Shared constants | Plans, supported languages, queue names, limits |

### Infrastructure
| Technology | Purpose |
|-----------|---------|
| pnpm workspaces | Monorepo package management |
| TypeScript 5.4+ | Full type safety |
| Docker + Docker Compose | Local development environment |
| Render | Cloud API server deployment |
| Netlify | Frontend static site deployment |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                     Browser (React)                  │
│         React Query · Zustand · Socket.io Client     │
└────────────────────────┬────────────────────────────┘
                         │ HTTP / WebSocket
┌────────────────────────▼────────────────────────────┐
│               API Server (Express + Socket.io)       │
│   Auth · Routes · LLM Streaming · Vector Search      │
│         Groq (Llama-3.3-70b) · Pinecone · JWT        │
└──────┬─────────────────────────────┬────────────────┘
       │ BullMQ Jobs                 │ Mongoose
┌──────▼──────────┐        ┌─────────▼──────────┐
│  Worker Service  │        │      MongoDB        │
│  Clone → Parse  │        │  Users · Orgs ·     │
│  Chunk → Embed  │        │  Repos · Convos     │
│  FlagEmbed BGE  │        └────────────────────┘
│  → Pinecone     │
└──────────────────┘
       │ Redis
┌──────▼──────────┐        ┌────────────────────┐
│      Redis       │        │      Pinecone       │
│  BullMQ Queues  │        │  768-dim Vectors    │
│  (indexing,     │        │  Code Chunk Metadata│
│   embedding)    │        └────────────────────┘
└──────────────────┘
```

---

## Project Structure

```
CodePilotEngineer/
├── packages/
│   ├── api/                          # Express API server
│   │   ├── src/
│   │   │   ├── index.ts              # Server entry point
│   │   │   ├── routes/
│   │   │   │   ├── auth.ts           # Login, register, GitHub OAuth
│   │   │   │   ├── repositories.ts   # Repo CRUD + indexing triggers
│   │   │   │   ├── queries.ts        # Query submission (SSE streaming)
│   │   │   │   ├── organizations.ts  # Org management
│   │   │   │   └── webhooks.ts       # Stripe + GitHub webhooks
│   │   │   ├── services/
│   │   │   │   ├── claude.ts         # Groq LLM streaming (Llama-3.3-70b)
│   │   │   │   └── vectorSearch.ts   # Pinecone search + FlagEmbed
│   │   │   ├── models/
│   │   │   │   ├── User.ts
│   │   │   │   ├── Organization.ts
│   │   │   │   ├── Repository.ts
│   │   │   │   └── Conversation.ts
│   │   │   ├── sockets/
│   │   │   │   └── handlers.ts       # WebSocket query processing
│   │   │   ├── middleware/
│   │   │   │   └── auth.ts           # JWT verification middleware
│   │   │   └── config/
│   │   ├── Dockerfile
│   │   └── package.json
│   │
│   ├── worker/                       # Background job processor
│   │   ├── src/
│   │   │   ├── index.ts              # Worker entry point
│   │   │   └── processors/
│   │   │       ├── indexingProcessor.ts    # Clone → parse → chunk
│   │   │       ├── embeddingProcessor.ts   # Embed → upsert to Pinecone
│   │   │       └── architectureExtractor.ts
│   │   ├── Dockerfile
│   │   └── package.json
│   │
│   ├── frontend/                     # React web app
│   │   ├── src/
│   │   │   ├── main.tsx              # React entry point
│   │   │   ├── App.tsx               # Route definitions
│   │   │   ├── pages/
│   │   │   │   ├── ChatPage.tsx      # Main Q&A interface
│   │   │   │   ├── RepositoriesPage.tsx
│   │   │   │   ├── DashboardPage.tsx
│   │   │   │   ├── LoginPage.tsx
│   │   │   │   └── HomePage.tsx
│   │   │   ├── components/
│   │   │   │   ├── AppLayout.tsx     # Main layout + sidebar
│   │   │   │   ├── CommandPalette.tsx
│   │   │   │   └── SEO.tsx
│   │   │   └── lib/
│   │   │       ├── config.ts         # API_URL config
│   │   │       └── utils.ts
│   │   ├── Dockerfile
│   │   ├── nginx.conf
│   │   └── package.json
│   │
│   └── shared/                       # Shared types & constants
│       ├── src/
│       │   ├── types/index.ts        # All TypeScript interfaces
│       │   └── constants/index.ts    # Plans, languages, queue names
│       └── package.json
│
├── docker-compose.yml                # Local dev environment
├── docker-compose.prod.yml           # Production Docker config
├── render.yaml                       # Render deployment config
├── netlify.toml                      # Netlify frontend config
├── pnpm-workspace.yaml               # pnpm workspace definition
├── tsconfig.json                     # Base TypeScript config
├── vite.config.ts                    # Vite bundler config
└── .env                              # Environment variables (never commit)
```

---

## How It Works

### Indexing Pipeline

When a user adds a repository, this flow runs asynchronously in the background:

```
1. User submits repo URL  →  POST /repositories
2. API creates Repository doc in MongoDB (status: "pending")
3. API enqueues INDEXING job in BullMQ (via Redis)
4. Worker picks up job:
     a. Clone repo to disk  (simple-git)
     b. Walk all files, skip: node_modules / .git / dist / build
     c. Detect language from file extension
     d. Parse each file with Tree-sitter (AST-level understanding)
     e. Chunk code into 60-line segments with 10-line overlap
     f. Extract metadata: imports, exports, function names, class names
     g. Enqueue EMBEDDING jobs (batches of ~200 chunks)
5. Worker picks up EMBEDDING jobs:
     a. Load code chunk
     b. Run through FlagEmbed BGE-base-en-v1.5 (LOCAL — no API call)
     c. Produce 768-dimensional float vector
     d. Upsert to Pinecone with metadata:
           { repositoryId, filePath, startLine, endLine, language, content }
        Vector ID = SHA1(repositoryId:filePath:startLine)
6. Repository.indexingStatus → "completed" in MongoDB
7. Socket.io emits progress events → Frontend updates live status bar
```

### Query Pipeline (RAG)

Retrieval-Augmented Generation — how answers are generated:

```
1. User asks: "How does the auth middleware work?"
2. Frontend  →  Socket.io "query" event  →  API
3. API embeds the question:
     FlagEmbed BGE-base → 768-dim vector
4. API queries Pinecone:
     - Cosine similarity search
     - Filter by repositoryId
     - Return top 8 most relevant code chunks
5. API builds LLM prompt:
     System:  CodePilot rules + code chunks as RAG context
     History: Last 3 messages
     User:    "How does the auth middleware work?"
6. API calls Groq (llama-3.3-70b-versatile), streaming=true
7. Each token is emitted back via Socket.io as it arrives
8. Full conversation saved to MongoDB with references
9. Frontend renders streaming markdown + syntax-highlighted code blocks
```

---

## Getting Started

### Prerequisites

- **Node.js** >= 20.0.0
- **pnpm** >= 8.0.0
- **Docker & Docker Compose** (for local infrastructure)
- Accounts / API keys:
  - [Groq](https://console.groq.com) — free LLM API
  - [Pinecone](https://www.pinecone.io) — free vector database
  - [GitHub OAuth App](https://github.com/settings/developers) — for social login
  - [Stripe](https://stripe.com) — for billing (optional for development)

### Installation

```bash
# 1. Clone the repository
git clone https://github.com/your-org/CodePilotEngineer.git
cd CodePilotEngineer

# 2. Install all dependencies across all packages
pnpm install

# 3. Set up environment variables
cp .env .env.local
# Edit .env with your actual API keys
```

### Environment Variables

Create a `.env` file in the root. All variables:

```bash
# ── Node ──────────────────────────────────────────────────────
NODE_ENV=development

# ── MongoDB ───────────────────────────────────────────────────
MONGODB_URI=mongodb://localhost:27017/codepilot
# Or MongoDB Atlas:
# MONGODB_URI=mongodb+srv://<user>:<pass>@cluster.mongodb.net/codepilot

# ── Redis (BullMQ) ────────────────────────────────────────────
REDIS_URL=redis://localhost:6379

# ── API ───────────────────────────────────────────────────────
API_PORT=3001
CORS_ORIGIN=http://localhost:3000
LOG_LEVEL=info

# ── JWT ───────────────────────────────────────────────────────
JWT_SECRET=your_secret_here          # openssl rand -hex 64
JWT_EXPIRES_IN=7d

# ── GitHub OAuth ──────────────────────────────────────────────
# Create at: https://github.com/settings/developers
GITHUB_CLIENT_ID=your_client_id
GITHUB_CLIENT_SECRET=your_client_secret
GITHUB_CALLBACK_URL=http://localhost:3001/auth/github/callback

# ── Groq (LLM) ────────────────────────────────────────────────
# Get key at: https://console.groq.com
GROQ_API_KEY=your_groq_key

# ── Pinecone (Vector DB) ──────────────────────────────────────
# Get key at: https://www.pinecone.io
PINECONE_API_KEY=your_pinecone_key
PINECONE_INDEX_NAME=codepilot-embeddings

# ── Stripe (Billing) ──────────────────────────────────────────
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...

# ── Resend (Email) ────────────────────────────────────────────
RESEND_API_KEY=re_...
FROM_EMAIL=noreply@yourdomain.com

# ── Frontend (Vite) ───────────────────────────────────────────
VITE_API_URL=http://localhost:3001
VITE_WS_URL=http://localhost:3001
```

### Running with Docker

```bash
# Start all services (MongoDB, Redis, API, Worker, Frontend)
pnpm docker:up

# View logs
pnpm docker:logs

# Stop all services
pnpm docker:down
```

Services started:

| Service | Port | Description |
|---------|------|-------------|
| Frontend | 3000 | React web app |
| API | 3001 | Express + Socket.io |
| MongoDB | 27017 | Primary database |
| Redis | 6379 | Job queue backend |
| Worker | — | Background job processor |

### Running without Docker

```bash
# Start MongoDB and Redis first (required)
mongod --fork --logpath /var/log/mongod.log
redis-server --daemonize yes

# Start all packages in development mode (hot reload)
pnpm dev

# Or start individually:
pnpm dev:api        # API server on :3001
pnpm dev:worker     # Background worker
pnpm dev:frontend   # Frontend on :3000
```

Open [http://localhost:3000](http://localhost:3000)

---

## API Reference

All authenticated endpoints require:
```
Authorization: Bearer <jwt_token>
```

### Authentication Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/auth/register` | Register with email + password | No |
| POST | `/auth/login` | Login with email + password | No |
| GET | `/auth/github` | Redirect to GitHub OAuth | No |
| GET | `/auth/github/callback` | GitHub OAuth callback | No |
| POST | `/auth/logout` | Logout (clear session) | Yes |
| GET | `/auth/me` | Get current user info | Yes |

**Register / Login payload:**
```json
{
  "email": "user@example.com",
  "password": "securepassword",
  "name": "Jane Doe"
}
```

**Response:**
```json
{
  "token": "<jwt>",
  "user": { "_id": "...", "email": "...", "name": "...", "role": "admin" }
}
```

### Repository Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/repositories` | List all repos for org | Yes |
| POST | `/repositories` | Add a new repository | Yes |
| GET | `/repositories/:id` | Get repository details | Yes |
| DELETE | `/repositories/:id` | Remove repository | Yes |
| POST | `/repositories/:id/index` | Trigger re-indexing | Yes |
| GET | `/repositories/:id/status` | Get indexing status | Yes |

**Add repository payload:**
```json
{
  "name": "my-backend",
  "url": "https://github.com/org/my-backend",
  "branch": "main",
  "provider": "github"
}
```

**Repository object:**
```json
{
  "_id": "...",
  "name": "my-backend",
  "url": "https://github.com/org/my-backend",
  "branch": "main",
  "provider": "github",
  "indexingStatus": "completed",
  "fileCount": 142,
  "organizationId": "...",
  "createdAt": "2024-01-01T00:00:00Z"
}
```

`indexingStatus` values: `pending` | `in_progress` | `completed` | `failed`

### Query Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/queries/conversations` | List all conversations | Yes |
| GET | `/queries/conversations/:id` | Get conversation with messages | Yes |
| DELETE | `/queries/conversations/:id` | Delete conversation | Yes |

Queries are submitted via **WebSocket** (see below), not HTTP POST.

### Webhook Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/webhooks/stripe` | Stripe subscription events |
| POST | `/webhooks/github` | GitHub push / PR events |

### Health Check

```
GET /health
→ 200 { "status": "ok", "timestamp": "..." }
```

---

## WebSocket Events

Connect to the API server via Socket.io. Pass the JWT on connection:

```javascript
const socket = io('http://localhost:3001', {
  auth: { token: localStorage.getItem('cp_token') }
});
```

### Client → Server

**Submit a query:**
```javascript
socket.emit('query', {
  query: 'How does the auth middleware work?',
  repositoryId: '64abc...',       // optional — scopes to one repo
  conversationId: '64def...'      // optional — continues existing chat
});
```

### Server → Client

**Streaming answer token:**
```javascript
socket.on('response:chunk', (data) => {
  // data.content — one token of the streaming response
});
```

**Answer complete:**
```javascript
socket.on('response:done', (data) => {
  // data.conversationId  — ID of the conversation
  // data.references      — array of CodeReference objects
});
```

**Indexing progress:**
```javascript
socket.on('indexing:progress', (data) => {
  // data.repositoryId
  // data.status          — "cloning" | "parsing" | "embedding" | "completed"
  // data.progress        — 0–100
  // data.filesProcessed
  // data.totalFiles
});
```

**Error:**
```javascript
socket.on('error', (data) => {
  // data.message — error description
});
```

**CodeReference object:**
```typescript
{
  filePath: string;     // e.g. "src/middleware/auth.ts"
  startLine: number;    // e.g. 12
  endLine: number;      // e.g. 45
  content: string;      // actual code snippet
  score?: number;       // cosine similarity score (0–1)
}
```

---

## Database Schema

### Users Collection
```typescript
{
  _id: ObjectId,
  email: string,           // unique
  password: string,        // bcrypt hashed
  name: string,
  githubId: string,        // from OAuth
  role: 'admin' | 'member' | 'viewer',
  organizationId: ObjectId,
  createdAt: Date,
  updatedAt: Date
}
```

### Organizations Collection
```typescript
{
  _id: ObjectId,
  name: string,
  slug: string,            // unique URL-safe identifier
  plan: 'free' | 'starter' | 'team' | 'enterprise',
  stripeCustomerId: string,
  stripeSubscriptionId: string,
  createdAt: Date
}
```

### Repositories Collection
```typescript
{
  _id: ObjectId,
  organizationId: ObjectId,
  name: string,
  url: string,
  provider: 'github' | 'gitlab' | 'bitbucket',
  branch: string,
  indexingStatus: 'pending' | 'in_progress' | 'completed' | 'failed',
  fileCount: number,
  archMetadata: {           // extracted architecture info
    services: any[],
    edges: any[],
    adrs: any[]
  },
  createdAt: Date,
  updatedAt: Date
}
```

### Conversations Collection
```typescript
{
  _id: ObjectId,
  userId: ObjectId,
  organizationId: ObjectId,
  repositoryId: ObjectId,
  title: string,
  messages: [
    {
      role: 'user' | 'assistant',
      content: string,
      references: CodeReference[],
      timestamp: Date
    }
  ],
  createdAt: Date,
  updatedAt: Date
}
```

### Pinecone Vector Schema
```typescript
{
  id: string,              // SHA1(repositoryId:filePath:startLine)
  values: number[],        // 768-dimensional float vector (BGE-base)
  metadata: {
    repositoryId: string,
    filePath: string,
    startLine: number,
    endLine: number,
    language: string,
    chunkType: string,
    content: string        // truncated to 4000 chars
  }
}
```

---

## Subscription Plans

| Plan | Price/mo | Repositories | Members | Queries/month |
|------|----------|-------------|---------|---------------|
| Free | $0 | 1 | 3 | 100 |
| Starter | $49 | 5 | 10 | 1,000 |
| Team | $199 | 20 | 50 | 10,000 |
| Enterprise | $999 | Unlimited | Unlimited | Unlimited |

---

## Deployment

### Frontend — Netlify

The `netlify.toml` is configured for SPA deployment. All routes redirect to `index.html`.

```bash
# Netlify auto-deploys from main branch
# Build command: pnpm build:frontend
# Publish dir:   packages/frontend/dist
```

Set environment variable in Netlify dashboard:
```
VITE_API_URL=https://your-api.onrender.com
```

### API + Worker — Render

The `render.yaml` defines both the API and Worker services.

```bash
# Render auto-deploys from main branch
# Build command: pnpm build:api && pnpm build:worker
# Start command: pnpm start:api
```

### Self-hosted — Docker Compose

```bash
# Production
docker-compose -f docker-compose.prod.yml up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f api
docker-compose logs -f worker
```

---

## Available Scripts

### Root (runs across all packages)

```bash
pnpm dev              # Start all packages in dev mode (hot reload)
pnpm build            # Build all packages
pnpm test             # Run tests across all packages
pnpm lint             # Lint all packages
pnpm type-check       # TypeScript type check all packages
pnpm clean            # Delete all dist/ and node_modules/
```

### Docker

```bash
pnpm docker:up        # Start all services
pnpm docker:down      # Stop all services
pnpm docker:logs      # Tail all service logs
```

### Per-package

```bash
pnpm dev:api          # Start API server (port 3001)
pnpm dev:worker       # Start background worker
pnpm dev:frontend     # Start frontend dev server (port 3000)

pnpm build:api
pnpm build:worker
pnpm build:frontend
```

---

## Supported Languages

| Language | Extensions |
|----------|-----------|
| TypeScript | `.ts`, `.tsx` |
| JavaScript | `.js`, `.jsx`, `.mjs`, `.cjs` |
| Python | `.py` |
| Go | `.go` |
| Rust | `.rs` |
| Java | `.java` |
| C++ | `.cpp`, `.cc`, `.cxx`, `.hpp` |
| C | `.c`, `.h` |
| Ruby | `.rb` |
| PHP | `.php` |
| Swift | `.swift` |
| Kotlin | `.kt` |

---

## Contributing

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/your-feature`
3. Make your changes
4. Run type check and lint: `pnpm type-check && pnpm lint`
5. Commit: `git commit -m "feat: your feature description"`
6. Push: `git push origin feature/your-feature`
7. Open a Pull Request against `main`

---

## License

MIT License — see [LICENSE](LICENSE) for details.
