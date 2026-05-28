# System Context — AI Hedge Fund Build

Paste this at the start of every Claude Code session. It contains everything you need to know about this project.

---

## What I Am Building

An AI-native research and operations system for a small hedge fund (1–3 employees). The system uses AI agents to automate daily analyst tasks: stock research, SEC filing monitoring, daily screening, earnings preparation, sentiment monitoring, and portfolio-level synthesis. All outputs are written to a Postgres database. Notion receives lightweight notifications only.

This is an internal tool — not a public-facing product. There are no external users. The system runs on scheduled agents and is accessed by a small internal team through Claude and Notion.

---

## The Most Important Architectural Rule

**Postgres is always the source of truth. Notion is always downstream.**

Every agent follows this pattern without exception:
1. Run analysis
2. Save full structured output to Neon Postgres
3. Send a one-line summary notification to Notion

Never save full content to Notion. Never treat Notion as a database. This architecture exists so that a custom web interface can be added later by simply reading from Postgres — with no migration, no rewriting of agents, and no data loss.

---

## Full Tech Stack

### Infrastructure
| Tool | Purpose |
|---|---|
| GitHub | Code versioning — private repo called `hedge-fund-ai` |
| Railway | Compute — runs all Python agents, cron jobs, background workers |
| Neon Postgres | Primary database — all structured agent outputs |
| Backblaze B2 | Object storage — raw files (PDFs, transcripts, CSVs) |
| Turbopuffer | Vector storage — semantic search across research and filings |
| DuckDB | Analytics — screener queries, financial aggregations (runs locally and on Railway) |
| Sentry | Error monitoring — alerts when agents fail |
| Prisma | ORM — all database interactions use Prisma, never raw SQL |

### Data Sources
| Tool | Purpose |
|---|---|
| financialdatasets.ai | Financials, prices, SEC filings — has native MCP |
| Quartr | Earnings call transcripts and calendar |
| EDGAR | SEC filings direct — free public API |
| NewsAPI | News feed for agent research |
| X API | Social sentiment monitoring — Basic tier |

### AI
| Tool | Purpose |
|---|---|
| Anthropic API | Powers all agents — use Haiku for routine/high-volume, Sonnet for research and analysis |
| Claude Code | Writes all code for this project |
| OpenAI Embeddings | text-embedding-3-small — chosen embedding model, must never change |

### Collaboration
| Tool | Purpose |
|---|---|
| Notion | Notification layer and team workspace — not a database |

### Future Additions (not built yet)
- Next.js + shadcn/ui on Vercel — custom web frontend reading from Postgres
- Better Auth + Google OAuth — team authentication
- Resend — transactional email for agent alerts and digests

---

## Naming Conventions

### GitHub Repo
`hedge-fund-ai`

### Backblaze B2 Buckets
- `hf-raw-filings` — SEC filings and regulatory documents
- `hf-transcripts` — earnings call transcripts and audio
- `hf-screener-outputs` — daily screener CSV exports and results

### Neon Postgres Tables
| Table | Purpose |
|---|---|
| `stocks` | Master stock universe and watchlist |
| `research_notes` | Full research memos written by agents |
| `filings` | SEC filing metadata and processing status |
| `screener_results` | Daily screener outputs and signals |
| `watchlist` | Active coverage list with priority and status |
| `earnings_briefs` | Pre-earnings call preparation memos |
| `sentiment_alerts` | X and NewsAPI sentiment flags |
| `portfolio_reviews` | Weekly portfolio synthesis memos |
| `orchestrator_logs` | Daily agent run logs and status |

### Turbopuffer Namespaces
- `research-notes` — embeddings of research memos and analyst notes
- `filings` — embeddings of SEC filing text
- `earnings-transcripts` — embeddings of earnings call transcripts

### Embedding Model
**text-embedding-3-small** (OpenAI) — this is permanent. Never switch models. All vectors in Turbopuffer use this model and switching would require re-embedding everything.

### Python Agent Files
| File | Purpose |
|---|---|
| `research_agent.py` | Single-stock research — pulls EDGAR, financials, news, writes memo to Postgres |
| `filing_watcher.py` | Monitors EDGAR for new filings from watchlist |
| `screener.py` | Daily DuckDB screener across full stock universe |
| `earnings_prep.py` | Pre-earnings brief — triggers 24 hours before each call |
| `sentiment_monitor.py` | X and NewsAPI sentiment monitoring — runs every 4 hours |
| `portfolio_agent.py` | Weekly portfolio synthesis — runs every Monday |
| `orchestrator.py` | Coordinates all agents, runs daily morning sequence |
| `seed_vector_store.py` | Ingests documents into Turbopuffer |
| `send_email.py` | Resend email utility (future) |

---

## Environment Variables

All secrets live in Railway's Variables tab. Never hardcode any key in any file. Read all keys using `os.environ.get('KEY_NAME')`.

| Variable Name | What It Is |
|---|---|
| `DATABASE_URL` | Neon Postgres connection string |
| `ANTHROPIC_API_KEY` | Anthropic API key |
| `OPENAI_API_KEY` | OpenAI API key for embeddings |
| `TURBOPUFFER_API_KEY` | Turbopuffer API key |
| `BACKBLAZE_KEY_ID` | Backblaze master key ID |
| `BACKBLAZE_APPLICATION_KEY` | Backblaze master application key |
| `BACKBLAZE_FILINGS_KEY_ID` | Scoped key for hf-raw-filings bucket |
| `BACKBLAZE_FILINGS_APP_KEY` | Scoped key for hf-raw-filings bucket |
| `BACKBLAZE_TRANSCRIPTS_KEY_ID` | Scoped key for hf-transcripts bucket |
| `BACKBLAZE_TRANSCRIPTS_APP_KEY` | Scoped key for hf-transcripts bucket |
| `BACKBLAZE_SCREENER_KEY_ID` | Scoped key for hf-screener-outputs bucket |
| `BACKBLAZE_SCREENER_APP_KEY` | Scoped key for hf-screener-outputs bucket |
| `FINANCIAL_DATASETS_API_KEY` | financialdatasets.ai API key |
| `QUARTR_API_KEY` | Quartr API key |
| `NEWS_API_KEY` | NewsAPI key |
| `TWITTER_BEARER_TOKEN` | X API Bearer Token |
| `SENTRY_DSN` | Sentry error monitoring DSN |
| `EMBEDDING_MODEL` | text-embedding-3-small |

---

## MCP Servers

| MCP | Status | Purpose |
|---|---|---|
| Notion MCP | Connect via Claude settings | Read and write Notion workspace |
| financialdatasets MCP | Connect via Claude settings | Live financial data queries |
| Railway MCP | Build and deploy — `railway_mcp.py` | Query and write to Postgres |
| EDGAR MCP | Build and deploy — `edgar_mcp.py` | Fetch SEC filings |
| Quartr MCP | Build and deploy — `quartr_mcp.py` | Fetch transcripts and earnings calendar |

---

## Agent Cron Schedule (Railway)

| Agent | Schedule | Time |
|---|---|---|
| Orchestrator (runs all) | `0 7 * * 1-5` | 7:00am weekdays |
| Sentiment monitor | `0 */4 * * 1-5` | Every 4 hours weekdays |
| Portfolio agent | `0 8 * * 1` | 8:00am every Monday |

The orchestrator runs agents in this order with error handling:
1. `filing_watcher.py` — 7:00am
2. Wait 5 minutes
3. `screener.py` — 7:05am
4. `earnings_prep.py` — if earnings tomorrow
5. Log summary to `orchestrator_logs` table
6. Send one-line status to Notion Daily Log page

---

## Notion Structure

Notion receives notifications only. These are the pages and databases agents write to:

| Notion Item | What Agents Write |
|---|---|
| Research Memos database | One-line ping: ticker, date, conviction level |
| Screener Results database | Flagged tickers, signal name, date |
| Earnings Calendar database | Marks Prep Done checkbox when brief is ready |
| Screener Results database | Sentiment alert summary when triggered |
| Daily Log page | One-line orchestrator status each morning |
| Weekly Portfolio Review page | One-paragraph portfolio summary each Monday |

---

## Key Architectural Decisions and Why

**Neon over Railway Postgres** — scale-to-zero billing (pay nothing when agents are idle), database branching for safe schema testing, cheaper storage at $0.35/GB.

**Prisma over raw SQL** — schema lives in one file, migrations are tracked and reversible, safer to evolve over time.

**Backblaze over AWS S3** — 74% cheaper storage, free egress up to 3x stored amount, S3-compatible API so switching later requires minimal code changes.

**Turbopuffer over Pinecone** — cheaper at small scale, built on object storage at $0.02/GB, purpose-built for AI agent workloads.

**Railway over Vercel** — Vercel cannot run persistent Python agents, long-running cron jobs, or background workers. Railway handles all of these natively.

**text-embedding-3-small over alternatives** — industry standard, cheap at $0.02 per million tokens, well-supported, permanent choice that cannot be changed without re-embedding everything.

**Notion as notification layer only** — preserves the option to add a Next.js web frontend later by simply pointing it at the existing Postgres database. No migration required.

---

## What Is Not Built Yet

These are planned for future phases. Do not build them unless explicitly asked:

- Custom Next.js web frontend
- Better Auth + Google OAuth
- Resend email integration
- Personal document ingestion pipeline (Word, PDF, Excel from local folders)
- OCR pipeline for scanned PDFs

---

## Current Build Phase

Check with the user at the start of each session which step they are currently on before doing anything. The full build plan has 38 steps across 6 phases. Work on exactly one step per session unless the user explicitly asks to combine steps.

---

## Security Rules

1. Never hardcode any API key, secret, or credential in any file
2. All secrets use `os.environ.get('VARIABLE_NAME')` 
3. Never commit a `.env` file to GitHub
4. All Backblaze buckets are Private — never Public
5. If writing any file that could contain a secret, flag it and ask before proceeding
6. GitHub secret scanning is enabled on the repo
