# template-mastra-base

A production-ready Mastra agent starter. One example agent, full eval pipeline, Docker, CI — everything you need to ship a Mastra agent without building the scaffold yourself.

---

## Quickstart (5 minutes)

**Prerequisites**: Node 22+, Docker Desktop, a Supabase project, an Anthropic API key.

```bash
# 1. Clone and install
git clone <repo> my-agent && cd my-agent
npm install

# 2. Configure environment
cp .env.example .env
# Fill in: APP_SECRET, SUPABASE_*, ANTHROPIC_API_KEY

# 3. Start local Supabase (first time only)
npx supabase start

# 4. Run
npm run dev
# → Mastra Studio at http://localhost:4111
```

Chat with the `leadIntake` agent in Studio to verify everything works. Send:

> Hi, this is John Smith from Acme Corp (john@acme.io). We need pricing for 50 seats by Friday.

Expected: structured JSON output with all fields populated.

---

## File Structure

```
template-mastra-base/
├── src/
│   ├── lib/
│   │   └── env.ts                  # Zod-validated env loader — crashes on bad config
│   └── mastra/
│       ├── index.ts                # Entry point: env → AIMock → Mastra instance
│       ├── agents/
│       │   └── _example.ts         # leadIntake agent — copy this for new agents
│       ├── lib/
│       │   ├── aimock.ts           # Routes LLM calls to AIMock when USE_AIMOCK=true
│       │   └── supabase.ts         # Supabase client factory (anon / service / user-scoped)
│       ├── scorers/
│       │   ├── _example.scorers.ts # hallucination + completeness + urgency scorers
│       │   └── datasets/
│       │       └── _example.json   # Eval dataset — 5 cases with thresholds
│       ├── tools/                  # Shared tools (inline tools live in agent files)
│       └── workflows/              # Mastra workflows
├── scripts/
│   └── eval.ts                     # Offline CI eval gate — exits 0/1 based on thresholds
├── prompts/
│   ├── README.md                   # Index of agent-building prompts
│   └── build-agent.md              # Parameterized prompt for adding a new agent
├── .github/
│   └── workflows/
│       └── ci.yml                  # typecheck → build + eval (parallel) → docker
├── Dockerfile                      # Multi-stage, node:22-slim runtime
├── docker-compose.yml              # Production compose
├── compose.dev.yml                 # Dev compose override
├── .env.example                    # All required env vars with comments
└── AGENTS.md                       # Conventions for AI coding agents
```

---

## Scripts

| Command | What it does |
|---|---|
| `npm run dev` | Start Mastra Studio at localhost:4111 |
| `npm run build` | Bundle for production (output → `.mastra/output/`) |
| `npm run start` | Start production server (no Studio) |
| `npm run eval` | Run offline eval gate against all cases in the dataset |
| `npm run typecheck` | TypeScript type check (zero-emit) |
| `npm run score:list` | List registered scorers |

---

## Adding a New Agent

1. Copy `src/mastra/agents/_example.ts` → `src/mastra/agents/my-agent.ts`
2. Rename the agent, update `id`, `instructions`, `model`, tools, and output schema
3. Register it in `src/mastra/index.ts` under `agents:`
4. Add eval cases to a new dataset file in `src/mastra/scorers/datasets/`
5. Use `prompts/build-agent.md` with Claude Code to generate a complete agent from a description

---

## Running Evals

```bash
# Against live Anthropic API
npm run eval

# Against AIMock (deterministic, no API cost)
npx @copilotkit/aimock --config aimock.json &
USE_AIMOCK=true npm run eval

# Custom dataset
node --env-file=.env --import tsx/esm scripts/eval.ts path/to/dataset.json
```

---

## Docker

```bash
# Build
docker build -t my-agent:latest .

# Run
docker compose up -d

# Health check
curl http://localhost:4111/health
```

> **Local Supabase note**: Docker containers can't reach `127.0.0.1` on the host. Set `SUPABASE_DB_URL` to use `host.docker.internal` instead when running via Docker Desktop locally.

---

## Deployment Notes

### Docker image size

The production image is ~676MB. This is larger than typical Node.js Docker images because:

- The base is `node:22-slim` (Debian, glibc) instead of `node:22-alpine` (musl)
- DuckDB ships native binaries that segfault on musl libc, even with `gcompat`
- DuckDB is required by `@mastra/observability` for trace storage

If you need a smaller image, the path is to swap DuckDB for `LibSQLStore` in the observability domain (see `src/mastra/index.ts`). Trade-off: slower trace queries in Mastra Studio, especially under load.

For typical VPS deployments (Hetzner, DigitalOcean, etc.) the 676MB size is not a problem — pulls take seconds and storage is cheap. Only optimize if you're targeting Lambda, Cloud Run cold starts, or memory-constrained environments under 1GB.

---

## Common Gotchas

| Symptom | Cause | Fix |
|---|---|---|
| `Invalid environment variables` on boot | Missing or malformed `.env` | Check each var listed in the error against `.env.example` |
| `ECONNREFUSED 127.0.0.1:54322` | Local Supabase not running | `npx supabase start` |
| Docker container crashes (SIGSEGV) | DuckDB requires glibc, not musl | Use `node:22-slim`, not `node:22-alpine` — see Deployment Notes |
| `ECONNREFUSED` inside Docker | `127.0.0.1` in DB URL | Replace with `host.docker.internal` |
| Agent not listed in Studio | Not registered in `mastra.agents` | Add to `src/mastra/index.ts` |
| Storage init error about missing `id` | `PostgresStore`/`LibSQLStore` requires `id` field | Pass `id: 'mastra-storage'` to the constructor |
| PostHog telemetry noise in restricted networks | Mastra runtime phones home on startup | Set `MASTRA_TELEMETRY_DISABLED=1` in `.env` |
| DB connection errors at scale | Direct Supabase connection has limited slots | Use the **session pooler** URL from Supabase dashboard (Project Settings → Database → Connection string → Session pooler) |
| Pino transport error in Docker | `pino-pretty` missing from production deps | Ensure it's in `dependencies`, not `devDependencies`, in any packages you add |

---

## Environment Variables

See `.env.example` for the full list with comments. Minimum required:

- `APP_SECRET` — min 32 chars, generate with `openssl rand -hex 32`
- `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `SUPABASE_DB_URL`
- At least one of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GOOGLE_GENERATIVE_AI_API_KEY`

---

## For AI Coding Agents

See `AGENTS.md` for conventions, boot order, import rules, and things to never do.
