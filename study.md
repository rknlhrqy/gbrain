# GBrain — Study Notes

> Synthesised from README, AGENTS, INSTALL_FOR_AGENTS, SECURITY, CLAUDE,
> CONTRIBUTING, llms.txt, and every document under [`docs/`](docs/) — including
> architecture, designs, ethos, guides, integrations, MCP setup, and eval docs.

---

## 1. What GBrain Is

GBrain is an **open-source (MIT) personal knowledge brain for AI agents**. It
ships as a TypeScript CLI and a Model Context Protocol (MCP) server in one Bun
binary. The author (Garry Tan, CEO of Y Combinator) uses it 16 hours a day to
drive his personal OpenClaw / Hermes agents.

The headline claim:

> Your AI agent is smart but forgetful. GBrain gives it a brain.

Concretely, GBrain is:

- A **local-first knowledge store**. Pages are markdown with YAML frontmatter
  on disk. A **derived database** (PGLite for zero-config, or Postgres +
  pgvector for scale) holds the chunks, embeddings, links, and timelines.
  Markdown is the **system of record**; the DB is rebuildable.
- A **hybrid retrieval engine**. Per-query pipeline: vector search (HNSW) +
  keyword search (`ts_rank`) + RRF fusion + source-aware ranking + optional
  reranker (ZeroEntropy). Three named search modes (`conservative`,
  `balanced`, `tokenmax`) bundle the knobs.
- A **self-wiring knowledge graph**. Every page write extracts
  `[Name](people/slug)` markdown links and `[[people/slug|Name]]` Obsidian
  wikilinks into typed edges (`attended`, `works_at`, `invested_in`,
  `founded`, `advises`, `mentions`). **No LLM calls** during extraction.
- A **9-phase nightly maintenance cycle** ("dream cycle"):
  `lint → backlinks → sync → synthesize → extract → patterns →
  recompute_emotional_weight → embed → orphans` (+ a `purge` 10th phase for
  the destructive-guard lifecycle).
- An **MCP server** speaking both stdio (single-user, Claude Code / Cursor)
  and HTTP + OAuth 2.1 (multi-user, ChatGPT, Perplexity, federated teams).
- A **fat-skill harness**. ~35 curated markdown skills ship in the bundle
  (`signal-detector`, `brain-ops`, `meeting-ingestion`, `enrich`,
  `daily-task-manager`, etc.). Skills are scaffolded into the host agent
  repo with `gbrain skillpack scaffold <name>` — additive copy, host owns
  the files after.

It is **NOT** a wiki, not Obsidian, not Notion. Those handle storage.
GBrain ingests a meeting, enriches the attendees, builds the graph, fixes the
citations, and surfaces the connection three weeks later when it matters.

### Headline numbers (production brain)

| Metric | Value |
|---|---|
| Total pages | ~100,000 |
| People pages | ~16,000 |
| Companies | ~5,000 |
| Concepts | ~8,000 |
| Originals (essays + ideas + drafts) | ~4,000 |
| Daily notes | ~3,500 |
| Media (tweets, books, papers, calls) | ~31,000 |
| LLM conversation transcripts | ~2,200 |
| Cron jobs running autonomously | 108 |
| Live skills | 273 (35 bundle + 238 user-built) |
| LongMemEval R@5 | **97.6%** ($0.50 / 1K queries, no LLM-in-loop) |
| BrainBench P@5 (relational) | **49.1%** (vs 17.8% for graph-off baseline → +31 points from the graph) |

---

## 2. What GBrain Does (User-Visible Surface)

1. **Ingests everything, enriches automatically.** Markdown sync from a git
   brain. Meeting transcripts (Whisper / Groq). EPUB / PDF books. PDFs of
   academic papers. Articles via Reader / Perplexity. Tweets. Voice notes.
   ChatGPT / Claude / Perplexity / agent-fork conversation exports. Email.
   Calendar. Slack archives.
2. **Hybrid search that beats vector-only RAG.** Two-stage CTE so HNSW stays
   usable when source-boost re-ranks the top-K. Hard exclude prefixes
   (`test/`, `archive/`, `attachments/`) by default. Intent classifier
   auto-selects detail level by query type.
3. **Self-maintains via the dream cycle.** Conversation transcripts become
   brain pages with content-hash dedup. Contradictions are detected,
   typed-claim trajectories tracked, anomalies flagged.
4. **Multi-source brains + temporal trajectory.** Multiple sources mount
   inside one DB (`host` brain + team brain). Author typed claims in the
   `## Facts` fence (`mrr=50000`, `arr=2000000`); `gbrain eval trajectory
   <slug>` prints the chronological history with regressions auto-flagged.
5. **Durable background work.** `gbrain agent run` submits LLM-loop
   subagents. `gbrain jobs supervisor` runs a self-restarting worker
   daemon with crash-cause classification (`runtime_error /
   oom_or_external_kill / graceful_shutdown / clean_exit`).

### Headline commands

```text
SETUP        gbrain init [--pglite | --supabase]
             gbrain migrate --to {supabase,pglite}
             gbrain doctor [--fast] [--fix]
CONTENT      gbrain sync [--workers N]
             gbrain import <path>
             gbrain embed [--stale]
             gbrain extract {links,timeline,all}
SEARCH       gbrain query "<question>"        # hybrid + RRF + rerank
             gbrain search "<text>"           # vector | keyword | hybrid
             gbrain whoknows <topic>          # expertise routing
             gbrain graph-query <slug>        # typed-edge traversal
             gbrain orphans                   # pages with zero inbound links
AGENTS       gbrain agent run "<prompt>"     # LLM subagent
             gbrain jobs submit shell --params ...   # operator-only
             gbrain jobs supervisor start
CYCLE        gbrain dream [--phase NAME]      # 9-phase nightly cycle
             gbrain autopilot --install
SERVE        gbrain serve                     # stdio MCP
             gbrain serve --http              # HTTP MCP + OAuth 2.1 + admin
EVAL         gbrain eval longmemeval <dataset>
             gbrain eval cross-modal --task "..."
             gbrain eval trajectory <entity>
             gbrain founder scorecard <entity>
```

---

## 3. Core Concepts

### 3.1 The Two-Axis Knowledge Model

Every query routes on TWO orthogonal axes. Misroute either one, and you get
silently wrong results.

| Axis | What it picks | Example |
|---|---|---|
| **Brain** | WHICH DATABASE | `host` (your personal DB), a mounted team DB, a CEO-class shared brain |
| **Source** | WHICH REPO INSIDE THE DATABASE | `wiki`, `gstack`, `openclaw`, `essays` |

Resolution chain for both axes (highest priority wins):
**flag → env var → dotfile → longest-prefix match → default**.

- `--brain <id>` / `GBRAIN_BRAIN_ID` / `.gbrain-mount`
- `--source <id>` / `GBRAIN_SOURCE` / `.gbrain-source`
- Mounts registered via `gbrain mounts add <id>` live in `~/.gbrain/mounts.json`

Rule of thumb: *if the data owner changes, it's a brain boundary. If the data
owner stays the same but the topic/repo changes, it's a source boundary.*

### 3.2 Trust Boundary (`OperationContext.remote`)

Every operation runs with a `remote` flag in its context:

- `remote === false` — **trusted local CLI caller**. Set by `src/cli.ts`.
- `remote === true` — **untrusted agent-facing caller**. Set by
  `src/mcp/server.ts` and the HTTP MCP transport.

Security-sensitive ops harden when `remote=true`:

- `file_upload` tightens filesystem confinement to a canonical path.
- `put_page` requires `wiki/agents/<subagent-id>/...` namespacing unless the
  trusted-workspace allow-list (`allowedSlugPrefixes`) is set.
- 4 ops are **`localOnly`** and rejected over HTTP entirely: `sync_brain`,
  `file_upload`, `file_list`, `file_url`, `purge_deleted_pages`,
  `get_recent_transcripts`. The MCP server returns a clear "local-only"
  error pointing at the local CLI.

Fail-closed semantics: anything that **isn't strictly `false`** is treated as
remote. This is checked by `scripts/check-test-isolation.sh` and 4 trust-boundary
unit tests so transports that forget to set the field can't downgrade trust.

### 3.3 System of Record: Markdown is Canon, DB is Derived

> The GitHub repo (markdown + frontmatter) is the system of record. The
> Postgres / PGLite database is a derived cache. We do not back up the
> database — we rebuild it from the repo.

**FS-canonical tables** (markdown is source, DB is derived):

- **Takes** — fenced table between `<!--- gbrain:takes:begin -->` / `:end -->`
- **Facts** — fenced table between `<!--- gbrain:facts:begin -->` / `:end -->`
- **Links** — inline `[text](slug)` + frontmatter `direction: incoming`
- **Timeline** — section after `<!-- timeline -->` sentinel
- **Tags** — frontmatter `tags:` YAML array

Disaster recovery is one command:

```bash
gbrain rebuild --confirm-destructive
gbrain sync && gbrain extract all
```

### 3.4 Page Model: Compiled Truth + Timeline

Every page splits into two zones, with the compiled-truth section weighted
higher by search:

```markdown
---
name: Alice Example
type: person
tags: [founder, fintech]
sources: [wiki]
---

# Alice Example

## Summary
[compiled truth — current synthesis, rewritten as evidence accumulates]

## Facts
<!--- gbrain:facts:begin -->
| metric | value | unit | period | valid_from |
| mrr | 50000 | USD | monthly | 2026-04-01 |
| arr | 2000000 | USD | annual | 2026-04-01 |
<!--- gbrain:facts:end -->

<!-- timeline -->
## Timeline
- **2026-04-03** | Met at YC W26 demo day [Source: meeting notes #4421]
- **2026-04-10** | Pitched at fund-a partner meeting [Source: email thread]
```

- **Compiled truth** is mutable, rewritten as evidence changes.
- **Timeline** is append-only. Contradictions are corrected with new dated
  entries, not edits.
- **Facts fence** carries typed claims; the `extract_facts` cycle phase wipes
  and reinserts the table per page so the markdown stays canonical.

### 3.5 Takes vs Facts (Two Epistemological Layers)

| | **Takes** (cold) | **Facts** (hot, v0.31+) |
|---|---|---|
| What | WHO believes WHAT | Brain owner's personal knowledge |
| Source | Markdown pages | Conversation, per turn |
| Extractor | LLM batch (nightly) | Haiku per-turn hook |
| Kinds | `take`, `fact`, `bet`, `hunch` | `event`, `preference`, `commitment`, `belief`, `fact` |
| Lifecycle | Cold, retrospective | Real-time |
| Volume (Garry's brain) | ~100K rows | Grows per session |
| Bridge | Dream `consolidate` phase promotes hot facts → cold takes nightly | |

Category error: never dump takes into facts (those are other people's
attributed beliefs); never dump facts into takes without consolidation.

---

## 4. Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              USER + AGENTS                                   │
│                                                                              │
│  Claude Code  Cursor  Windsurf  ChatGPT  Perplexity  OpenClaw  Hermes        │
└────────────────────────────────┬─────────────────────────────────────────────┘
                                 │ MCP (stdio | HTTP + OAuth 2.1)
                                 │ + CLI invocations
                                 ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                          GBRAIN HARNESS (Bun binary)                         │
│                                                                              │
│  ┌─────────────┐    ┌──────────────────┐    ┌─────────────────────────────┐  │
│  │  src/cli.ts │    │  src/mcp/        │    │  src/commands/serve-http.ts │  │
│  │  (local)    │    │  server.ts       │    │  (Express + OAuth 2.1)      │  │
│  │  remote=    │    │  (stdio)         │    │  + Admin SPA + SSE feed     │  │
│  │   false     │    │  remote=true     │    │  remote=true                │  │
│  └──────┬──────┘    └────────┬─────────┘    └──────────────┬──────────────┘  │
│         │                    │                             │                 │
│         └────────────────────┴─────────────────────────────┘                 │
│                                 │                                            │
│                                 ▼ dispatchToolCall(ctx, params)              │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │              src/core/operations.ts  (CONTRACT-FIRST, ~47 ops)         │  │
│  │   query | search | put_page | get_page | submit_job | find_experts     │  │
│  │   find_trajectory | find_anomalies | find_contradictions | ...         │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│         │                              │                       │             │
│         ▼                              ▼                       ▼             │
│  ┌─────────────┐   ┌─────────────────────────────────┐   ┌────────────────┐  │
│  │ Search      │   │ Engine Factory                  │   │ Minions Queue  │  │
│  │  hybrid.ts  │   │  src/core/engine-factory.ts     │   │  (Postgres-    │  │
│  │  + intent   │   │                                 │   │   native)      │  │
│  │  + expand   │   │ ┌────────────┐ ┌──────────────┐ │   │                │  │
│  │  + RRF      │◀──┤ │ PGLite     │ │ Postgres     │ │   │ jobs +         │  │
│  │  + rerank   │   │ │ WASM       │ │ pgvector     │ │   │ subagents +    │  │
│  │  + source-  │   │ │ embedded   │ │ Supabase /   │ │   │ workers +      │  │
│  │    boost    │   │ │ Postgres   │ │ self-hosted  │ │   │ supervisor     │  │
│  │  + dedup    │   │ │ 17.5       │ │              │ │   │                │  │
│  │  + budget   │   │ └────────────┘ └──────────────┘ │   │ pg LISTEN/     │  │
│  └─────────────┘   └─────────────────────────────────┘   │ NOTIFY         │  │
│         │                              │                  │ (poll on      │  │
│         │                              │                  │  PGLite)      │  │
│         ▼                              ▼                  └────────────────┘  │
│  ┌─────────────┐   ┌──────────────────────────────────────────────────────┐ │
│  │ AI Gateway  │   │              Pages + Chunks + Links                  │ │
│  │ src/core/ai/│   │   pages, content_chunks, links, timeline_entries,    │ │
│  │ gateway.ts  │   │   tags, raw_data, files, ingest_log, config,         │ │
│  │             │   │   page_versions, oauth_clients, mcp_request_log,     │ │
│  │ OpenAI /    │   │   sources, dream_verdicts, takes, facts, eval_*      │ │
│  │ Voyage /    │   └──────────────────────────────────────────────────────┘ │
│  │ Anthropic / │                                                            │
│  │ ZeroEntropy │   ┌──────────────────────────────────────────────────────┐ │
│  │ /Gemini /   │   │              Cycle Engine (src/core/cycle.ts)        │ │
│  │ Ollama /    │   │  lint → backlinks → sync → synthesize → extract →    │ │
│  │ LiteLLM     │   │   patterns → recompute_emotional_weight → embed →    │ │
│  └─────────────┘   │              orphans → purge                         │ │
│                    └──────────────────────────────────────────────────────┘ │
└──────────────────────────────┬───────────────────────────────────────────────┘
                               │
                               ▼ Markdown is source of truth
┌──────────────────────────────────────────────────────────────────────────────┐
│              BRAIN REPO  (git-tracked markdown + YAML frontmatter)           │
│                                                                              │
│  brain/                                                                      │
│  ├── RESOLVER.md            # master decision tree                           │
│  ├── people/                # one page per human                             │
│  ├── companies/             # one page per organisation                      │
│  ├── deals/                 # financial transactions                         │
│  ├── meetings/              # records of specific events                     │
│  ├── projects/ ideas/ concepts/ writing/ programs/                           │
│  ├── org/ civic/ media/ personal/ household/ hiring/                         │
│  ├── sources/ inbox/ archive/                                                │
│  └── gbrain.yml             # db_tracked vs db_only tiering                  │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Layered view (philosophical)

```
                    ┌──────────────────────────────────────┐
   Latent space     │  L4  Skills (fat markdown recipes)   │   judgement
                    └──────────────────────────────────────┘
                                    │
                    ┌──────────────────────────────────────┐
                    │  L3  Thin Harness (~200 LOC CLI)     │
                    └──────────────────────────────────────┘
                                    │
                    ┌──────────────────────────────────────┐
                    │  L2  Operations Contract (~47 ops)   │   deterministic
                    │      operations.ts ⇒ CLI + MCP +     │
                    │      subagent tool allowlist         │
                    └──────────────────────────────────────┘
                                    │
                    ┌──────────────────────────────────────┐
   Deterministic    │  L1  BrainEngine (pluggable: PGLite  │
                    │      or Postgres) + Search + Jobs    │
                    └──────────────────────────────────────┘
```

> *"Push intelligence UP into skills. Push execution DOWN into tools.
> Same input, same output, every time."*
> — `docs/ethos/THIN_HARNESS_FAT_SKILLS.md`

---

## 5. Data Flow Charts

### 5.1 Ingestion: a meeting transcript becomes brain knowledge

```
   ┌──────────────────┐   gbrain agent run
   │  Inbound signal  │   recipes/meeting-ingest cron
   │  (raw text,      │   webhook from Fireflies/Granola
   │   audio, PDF,    │
   │   tweet, email)  │
   └────────┬─────────┘
            │
            ▼ Whisper / Groq (audio), pdf-parse (PDF), etc.
   ┌──────────────────┐
   │  Diarised text   │
   └────────┬─────────┘
            │
            ▼ Skill: meeting-ingestion (markdown recipe)
   ┌──────────────────────────────────────────────────┐
   │  Page write to brain repo:                       │
   │  meetings/2026-04-03-fund-a-call.md              │
   │  ├─ frontmatter (name, type, tags, attendees)    │
   │  ├─ compiled-truth: agent's own analysis         │
   │  ├─ ## Facts fence (typed claims)                │
   │  └─ full transcript below <!-- timeline -->      │
   └────────┬─────────────────────────────────────────┘
            │
            ▼ gbrain sync (filesystem walk)
   ┌──────────────────┐
   │  Markdown parser │  splitBody + frontmatter validation
   │  + chunker       │  (recursive | semantic | LLM-guided)
   └────────┬─────────┘
            │ chunks
            ▼
   ┌──────────────────────────────────────────────────┐
   │  put_page (src/core/operations.ts)               │
   │  ├─ INSERT pages + content_chunks                │
   │  ├─ auto-link: extract.ts walks [Name](slug) +   │
   │  │  [[wikilink]] → INSERT links (typed edges)    │
   │  ├─ timeline parser → timeline_entries           │
   │  └─ raw_data (preserve source)                   │
   └────────┬─────────────────────────────────────────┘
            │
            ▼ gbrain embed --stale  (or scheduled)
   ┌──────────────────┐
   │  AI Gateway →    │  OpenAI / Voyage / Gemini /
   │  embed batches   │  ZeroEntropy / Ollama / ...
   │  (size-budgeted) │  1536-2560 dim vectors
   └────────┬─────────┘
            │
            ▼ UPDATE content_chunks SET embedding = $vector
   ┌──────────────────┐
   │  Searchable      │  pgvector HNSW index
   │  within minutes  │  pg_trgm + ts_rank indexes
   └──────────────────┘
            │
            ▼ Enrichment fan-out (every cron job calls enrich)
   ┌──────────────────────────────────────────────────┐
   │  Tier 1 (key people): 10-15 API calls            │
   │  Tier 2 (notable):    3-5 API calls              │
   │  Tier 3 (minor):      1-2 API calls              │
   │  Sources: web, X, LinkedIn, Crustdata, ...       │
   │  → New timeline entries on every attendee page   │
   │  → Cross-links person ↔ company ↔ deal           │
   └──────────────────────────────────────────────────┘
```

### 5.2 Query: a question becomes ranked results

```
   ┌──────────────────┐
   │  User question   │  e.g. "what did alice and i discuss last quarter?"
   └────────┬─────────┘
            │
            ▼  CLI: gbrain query   MCP tool call: query
   ┌──────────────────────────────────────────────────┐
   │  src/core/operations.ts :: query handler         │
   └────────┬─────────────────────────────────────────┘
            │
            ▼  hybridSearch (src/core/search/hybrid.ts)
   ┌──────────────────────────────────────────────────┐
   │  1. Intent classifier                            │
   │     entity | temporal | event | general          │
   │     → sets detail level                          │
   ├──────────────────────────────────────────────────┤
   │  2. Sanitise query (prompt-injection guard)      │
   ├──────────────────────────────────────────────────┤
   │  3. Multi-query expansion (Haiku)                │
   │     (skipped in `conservative` & `balanced`,     │
   │      ON in `tokenmax`)                           │
   ├──────────────────────────────────────────────────┤
   │  4. Cache lookup (query_cache, knobs_hash gate)  │
   ├──────────────────────────────────────────────────┤
   │  5. Parallel:                                    │
   │     ┌───────────────┐  ┌─────────────────────┐   │
   │     │ vector search │  │ keyword search      │   │
   │     │ pgvector HNSW │  │ ts_rank x source-   │   │
   │     │ 2-stage CTE   │  │ factor CASE         │   │
   │     │ + source-boost│  │ + hard-exclude prefs│   │
   │     └───────┬───────┘  └──────────┬──────────┘   │
   │             └────────┬────────────┘              │
   ├──────────────────────────────────────────────────┤
   │  6. RRF fusion: score = Σ 1/(60 + rank)          │
   ├──────────────────────────────────────────────────┤
   │  7. Optional reranker (ZeroEntropy zerank-2)     │
   ├──────────────────────────────────────────────────┤
   │  8. Dedup (4 layers): source dedup, Jaccard,     │
   │     type cap 60%, max 2 chunks/page              │
   ├──────────────────────────────────────────────────┤
   │  9. Token budget enforcement (mode-dependent)    │
   └────────┬─────────────────────────────────────────┘
            │
            ▼ JSON result + optional snippets
   ┌──────────────────┐
   │  Capture (off    │  GBRAIN_CONTRIBUTOR_MODE=1
   │  by default)     │  → eval_candidates table
   └────────┬─────────┘
            │
            ▼
   ┌──────────────────┐
   │  Caller (agent)  │  Loads compiled-truth pages via
   │                  │  gbrain get for full context
   └──────────────────┘
```

### 5.3 The Brain-Agent Loop (operational)

```
            ┌───────── inbound message / signal ─────────┐
            ▼                                            │
   ┌──────────────────┐                                  │
   │ 1. DETECT (async)│  signal-detector skill           │
   │ Sonnet sub-agent │  → entities, original ideas      │
   └────────┬─────────┘                                  │
            │                                            │
            ▼                                            │
   ┌──────────────────┐                                  │
   │ 2. READ          │  brain-first lookup              │
   │ keyword → hybrid │  brain context > external APIs   │
   │ → slug guess     │                                  │
   └────────┬─────────┘                                  │
            │                                            │
            ▼                                            │
   ┌──────────────────┐                                  │
   │ 3. RESPOND       │  answer informed by brain        │
   │ to user          │                                  │
   └────────┬─────────┘                                  │
            │                                            │
            ▼                                            │
   ┌──────────────────┐                                  │
   │ 4. WRITE         │  append to timeline (immutable)  │
   │ to brain pages   │  rewrite compiled truth          │
   │                  │  cross-link entities             │
   └────────┬─────────┘                                  │
            │                                            │
            ▼                                            │
   ┌──────────────────┐                                  │
   │ 5. SYNC          │  gbrain sync && gbrain embed     │
   │                  │  searchable within minutes       │
   └────────┬─────────┘                                  │
            │                                            │
            └──── brain is smarter on next loop ─────────┘

   Invariants:
   • Every READ improves the response.
   • Every WRITE improves future reads.
   • Every cron job MUST call enrich.
```

### 5.4 Dream Cycle (nightly 9-phase + purge)

```
   ┌────────────────────────────────────────────────────────────────────┐
   │                    gbrain dream  (or autopilot cron)               │
   │                                                                    │
   │  Acquires gbrain-cycle lock (DB row + ~/.gbrain/cycle.lock)        │
   │  Each phase: independently runnable via --phase NAME               │
   │  CycleReport schema_version: "1" — totals grow additively          │
   │                                                                    │
   │   1 lint ───────────► detect LLM artifacts, placeholder dates      │
   │     │                                                              │
   │   2 backlinks ──────► enforce iron law: every reference            │
   │     │                  has a bidirectional link                    │
   │     ▼                                                              │
   │   3 sync ───────────► pull markdown deltas; bookmark advances      │
   │     │                  only if head didn't drift mid-sync          │
   │     ▼                                                              │
   │   4 synthesize ─────► transcripts (.txt) → brain pages             │
   │     │                  Haiku verdict cached in dream_verdicts      │
   │     │                  Sonnet subagent per worth-processing tx     │
   │     │                  Allow-listed slug prefixes only             │
   │     ▼                                                              │
   │   5 extract ────────► auto-link materialisation                    │
   │     │                  (typed edges, timeline entries)             │
   │     │                  also extract_facts: rewrite ## Facts fence  │
   │     ▼                                                              │
   │   6 patterns ───────► cross-session theme detection                │
   │     │                  Single Sonnet subagent, lookback_days=30    │
   │     │                  Names a pattern only if ≥ 3 reflections     │
   │     ▼                                                              │
   │   7 recompute_emotional_weight                                     │
   │     │                  tags + take density + user-holder ratio     │
   │     │                  → 0..1 score on pages                       │
   │     ▼                                                              │
   │   8 embed ──────────► embed --stale: bring all chunks up to date   │
   │     │                                                              │
   │     ▼                                                              │
   │   9 orphans ────────► surface pages with zero inbound links        │
   │     │                                                              │
   │     ▼                                                              │
   │  10 purge  ─────────► hard-delete pages whose deleted_at > 72h      │
   │                       hard-delete archived sources whose TTL fired │
   │                                                                    │
   │  Subagents run under a rate-leased Anthropic client                │
   │  Crash classifier: runtime_error | oom | graceful | clean          │
   │  Cycle lock TTL renewed mid-phase by yieldDuringPhase()            │
   └────────────────────────────────────────────────────────────────────┘
```

### 5.5 Minions Job Queue (durable background work)

```
   gbrain jobs submit         gbrain agent run         gbrain dream
        │                          │                       │
        ▼                          ▼                       ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  MinionQueue.add() — Postgres-native, BullMQ-inspired        │
   │  Protected job names ('shell', 'subagent', 'autopilot-cycle')│
   │  require {allowProtectedSubmit: true} → MCP refuses          │
   │  Idempotency keys, backoff (fixed | exponential + jitter),   │
   │  timeouts, max_stalled, parent/child + child_done inbox      │
   └────────────────────────────────┬─────────────────────────────┘
                                    │ INSERT INTO minion_jobs
                                    ▼
   ┌──────────────────────────────────────────────────────────────┐
   │              minion_jobs (Postgres)                          │
   │  states: pending → waiting → active → completed | failed |   │
   │          stalled → retry → dead                              │
   └────────────────────────────────┬─────────────────────────────┘
                                    │ FOR UPDATE SKIP LOCKED
                                    │ + pg LISTEN/NOTIFY (polls   │
                                    │   on PGLite)                │
                                    ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  MinionWorker  (gbrain jobs work)                            │
   │  Handler registry: shell | subagent | subagent_aggregator |  │
   │                    sync | embed | lint | extract | backlinks│
   │                    | import | autopilot-cycle                │
   │  Lock renewal, graceful shutdown, abort signal, timeout safe│
   │  net (5s grace → SIGTERM → 5s → SIGKILL).                   │
   └────────────────────────────────┬─────────────────────────────┘
                                    │
                                    ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  MinionSupervisor (gbrain jobs supervisor)                  │
   │  Spawns 'gbrain jobs work' as child                          │
   │  Exit classifier: oom | runtime | graceful | clean            │
   │  Restart with exp backoff, clean-restart budget cap (10/60s)│
   │  Optional tini-as-PID-1 wrapping for native-addon zombies   │
   └──────────────────────────────────────────────────────────────┘
```

---

## 6. Component Map (where things live)

| Layer | Source | Purpose |
|---|---|---|
| **Entry points** | `src/cli.ts`, `src/mcp/server.ts`, `src/commands/serve-http.ts` | Three transports → same op dispatch |
| **Contract** | `src/core/operations.ts` | ~47 typed ops; one source of truth for CLI + MCP + subagent allow-list |
| **Engine factory** | `src/core/engine-factory.ts` | Dynamic import of `pglite` or `postgres` engine |
| **PGLite engine** | `src/core/pglite-engine.ts` | Embedded Postgres 17.5 via WASM; zero config; forward-reference bootstrap |
| **Postgres engine** | `src/core/postgres-engine.ts` | postgres.js + pgvector; Supabase / self-hosted; HNSW + pg_trgm |
| **Schema** | `src/schema.sql` (+ `src/core/migrate.ts`) | 10 base tables + 65+ migrations; engine-specific `sqlFor` branches |
| **Search** | `src/core/search/{hybrid,intent,expansion,dedup,sql-ranking,source-boost,rerank}.ts` | The 9-step query pipeline |
| **AI gateway** | `src/core/ai/gateway.ts` + `recipes/` | Unified seam for OpenAI / Voyage / Anthropic / Gemini / ZeroEntropy / Ollama / LiteLLM |
| **Ingestion** | `src/core/import-file.ts`, `src/core/chunkers/{recursive,semantic,code}.ts` | Markdown → chunks (29 languages via tree-sitter WASM) |
| **Sync** | `src/core/sync.ts`, `src/commands/sync.ts` | FS walk + manifest + bookmark; parallel worker engines |
| **Knowledge graph** | `src/core/link-extraction.ts`, `src/commands/extract.ts` | Wikilink + markdown link → typed edges (no LLM) |
| **Cycle** | `src/core/cycle.ts` + `src/core/cycle/{synthesize,patterns,extract-facts,...}.ts` | 10-phase nightly maintenance |
| **Minions** | `src/core/minions/{queue,worker,supervisor,handlers/}` | Postgres-native job queue + subagent loop |
| **OAuth / HTTP** | `src/core/oauth-provider.ts`, `src/commands/serve-http.ts`, `admin/` | OAuth 2.1 + admin SPA + SSE feed |
| **Eval surface** | `src/commands/eval-*.ts`, `src/core/eval-*` | LongMemEval, cross-modal, replay, contradictions, trajectory |
| **Skills** | `skills/*/SKILL.md` + `skills/RESOLVER.md` | 35-skill bundle, scaffolded into host repos |

---

## 7. Engine Choice

| | PGLite (default) | Postgres + pgvector |
|---|---|---|
| Setup | `gbrain init` (~2s) | Account + connection string |
| Scale | Good < 1,000 files | Production-proven at 100K+ |
| Multi-device | Single machine | Any device via remote MCP |
| Cost | Free | Supabase Pro ~$25/mo |
| Concurrency | Single process | Connection pooling |
| Backups | File copy | Managed |
| Engine kind | `'pglite'` | `'postgres'` |
| Round-trip | `gbrain migrate --to pglite` | `gbrain migrate --to supabase` |

Both implement the identical `BrainEngine` interface, identical SQL, identical
test surface. The factory chooses at startup; everything above the interface
is engine-agnostic.

---

## 8. Hybrid Search (the 49.1% P@5 secret sauce)

The graph layer is worth **31 P@5 points** on relational queries (vs vector-only
RAG). The search pipeline (`src/core/search/hybrid.ts`):

1. **Intent classifier** — entity / temporal / event / general → detail level
2. **Multi-query expansion** (Haiku, optional per mode)
3. **Cache lookup** — keyed on `(source_id, knobs_hash, embedding similarity)`
4. **Parallel keyword + vector** with source-aware ranking inlined into SQL
   - `originals/`, `concepts/`, `writing/` outrank `wintermute/chat/`, `daily/`,
     `media/x/`
   - Hard exclude prefixes (`test/`, `archive/`, `attachments/`) at retrieval
5. **RRF fusion** — `score = Σ 1/(60 + rank)`
6. **Optional reranker** — ZeroEntropy `zerank-2` cross-encoder
7. **4-layer dedup** — source-dedup, Jaccard > 0.85, type cap 60%, max 2/page
8. **Token-budget enforcement** — per the active search mode

### Search Modes (one-time install decision)

| Knob | `conservative` | `balanced` | `tokenmax` |
|---|---|---|---|
| Token budget | **4 000** | **12 000** | **off** |
| LLM expansion | off | off | **on** |
| Default search limit | 10 | 25 | 50 |
| Cache enabled | yes | yes | yes |
| Cost @10K queries/mo × Haiku | **$40/mo** | $100/mo | $200/mo |
| Cost @10K queries/mo × Opus | $200/mo | $500/mo | **$1,000/mo** |

Corner-to-corner spread is **25×**. The installer asks the operator to pick
explicitly — silent default acceptance is treated as wrong.

---

## 9. Knowledge Graph (self-wiring, no LLM)

`src/core/link-extraction.ts` runs on every page write:

- Matches `[Name](people/slug)` markdown links AND `[[people/slug|Name]]`
  Obsidian wikilinks
- Heuristics infer link type from surrounding context:
  `attended` | `works_at` | `invested_in` | `founded` | `advises` |
  `source` | `mentions`
- Bulk insert via `unnest($1::text[], $2::text[], ...)` arrays — 4-5 params
  regardless of batch size, sidesteps Postgres's 65535-param cap
- Auto-link disabled for `ctx.remote && !trustedWorkspace` so a sub-agent's
  put_page doesn't materialise side-effects without supervision

Query the graph: `gbrain graph-query alice --depth 2 --direction in` returns
who attended what with Alice, transitively.

The graph also powers:

- `whoknows <topic>` — expertise routing via
  `score = log(1 + raw_match) × max(0.1, exp(-days/180)) × (0.5 + 0.5×salience)`
- `find_trajectory <entity>` — chronological typed-claim history with
  regressions auto-flagged (10 % drop default)
- `find_contradictions` — six-verdict enum
  (`no_contradiction | contradiction | temporal_supersession |
  temporal_regression | temporal_evolution | negation_artifact`)

---

## 10. Skills (the 35-skill bundle)

Skills are **fat markdown files** with YAML frontmatter:

```yaml
---
name: meeting-ingestion
triggers:
  - "ingest meeting"
  - "process transcript"
writes_pages:
  - meetings/*
  - people/*
---
```

Agents scan `skills/*/SKILL.md` frontmatter at startup, match user intent
against `triggers:` arrays, and invoke on high score. `skills/RESOLVER.md` is
the master dispatcher.

### Categories (35 skills)

| Bucket | Examples |
|---|---|
| Always-on | `signal-detector`, `brain-ops`, `repo-architecture` |
| Content ingestion | `idea-ingest`, `media-ingest`, `meeting-ingestion`, `voice-note-ingest`, `book-mirror`, `brain-pdf` |
| Research + synthesis | `perplexity-research`, `academic-verify`, `strategic-reading`, `archive-crawler`, `article-enrichment`, `concept-synthesis`, `data-research` |
| Brain operations | `query`, `enrich`, `maintain`, `citation-fixer`, `frontmatter-guard` |
| Operational | `daily-task-manager`, `daily-task-prep`, `cron-scheduler`, `reports`, `cross-modal-review`, `webhook-transforms`, `minion-orchestrator` |
| Identity + meta | `soul-audit`, `testing`, `skillify`, `skill-creator`, `skillpack-check`, `skillpack-harvest`, `functional-area-resolver` |
| Conventions (shared deps) | `_AGENT_README.md`, `_brain-filing-rules.md`, `_output-rules.md`, `_friction-protocol.md`, `conventions/*` |

### Skillpack v0.36 contract (scaffolding, not installing)

Pre-v0.36 used a "managed-block" model: an `install` command copied skills
into the host repo and wrote a fence in RESOLVER.md it would rewrite on
upgrade. v0.36 retired that:

- `gbrain skillpack scaffold <name>` — **one-time additive copy**. Refuses
  to overwrite existing files.
- `gbrain skillpack reference <name>` — read-only diff lens. Optional
  `--apply-clean-hunks` auto-merges non-conflicting changes.
- `gbrain skillpack migrate-fence` — one-shot for pre-v0.36 users.
- `gbrain skillpack harvest <slug> --from <host>` — reverse direction; lift
  a proven host skill back into the gbrain bundle. Default-on privacy linter
  rejects anything matching `\bWintermute\b`, emails, Slack-channel
  patterns, or a custom `~/.gbrain/harvest-private-patterns.txt` list.

---

## 11. Deployment Topologies

### Pattern 1 — One server, many MCP clients (recommended)

GBrain runs on the box that hosts the primary agent. Other agents — on other
machines, other apps, teammates' laptops — connect as thin MCP clients via
HTTP + OAuth 2.1.

```bash
# Host machine
gbrain serve --http --bind 0.0.0.0 --public-url https://brain.example.com

# Thin client (any other machine)
gbrain init --mcp-only
```

Recommended network: **Tailscale**. Run the server on a Tailscale-private
address. Public-internet exposure works (OAuth + loopback-bind-by-default +
admin dashboard), it's just a larger blast radius.

Source-scoped OAuth clients (v0.34+): a teammate's client can be tied to one
source with `--source dept-x`; writes are RLS-isolated. `--federated-read
S1,S2,S3` adds orthogonal read scopes for cross-department reads while
writing to one canon.

### Pattern 2 — Local PGLite per worktree (GStack code-search use case)

For an engineering agent that needs symbol-aware code search on the repo
it's currently editing. Per-worktree PGLite, indexed against that worktree's
source tree.

```bash
gbrain init --pglite
gbrain import .
echo "$(pwd)" > .gbrain-source
```

`gbrain code-def`, `gbrain code-refs`, `gbrain code-callers` walk
tree-sitter call-graph edges across 29 languages. **Not for shared
knowledge** — single-process, worktree-scoped, no concurrent writers.

### Pattern 3 — Federated repos (advanced)

Multiple GBrain servers indexing the same brain repo from their own git
clones. Each server has its own DB; they sync to the same markdown source
of truth. Use this when crossing a hard trust boundary (personal brain RW +
team brain RO).

> *Most teams should NOT pick this.* The MCP-client path (pattern 1) gives
> you one source of truth, one set of skills, one knowledge graph, and
> OAuth-gated multi-user access.

---

## 12. Trust & Security Model

### OAuth 2.1 (v0.26+)

`gbrain serve --http` ships full RFC-compliant OAuth 2.1:

- `client_credentials` (Perplexity / Claude)
- `authorization_code` with PKCE (ChatGPT)
- `refresh_token` with rotation
- `revokeToken` with `client_id` binding
- RFC 7591 Dynamic Client Registration (`--enable-dcr`, off by default)
- Public PKCE clients (`token_endpoint_auth_method: "none"`) supported
- Scopes: `read` | `write` | `admin`
- Tokens stored as SHA-256 hashes
- Single-use auth codes via atomic `DELETE ... RETURNING`
- Two-bucket rate limiter (pre-auth IP + post-auth token-id) with
  bounded LRU map
- `mcp_request_log` audit row per request (redacted params by default)
- Loopback-bind by default (`127.0.0.1`); explicit `--bind 0.0.0.0` to expose

### Local-only operations

5 ops are `localOnly: true` and rejected over HTTP regardless of scope:
`sync_brain`, `file_upload`, `file_list`, `file_url`, `purge_deleted_pages`,
`get_recent_transcripts`. The MCP server returns a clear error pointing at
the local CLI.

### RLS (Row-Level Security)

Every public-schema table requires RLS. Supabase exposes `public` via
PostgREST to the anon key; RLS off = exfiltration vector. The v0.26.7
**auto-RLS event trigger** fires on `CREATE TABLE` and runs `ALTER TABLE ...
ENABLE ROW LEVEL SECURITY` on every new `public.*` table. Escape hatch is
the table COMMENT `GBRAIN:RLS_EXEMPT reason=<why>` — must be added by hand
in psql, "write it in blood" design. PGLite no-ops (single-user, no
PostgREST).

### Destructive-guard lifecycle

`pages.deleted_at` and `sources.archived_at` columns model a 72-hour
soft-delete TTL. `gbrain sources remove`, `gbrain pages purge-deleted`, and
the `delete_page` MCP op all go through `checkDestructiveConfirmation` which
counts pages/chunks/embeddings/files on the target source and refuses to
proceed without `--confirm-destructive` when data is present. `--yes` alone
is rejected.

---

## 13. Eval Methodology

GBrain takes evaluation seriously. Every retrieval claim ships with a
reproducible command.

### Three eval frameworks

1. **`gbrain eval replay`** — capture queries with
   `GBRAIN_CONTRIBUTOR_MODE=1`; snapshot with
   `gbrain eval export --since 7d > base.ndjson`; make a code change; replay
   with `gbrain eval replay --against base.ndjson`. Reports
   **Jaccard@k**, **top-1 stability**, **latency Δ**. Healthy ≥ 0.85, safe
   merge ≥ 85 % top-1.

2. **`gbrain eval longmemeval <dataset>`** — public LongMemEval benchmark
   in-process. Hermetic: one in-memory PGLite per question, `TRUNCATE`
   between questions, `~/.gbrain` never opened. Hand the JSONL output to
   LongMemEval's published `evaluate_qa.py` to score. **R@5 = 97.6 %**.

3. **`gbrain eval cross-modal --task "..." --output report.md`** — 3
   different-provider frontier models (OpenAI gpt-4o, Anthropic
   claude-opus-4-7, Google gemini-1.5-pro) score the output against the task
   on 5 dimensions. Verdict PASS (mean ≥ 7, min ≥ 5) / FAIL /
   INCONCLUSIVE. Cost cap $5 TTY, $1 non-TTY.

Plus `gbrain eval suspected-contradictions`, `gbrain eval trajectory`,
`gbrain founder scorecard`, `gbrain eval whoknows`.

### Metric glossary (`docs/eval/METRIC_GLOSSARY.md` — CI-regenerated)

| Metric | Plain English |
|---|---|
| **P@k** | "What fraction of the top-K are actually correct?" |
| **R@k** | "Did the right answer land in the top-K?" |
| **MRR** | "Average reciprocal rank of the first correct hit." |
| **nDCG@k** | "Discounted-gain ranking quality — > 0.65 = ship it." |
| **Jaccard@k** | "How much do two top-K sets overlap?" |
| **Wilson 95 % CI** | "Honest confidence band on a small-sample %." |

Every JSON eval response carries `_meta.metric_glossary` so the metric
jargon is self-documenting on the wire.

---

## 14. Philosophy & Design Ethos

### Thin Harness, Fat Skills

> *The secret sauce isn't the model. It's the thing wrapping the model: the
> harness. Live repo context. Prompt caching. Purpose-built tools. Context
> bloat minimization.*
> — `docs/ethos/THIN_HARNESS_FAT_SKILLS.md`

GBrain pushes intelligence UP into markdown skills (where 90 % of value
lives) and pushes execution DOWN into deterministic tools. The CLI is a
thin (~200 LOC) context-managing harness. Skills are reusable markdown
procedures encoding judgement.

### Markdown Skills as Recipes

> *The markdown is all four: documentation for humans reading it,
> specification for the implementing agent, package for the distribution
> system, source code for the resulting capability.*
> — `docs/ethos/MARKDOWN_SKILLS_AS_RECIPES.md`

When models cross the judgement threshold, a markdown recipe becomes
simultaneously documentation, specification, package, and executable source
code. Distribution speed = `git push`. "Homebrew for personal AI."

### Knowledge Runtime (4 layers)

`docs/designs/KNOWLEDGE_RUNTIME.md` frames GBrain's roadmap as four layered
abstractions:

1. **Resolver SDK** — unified `{value, confidence, source, fetchedAt,
   costEstimate, raw}` for every external lookup
2. **Enrichment Orchestrator** — tier routing + evidence-weighted
   completeness rubrics
3. **Scheduler** — quiet hours at scheduler level, deterministic staggering,
   circuit breakers
4. **BrainWriter** — *"Iron Law: LLM picks WHAT. Code guarantees WHERE and
   HOW."* All writes through typed operations, validators, JSON-Schema
   sanitisation.

### Recurring themes

- **Determinism + judgement split.** Latent space for understanding,
  deterministic code for execution.
- **Structuring as distribution.** Markdown recipes are not docs, they're
  distributable packages.
- **Evidence over heuristics.** "Length > 500 chars" is a naïve completeness
  rubric; real completeness is evidence-weighted.
- **Orchestration, not execution.** GBrain orchestrates around execution;
  handlers live in platforms.
- **Fail-closed by default.** Circuit breakers, validators, health checks.
- **Skills that rewrite themselves.** `/improve` reads NPS feedback,
  extracts patterns, writes rules back into matching skills.

---

## 15. Integrations & Embedding Providers

Recipes live in `recipes/` (YAML frontmatter + markdown setup body). Each
recipe self-describes: secrets, health_checks, setup_time, requires (DAG).

### 14 embedding providers supported

- **OpenAI** (default) — `text-embedding-3-large` 1536d, $0.13/M
- **Voyage AI** — `voyage-4-large` 1024-2048d Matryoshka, code/finance/law variants
- **Google Gemini** — `gemini-embedding-001` 768-3072d Matryoshka, $0.025/M
- **Azure OpenAI** — enterprise tenancy
- **MiniMax** — `embo-01` 1536d, asymmetric db/query encoding
- **DashScope** (Alibaba) — `text-embedding-v3` 64-1024d, CJK-aware
- **Zhipu AI** — `embedding-3` 256-2048d Matryoshka
- **Ollama** (local, free) — `nomic-embed-text` 768d
- **llama-server** (local, free) — any GGUF, `--embeddings` endpoint
- **ZeroEntropy** — `zembed-1` 2560/1280/640/320/160/80/40d + `zerank-2` reranker
- **LiteLLM proxy** — routes to 100+ providers

Dimension tradeoff: pgvector HNSW caps at 2000 dims. Above that → exact
vector scans (correct but slower). Standard recommendation: stay at 1024 or
1536.

### Credential gateway

Two paths:
- **ClawVisor** — credential vaulting + task-scoped authorisation
  (Gmail / Calendar / Drive / Slack / GitHub)
- **Hermes Agent** — built-in multi-platform messaging gateway

### MCP client coverage

| Client | Transport | Notes |
|---|---|---|
| Claude Code | stdio | Zero setup |
| Cursor | stdio | Zero setup |
| Windsurf | stdio | Zero setup |
| Claude Desktop | stdio or HTTP OAuth | Per-client `mcp.json` |
| Claude Cowork | HTTP OAuth | |
| ChatGPT desktop | HTTP OAuth 2.1 + PKCE | **Only path** (no stdio) |
| Perplexity | HTTP OAuth `client_credentials` | |

---

## 16. Storage Tiering

For brains crossing 100K files where bulk machine-generated content
dominates the size. Declare which directories are committed vs DB-only in
`gbrain.yml`:

```yaml
storage:
  db_tracked:
    - people/
    - companies/
    - deals/
    - concepts/
  db_only:
    - media/x/
    - media/articles/
    - meetings/transcripts/
```

- `gbrain sync` auto-manages `.gitignore` for `db_only` paths (idempotent,
  skipped on `--dry-run`)
- `gbrain export --restore-only` repopulates missing db_only files from the
  DB — essential for container restarts
- `gbrain storage status` shows tier health
- Backward compatible; progressive enhancement

---

## 17. Reliability & Repair Surface

| Command | What it fixes |
|---|---|
| `gbrain doctor [--fast] [--fix]` | 30+ health checks: schema_version, RLS, JSONB integrity, markdown body completeness, sync failures, graph coverage, queue health, supervisor crash classification, sync freshness, emit eval-capture errors, reranker health, OAuth scope drift |
| `gbrain repair-jsonb [--dry-run]` | v0.12.0 double-encoded JSONB rows (5 columns affected) |
| `gbrain integrity check\|auto\|review` | Bare tweets, dead links; three-bucket repair (auto / review-queue / skip) |
| `gbrain orphans` | Pages with zero inbound wikilinks |
| `gbrain frontmatter install-hook` | Pre-commit hook with 12-code error classifier |
| `gbrain apply-migrations [--force-retry N]` | Idempotent schema migrations from TS registry; --force-retry resets a wedged migration |
| `gbrain sync --skip-failed` / `--retry-failed` | Acknowledge or re-attempt unsyncable files |

### Doctor scoring (`brain_score / 100`)

Five components: embed coverage / 35, link density / 25, timeline coverage /
15, orphans / 15, dead links / 10. Sum = 100.

---

## 18. Versioning & Migration

- **Version format** is **mandatory `MAJOR.MINOR.PATCH.MICRO`** — four
  numeric segments (e.g. `0.31.4.1`). The `.MICRO` slot is the dot-suffix
  follow-up channel.
- **Five files must move together** every release: `VERSION`,
  `package.json`, `CHANGELOG.md`, `TODOS.md`, `CLAUDE.md` annotations. Auto-
  derived: `bun.lock`, `llms-full.txt`, `llms.txt`.
- CI version-gate fails if `VERSION` and `package.json` disagree, OR if
  `VERSION` doesn't strictly exceed master's.
- `gbrain upgrade` runs `gbrain post-upgrade` which runs
  `gbrain apply-migrations`. The chain is best-effort; failures land in
  `~/.gbrain/upgrade-errors.jsonl` and surface in `gbrain doctor`.
- Every release with user-facing action has a `## To take advantage of
  v[version]` self-repair block in `CHANGELOG.md`.

---

## 19. Testing Layout

Five tiers of test commands (`bun run test`, `verify`, `test:full`,
`test:slow`, `test:serial`, `test:e2e`, `ci:local`):

- **Unit** (`*.test.ts`) — parallel 8-shard fan-out, ~85 s for 3650+ tests.
  No DB required.
- **Slow** (`*.slow.test.ts`) — intentional cold-path correctness checks.
- **Serial** (`*.serial.test.ts`) — quarantine for cross-file contention
  (`mock.module(...)`, env-coupled tests). Runs at `--max-concurrency=1`.
- **E2E** (`test/e2e/*.test.ts`) — real Postgres + pgvector container.
  Spin up `gbrain-test-pg`, run, tear down. Lifecycle is documented in
  CLAUDE.md and must be followed verbatim.
- **Local CI gate** (`bun run ci:local`) — full Docker stack: gitleaks +
  unit + all 29 E2E sequential against fresh pgvector.

Static lint rules in `scripts/check-test-isolation.sh` enforce R1-R4:
no `process.env.X = ...` (use `withEnv`), no `mock.module(...)` in
non-serial files, canonical PGLite block in `beforeAll`, paired
`afterAll(disconnect)`.

---

## 20. Synthesis — What Makes GBrain Different

| Feature | What everyone else does | What GBrain does |
|---|---|---|
| Storage | Markdown OR DB | Markdown **and** derived DB (rebuildable) |
| Retrieval | Vector RAG | Hybrid + RRF + source-boost + rerank + dedup |
| Graph | Hand-curated or LLM-extracted | Self-wiring from wikilinks, no LLM in extraction |
| Identity | Notes-on-disk | Two-axis (brain × source) with RLS isolation |
| Trust | Single tenant | `OperationContext.remote`, `localOnly` ops, OAuth 2.1, source-scoped tokens |
| Maintenance | Manual | 10-phase dream cycle nightly |
| Memory | Conversation buffer | Takes (cold) + Facts (hot), promotion via consolidate |
| Distribution | npm / pip / docker | Markdown skills, `git push` |
| Eval | Vibes | LongMemEval + BrainBench + replay + cross-modal + receipts |

The core thesis: **agents are smart but forgetful; the harness is the
bottleneck; the harness should be thin and the skills should be fat;
markdown is the system of record, the DB is derived, and the brain
compounds nightly.**

---

## Appendix A — Key file index (from CLAUDE.md)

The 25 most architecturally significant files:

1. `src/core/operations.ts` — contract-first op definitions (~47 ops)
2. `src/core/engine.ts` — `BrainEngine` interface
3. `src/core/engine-factory.ts` — dynamic engine import
4. `src/core/pglite-engine.ts` — WASM Postgres 17.5
5. `src/core/postgres-engine.ts` — postgres.js + pgvector
6. `src/core/schema-embedded.ts` — auto-generated from `src/schema.sql`
7. `src/core/migrate.ts` — schema migration runner (65+ migrations)
8. `src/core/import-file.ts` — markdown → chunks pipeline
9. `src/core/chunkers/{recursive,semantic,code}.ts` — 3-tier chunking
10. `src/core/search/hybrid.ts` — the query pipeline
11. `src/core/search/{intent,expansion,dedup,sql-ranking,source-boost,rerank}.ts`
12. `src/core/ai/gateway.ts` — unified AI seam
13. `src/core/link-extraction.ts` — typed-edge extractor
14. `src/core/cycle.ts` + `src/core/cycle/*` — dream cycle phases
15. `src/core/minions/{queue,worker,supervisor}.ts` — job queue
16. `src/core/minions/handlers/{shell,subagent,subagent-aggregator}.ts`
17. `src/core/oauth-provider.ts` — OAuth 2.1 server-side
18. `src/commands/serve-http.ts` — Express 5 HTTP MCP + admin SPA
19. `src/mcp/server.ts` — stdio MCP
20. `src/mcp/dispatch.ts` — shared tool-call dispatcher
21. `src/commands/doctor.ts` — health checks + auto-repair
22. `src/commands/dream.ts` — cycle CLI
23. `src/core/destructive-guard.ts` — 72h soft-delete TTL
24. `skills/RESOLVER.md` — skill dispatcher
25. `docs/architecture/brains-and-sources.md` — two-axis model

---

## 21. The Two Companion Essays (root-folder PDFs)

Two long-form posts ship at the repo root, each an image-only PDF (single
~11,800-pt scroll-page rendered from a phone-app capture). They're not
referenced anywhere in README/docs but they're the **canonical narrative
explanation** of *why* GBrain exists. Together they're the strongest
single-doc orientation for new contributors.

### 21.1 `how-to-verify-skills.PDF` — "How to really stop your agents from making the same mistakes"

By **Garry Tan**, April 22. Sub-headline: *Skillify — the 10-item checklist
that turns every agent failure into a permanent capability.*

**The thesis.** LangChain raised $160 M and built LangSmith (trajectory
evals, track-to-dataset pipelines, LLM-as-judge, regression suites). It's
sophisticated. But most AI-agent "reliability" in 2025 is still
vibes-based — `Please don't hallucinate` incantations. The framework gave
you a gym membership without a workout plan. **Skillify** is that workout
plan.

**The two failures that motivated it.**

1. **"The Trip That Was Already in the Database."** The agent was asked
   about a 5-year-old business trip. It hit the OpenClaw calendar API
   → blocked (too far back). Tried email search → noisy. Tried calendar
   API again with different params → still blocked. Five minutes later,
   searched the local knowledge base and found it instantly: **3,146
   calendar files spanning 2013-2026, already indexed**. The agent had
   the right tool (`calendar-recall.mjs`). It chose cleverness over
   discipline.
2. **"28 Minutes."** Agent says: *"Your next meeting is in 28 minutes."*
   Reality: 88 minutes away. Agent did UTC→PT timezone math in its head.
   Script `context-now.mjs` already existed, runs in 50 ms, zero
   ambiguity. Agent just didn't run it.

> *"Both failures have the same shape: the agent had the right tool and
> chose cleverness instead of discipline. In a normal AI setup, the AI
> apologises, promises to do better, two weeks later the same thing
> happens with a different query. The agent has no memory of the bug, no
> test for it, no script for it. We invented a $160 M framework for
> evaluating fancy agentic behaviour, but most agent reliability is still
> vibes-based."*

**The 10-item Skillify checklist** — what `gbrain doctor` actually
enforces:

| # | Check | What it pins |
|---|---|---|
| 1 | **`SKILL.md`** — the contract | name, triggers, rules |
| 2 | **Deterministic code** — `scripts/*.mjs` | no LLM for what code can do |
| 3 | **Unit tests** (vitest) | pure functions, regexes, time math |
| 4 | **Integration tests** | live endpoints + real data |
| 5 | **LLM evals** | quality + correctness (LLM-as-judge against rubric) |
| 6 | **Resolver trigger** | entry in `AGENTS.md` / `RESOLVER.md` |
| 7 | **Resolver eval** | verify the trigger actually routes |
| 8 | **`check-resolvable` + DRY audit** | every skill reachable; no overlaps |
| 9 | **E2E smoke test** | the full pipeline, end to end |
| 10 | **Brain filing rules** | knows where outputs land |

**The shocker stat.** First time Garry ran `check-resolvable` after a
month of building skills:
> *"Found 6 unreachable skills out of 40+. **Fifteen percent of the
> system's capabilities were dark.** A flight tracker nobody could
> invoke. A context-ideas generator that only ran on cron and couldn't
> be triggered manually. A citation fixer that existed in `skills/` but
> wasn't listed in the resolver at all."*

**Skillify as a verb.** Garry's actual workflow: build something
together with the agent in conversation. Try, fail. Say one word:
*"skillify it."* The agent runs all 10 steps. He has ~100 such skills
now. Examples named in the essay: `calendar-recall`, `context-now`,
the OAuth webhook skill ("Bun didn't work — can you remember this as a
webhook skill"), the headless-browser skill.

**Why Hermes Agent alone isn't enough.** Nous Research's Hermes ships
with a `skill_manager` tool that lets agents create / patch / delete
skills based on what they learn — beautiful procedural-memory
mechanism. **But Hermes doesn't test its skills.** No unit tests, no
resolver eval, no DRY audit. Failure modes Garry has watched
accumulate in untested skill systems:

- Agent creates `deploy-k8s` on Monday; Thursday it creates
  `kubernetes-deploy` in a different conversation. Both exist. Both
  trigger on similar prompts. **Ambiguous routing**, nobody notices
  until the wrong one gets used.
- Skill works perfectly when written. Six weeks later the upstream API
  changes its output shape. Skill silently returns garbage until a
  human spots it.
- Autonomously-created skill has a weak trigger that never matches.
  Becomes an orphan, eating index tokens, never running, rotting.

> *"Hermes handles creation beautifully. GBrain handles verification.
> You need both."*

**The big idea.**
> *"In a healthy software-engineering team, every bug gets a test. That
> test lives forever. The bug becomes structurally impossible to recur.
> AI agents should work the same way. Every failure becomes a skill.
> Every skill gets tests. Every skill's judgement improves permanently,
> not just till the context window fills."*

### 21.2 `journey-of-building-this-app.PDF` — "Meta-Meta-Prompting: The Secret to Making AI Agents Work"

By **Garry Tan**, May 5. The other end of the same arc: why he stays up
till 2 AM as YC CEO building this, and what 100,000 brain pages
actually feel like to use.

**The hook — the book that read him back.** A friend recommended Pema
Chödrön's *When Things Fall Apart* during a hard period. Garry asked
his agent to do a **book mirror** — extract all 22 chapters, run a
sub-agent per chapter that simultaneously summarises Chödrön's idea AND
maps it to Garry's actual life. Not "this applies to leaders" pablum;
specific mapping using his family history (immigrant parents, dad from
HK/Singapore, mom from Burma), his job (YC, founders, open source),
what he's been working on with his therapist, what's in his meeting
notes. The output was a **30 000-word two-column brain page**. Took
**~40 minutes**. Costs **~$3 in API calls**. Garry's verdict:

> *"A $300/hour therapist reading this book and applying it to my life
> couldn't do this in a free afternoon. AI did it with full ownership of
> my emotional context, my reading history, my meeting notes, and my
> hundred relationships of loaded cross-references."*

He has done this with **20+ books**. Each one richer than the last
because *the brain* gets richer.

**How book-mirror got better through iteration** — the canonical
case-study for skillification on a single feature:

- **v1**: hallucinated his family ("parents divorced when they
  weren't", "grew up in Hong Kong" when he was born in California).
  → Added a **mandatory fact-check step** against brain-known facts.
- **v2**: less hallucination, but flattened the book's 22 chapters of
  difficult, specific texture into generic motivational quotes.
  → Added **chapter-by-chapter mode** (one pass per chapter, no
  aggregation).
- **v3**: specific per-chapter, but the connections to his life were
  random ("you've talked about feeling anxious before company meetings
  on Thursday").
  → Added **second-pass mirror with explicit similarity scoring**: a
  sub-agent grounded on each chapter's central thesis ("What does this
  chapter say about groundlessness?"), then mapped to compiled patterns
  in the brain only when actually related.

> *"This is what skillification means in practice. Take a thing you
> want to do well, do it manually, extract the failure pattern, write a
> tested skill file with triggers and edge cases — and every fix
> compounds across all future book mirrors."*

**Skills that build skills.** `skillify` itself is a **meta-skill that
creates new skills**. When Garry encounters a workflow he's going to
repeat, he says *"skillify this"* and it auto-generates the tested skill
file. Book-mirror was skillified the first time he did it manually.
`meeting-prep` was skillified after he noticed he was doing the same
steps before every call. Skills compose: `book-mirror` calls
`brain-ops` for storage, `enrich` for context, `cross-modal-eval` for
quality, `pdf-generation` for output.

**The Demis Hassabis prep.** Demis came to YC for a fireside chat;
Mallaby's biography of him had just dropped. *In under two minutes* the
system pulled: Demis's full brain page (months of accumulated articles,
podcast transcripts, Garry's own notes), his published beliefs on AGI
timelines ("50 % scaling, 50 % innovation"), Mallaby biography
highlights, his stated research priorities (continual learning, world
models, long-term memory), cross-refs to Garry's public statements on
AI, three demo scripts for showing the brain's multi-hop reasoning
during the chat, plus conversation hooks where their worldviews overlap
and diverge.

> *"This wasn't a better Google search. It was preparation that used
> my accumulated context about Demis, my own positions, and the
> strategic goals for the conversation. The system prepped not just
> facts but angles."*

**What 100,000 pages of brain looks like.** Every person Garry meets
gets a page (timeline + state + open threads + score). Every meeting:
transcript + structured summary + **entity propagation** — system
walks every person/company mentioned and updates their pages. Every
article / podcast / video / book gets ingested, tagged, cross-referenced.

Schema is simple: **compiled truth at the top** (current best
understanding) + **append-only timeline below** (chronological events)
+ **raw-data sidebars** (source material). *Personal Wikipedia where
every page is continuously updated by an AI that was at the meeting,
read the email, watched the talk, and ingested the PDF.*

> *"This is the difference between having a filing cabinet and having
> a nervous system. The filing cabinet stores things. The nervous
> system integrates, flags what's changed, and surfaces what's relevant
> right now."*

**The four-layer architecture** (companion view of section 4 above):

- **The harness is thin.** OpenClaw is the runtime. A few thousand lines
  of routing. It knows nothing about books, meetings, or founders. It
  just routes.
- **The skills are fat.** 100+ self-contained markdown files. Each one
  is a specific task. `meeting-ingestion`, `enrich`, `media-ingest`,
  `perplexity-research` are name-checked.
- **The data is fungible.** 100 000 pages of structured knowledge in
  the brain repo. Linked, searchable, growing daily.
- **The models are interchangeable.** Opus 4.7 for precision; GPT-5.5
  for recall + exhaustive extraction; DeepSeek V4-Pro for creative
  third-perspective work; Groq + Llama for speed. **The skill decides
  which model is right for the job. The harness doesn't care.**

> *"When someone asks 'what's the model running this?', the answer is:
> wrong question. The model is just the engine. Everything else is the
> car."*

**The "How to Start" path** (essentially mirrors `INSTALL_FOR_AGENTS.md`
but in Garry's voice):

1. **Pick a harness** (OpenClaw / Hermes / your own). Keep it thin.
   Tailscale to your laptop, or Render / Railway in the cloud.
2. **Start a brain with GBrain.** "Inspired by Karpathy's LLM Wiki,
   implemented in OpenClaw, extended into GBrain. Best retrieval system
   I've benchmarked: **95 % recall on LongMemEval, 49 % precision on
   the retrieval loop**, ships **38 installable skills**." (The R@5
   number here is rounded from the 97.6 % cited in README.)
3. **Do something interesting.** Don't start by planning your shell
   architecture. Start by doing a thing — a report, a research dive, a
   meeting prep, an idea capture.
4. **Use it and look at the output.** It'll be mediocre at first. Use
   `cross-modal-eval` — three different models scoring the same output
   along six dimensions you care about. That's where you find out if
   the system understands your specific life, your specific reading,
   your specific work, your judgement.

**Closing line:**
> *"Fat skills. Fat code. Thin harness. The LLM on its own is just an
> engine. You can build your own car. Everything I described here is
> open source and free on GitHub: github.com/garrytan/gbrain."*

### 21.3 How these PDFs map onto the codebase

| PDF claim | Where it lives in the repo |
|---|---|
| 10-item Skillify checklist | `gbrain doctor` checks + `gbrain skillify check <script>` + `scripts/skillify-check.ts` (section 6 of `CLAUDE.md`) |
| `check-resolvable` ("15 % of capabilities were dark") | `src/commands/check-resolvable.ts`, exit-1-on-any-issue (section "src/core/check-resolvable.ts" of CLAUDE.md) |
| DRY audit / auto-fix | `src/core/dry-fix.ts`, `gbrain doctor --fix` |
| Resolver-eval | `src/commands/routing-eval.ts` + `routing-eval.jsonl` fixtures per skill |
| Brain filing rules | `skills/_brain-filing-rules.md` + `src/core/filing-audit.ts` (Check 6) |
| `skillify scaffold` (meta-skill) | `src/commands/skillify.ts` + `src/core/skillify/{generator,templates}.ts` |
| Book-mirror skill | `src/commands/book-mirror.ts` + subagent jobs + `media/books/<slug>-personalized.md` output path |
| Compiled-truth + timeline schema | Section 3.4 above + `src/core/markdown.ts:splitBody` |
| 100K-page production brain | README headline-numbers table |
| Thin harness, fat skills, fat code, interchangeable models | `docs/ethos/THIN_HARNESS_FAT_SKILLS.md` + `docs/ethos/MARKDOWN_SKILLS_AS_RECIPES.md` |
| LongMemEval 95 % R@5 / 49 % P@5 | `gbrain eval longmemeval` + the BrainBench corpus in the sibling `gbrain-evals` repo |

The essays are the **"why"**, the codebase is the **"what"**, and
sections 1-20 of this study are the **"how"**.

---

## Appendix B — Reading order for a new contributor

1. `README.md` (this synthesis is built on it)
2. `AGENTS.md` — install + operating protocol
3. `CLAUDE.md` — architecture reference, key files, trust boundaries
4. `docs/architecture/brains-and-sources.md` — the two-axis model
5. `docs/architecture/system-of-record.md` — markdown is canon
6. `docs/architecture/topologies.md` — deployment patterns
7. `docs/ethos/THIN_HARNESS_FAT_SKILLS.md`
8. `docs/ethos/MARKDOWN_SKILLS_AS_RECIPES.md`
9. `docs/designs/KNOWLEDGE_RUNTIME.md` — the four-layer abstraction
10. `docs/designs/MINIONS_AGENT_ORCHESTRATION.md`
11. `docs/guides/brain-agent-loop.md`
12. `docs/guides/search-modes.md` + `docs/eval/SEARCH_MODE_METHODOLOGY.md`
13. `docs/guides/skillpacks-as-scaffolding.md`
14. `docs/eval-bench.md` + `docs/eval/METRIC_GLOSSARY.md`
15. `CONTRIBUTING.md`

— end of study notes —
