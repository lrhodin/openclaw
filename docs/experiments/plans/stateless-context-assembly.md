---
summary: "Plan: Replace compaction with stateless context assembly (retrieval + assembler + memory keeper + structured memory)"
owner: "openclaw"
status: "draft"
last_updated: "2026-02-01"
title: "Stateless Context Assembly Plan"
---

# Stateless Context Assembly Plan

## Context

OpenClaw currently manages long conversations via compaction (pruning + summarization) and
context-pruning extensions. This plan replaces compaction with a stateless pipeline that builds
fresh context each turn via a Context Assembler and a Retrieval Agent. Every worker run receives
a curated, bounded context window built from storage, not unbounded in-memory history.

## Goals

- Eliminate compaction/pruning as the primary context-limit strategy when enabled.
- Build context from scratch each run via Context Assembler + Retrieval Agent.
- Ensure retrieval is agentic but constrained: transcripts + memory only.
- Keep a background Memory Keeper that consolidates durable memories with evolution support.
- Introduce structured memory storage (SQLite) alongside existing markdown memory files.
- Persist internal agent runs separately for introspection/debugging, but hide from UI.
- Every message gets consistent context regardless of conversation length — no weird resets.

## Non-goals

- Changing storage formats for transcripts or memory files.
- Disabling compaction globally when the feature flag is off.
- Adding new user-facing UI for internal sessions (can follow later).
- Providing new user-visible slash commands in the first pass (beyond /compact deprecation).
- Full knowledge graph (graph-based retrieval is a future enhancement, not v1).

## Key Decisions

- Assembler runs in the agent workspace (can read repo files).
- Retrieval Agent runs in the transcripts directory workspace and only reads transcripts/memory.
- Retrieval uses memory_search/memory_get for memory, and read/grep/ls for transcripts.
- Internal sessions are stored in a separate transcripts directory, hidden from UI.
- Minimal provider fixups only; no pruning/compaction/summarization at the sanitizer layer.
- Auto-rotate on overflow becomes an assembler handoff (new session + fresh context).
- **Deterministic recent window**: the last N messages are always included without retrieval.
  Retrieval supplements recent memory — it does not replace it.
- **No fallback theater**: if the assembler/retrieval system fails, it fails honestly. The
  deterministic recent window is the natural floor, not a fallback mechanism.
- **Model independence**: assembler/retrieval and worker models are configured independently.
  The assembler model matters *more* — constructing the right context is harder than executing
  with good context. Power users should put their best model on assembly.
- **Token budget targeting**: the assembler targets 40-60% of the worker model's context window
  for assembled content, leaving the rest as working memory for reasoning. Research shows
  quality degrades significantly as context fills beyond ~75%.
- **Structured memory (Option B hybrid)**: when contextAssembly is enabled, a structured
  memories table in SQLite stores discrete facts/decisions/preferences with evolution support.
  Markdown memory files (MEMORY.md, daily notes) remain as human-editable narrative memory.
  The Memory Keeper writes to both surfaces. Retrieval queries both.

## Architecture Overview

### Token Budget

The assembler enforces an explicit token budget for assembled context:

```
Total context window (e.g. 200k)
├── System prompt + tools (~10-15%)
├── Assembled context (target: 40-60%)
│   ├── Deterministic recent window
│   ├── Retrieved context
│   └── Completion buffer (~5% reserved for assembler's own work)
└── Working memory for reasoning (remaining 25-50%)
```

The budget is configurable. The assembler must complete its work within the budget — if
retrieved context exceeds the target, it prioritizes recent window > high-relevance retrieval
> lower-relevance retrieval.

### Embedding Pre-filter

Before the Retrieval Agent runs, a fast embedding-based relevance pass narrows the search
space. This is deterministic and cheap (no LLM call):

1. Embed the current query/message.
2. Score all transcript chunks and memory entries by cosine similarity.
3. Return the top-K candidates (configurable, default ~50 chunks).
4. The Retrieval Agent then deeply analyzes only pre-filtered candidates.

This makes retrieval both faster and more targeted. The Retrieval Agent doesn't scan
everything — it evaluates a focused set of high-relevance candidates.

Prefer contiguous message spans over scattered individual messages (coherent discourse
segments score higher than isolated turns).

### Deterministic Recent Window

Before any agentic retrieval runs, the last N messages (configurable, default ~15 turns) are
deterministically included in assembled context. This provides:

- Conversational continuity for rapid back-and-forth without retrieval overhead.
- A natural floor: even if retrieval returns nothing, the worker has recent context.
- Reduced retrieval load — the agent only needs to find *older* relevant context.
- Full traces, not summaries — preserving implicit decisions in recent conversation.

The recent window is extracted directly from the current session transcript. No LLM call needed.

### Retrieval Agent (agentic data access)

```
query/intent → Embedding Pre-filter → Retrieval Agent (tools: memory_search/get + read/grep transcripts) → curated result
```

- Runs with workspaceDir = agent sessions transcripts dir.
- Has no access to repo/workspace files.
- Receives pre-filtered candidate chunks from the embedding pass.
- Returns curated text + sources, not raw dumps.
- For memory files, uses memory_search + memory_get.
- For structured memories, queries the memories table directly.
- **Parallel tool calls**: the retrieval agent is given tools it can run in parallel, so it can
  dig for different things concurrently. This is the primary latency mitigation.
- Ensure internal sessions are not indexed by memory search.

### Context Assembler + Worker

```
[Event] → Context Assembler(purpose, token_budget) → Worker Agent → [Output]
```

- Assembler runs in the agent workspace (can read repo context).
- Assembler uses Retrieval Agent as a tool when it needs transcripts or memories.
- Output is a structured AssembledContext (briefing + curated messages + file list).
- Briefing is injected via extraSystemPrompt.
- Assembled messages replace raw history in the worker run.
- Assembler output does not appear in /context or systemPromptReport.
- **Deterministic recent window is merged with retrieval results** — assembler decides
  ordering and deduplication, but cannot drop recent messages.
- **Context refactoring**: the assembler can rewrite contradictory or stale retrieved
  context, not just arrange it. If retrieved memories conflict, the assembler resolves
  them based on timestamps and supersedes relationships.

### Memory Keeper

- Triggered on conclusion events and scheduled cron runs.
- Uses its own Context Assembler (purpose: "consolidate memory").
- Runs in parallel, never blocks user responses.

#### Markdown Surface (always active)

- Writes to memory/YYYY-MM-DD.md as chronological log of sessions.
- Curates MEMORY.md, USER.md, SOUL.md with narrative context.
- Maintains a memory-keeper-state.md for incremental progress.

#### Structured Memory Surface (when contextAssembly enabled)

Memory evolution workflow:

1. **Extract** — pull discrete facts/decisions/preferences from new session.
2. **Match** — query existing memories table by embedding similarity + keyword overlap.
3. **Evaluate** — for each match, classify relationship:
   - **Supersedes**: new info replaces old (preference changed, decision reversed)
   - **Contradicts**: conflicting info, needs resolution
   - **Enriches**: new info adds detail to existing memory
   - **Unrelated**: no meaningful connection
4. **Update** — rewrite matched memory entries in-place when superseded or enriched.
   Set `active = false` on fully superseded entries. Add `supersedes` links.
5. **Link** — create memory_links entries for related/enriches relationships.
6. **Append** — genuinely new information (no match) becomes a new memory entry.
7. **Re-embed** — updated entries get new embeddings automatically.

#### Structured Memory Schema

```sql
CREATE TABLE memories (
  id TEXT PRIMARY KEY,           -- uuid
  content TEXT NOT NULL,          -- the memory text
  type TEXT NOT NULL,             -- fact / preference / decision / project / person / event
  keywords TEXT,                  -- json array of keywords
  tags TEXT,                      -- json array of categorical tags
  entities TEXT,                  -- json array of people/projects/concepts referenced
  embedding BLOB,                -- vector for similarity search
  source_session TEXT,            -- session_id that created this memory
  created_at INTEGER NOT NULL,    -- unix timestamp
  updated_at INTEGER NOT NULL,    -- unix timestamp
  supersedes TEXT,                -- id of memory this replaces (nullable)
  confidence REAL DEFAULT 1.0,    -- certainty score 0-1
  active INTEGER DEFAULT 1        -- soft delete (0 = superseded/archived)
);

CREATE TABLE memory_links (
  from_id TEXT NOT NULL,
  to_id TEXT NOT NULL,
  relation TEXT NOT NULL,          -- related / contradicts / enriches / supersedes
  created_at INTEGER NOT NULL,
  PRIMARY KEY (from_id, to_id)
);
```

### Internal Sessions

- Internal runs (retrieval, assembler, keeper, handoffs) are persisted to a separate
  transcripts directory (e.g. ~/.openclaw/agents/<id>/sessions-internal/...).
- Session entries include internal: true so they are hidden in sessions_list/UI by default.
- Memory indexing ignores internal sessions by default.
- **Observability**: internal sessions are inspectable via a debug flag or dedicated command,
  but never clutter the main history view for users just looking at their conversations.

## Minimal Provider Fixups (Deterministic Only)

Keep only provider-required fixes; no pruning, no summarization:
- Google turn-ordering bootstrap
- Image content normalization
- Tool-use/result pairing repair

## Storage & Indexing

- JSONL transcripts remain the immutable source of truth.
- Markdown memory files remain human-editable and curated.
- SQLite memory index stays, extended with memories + memory_links tables.
- Structured memories are queryable by the Retrieval Agent alongside markdown memories.
- Promote memorySearch.experimental.sessionMemory to stable.
- Ensure internal sessions are excluded from indexing by default.

## Overflow Handoff (Auto-rotate Replacement)

When context assembly is enabled and a run overflows:
1. Start a new session.
2. Invoke the Context Assembler with purpose "handoff after overflow".
3. Assembler pulls from the previous session transcript and returns a curated context.
4. Responder runs with assembled context.
5. The handoff assembler run is stored as an internal session with kind=handoff.

No user-facing error; the system retries with a fresh session.

## Config & Feature Flag

Add agents.defaults.contextAssembly:

```ts
type AgentContextAssemblyConfig = {
  enabled?: boolean;              // default: false
  assemblerModel?: string;        // model for assembler + retrieval (default: agent's main model)
  workerModel?: string;           // model for the actual task execution (default: agent's main model)
  recentWindowTurns?: number;     // deterministic recent messages to always include (default: 15)
  tokenBudgetPercent?: number;    // target % of context window for assembled content (default: 50)
  preFilterTopK?: number;         // embedding pre-filter candidate count (default: 50)
  structuredMemory?: boolean;     // enable structured memories table (default: true)
  memoryKeeper?: {
    enabled?: boolean;            // default: true when contextAssembly enabled
    evolution?: boolean;          // update existing memories vs append-only (default: true)
  };
};
```

When contextAssembly.enabled = true, disable:
- Compaction extensions and pruning extensions.
- Pre-compaction memory flush turn.
- /compact command (warn or deprecate).

## Implementation Plan

### Phase 1: Retrieval Agent

- New module: src/agents/retrieval/ (agent.ts, prompt.ts, tools.ts, types.ts).
- Tool retrieve() runs a subagent with workspaceDir = transcripts dir.
- Uses memory_search/memory_get + read/grep/find/ls.
- Tools support parallel execution for concurrent retrieval.
- Ensure internal sessions are not indexed by memory search.
- Embedding pre-filter as a deterministic step before agent invocation.

### Phase 2: Context Assembler

- New module: src/agents/context-assembler/ (assembler.ts, prompt.ts, tools.ts, types.ts).
- Assembler runs in the agent workspace.
- Token budget enforcement — counts tokens and stays within target.
- Deterministic recent window extraction from current session transcript.
- Inject briefing via extraSystemPrompt.
- Assembled messages replace raw history in the worker run.
- Recent window messages cannot be dropped by the assembler.
- Context refactoring: assembler can rewrite contradictory retrieved content.

### Phase 3: Responder Integration

- Integrate in all entrypoints:
  - Auto-reply (src/auto-reply/reply/agent-runner.ts)
  - CLI agent runs (src/commands/agent.ts)
  - Cron isolated agent runs
  - Hooks that invoke agent runs
- Ensure /context and systemPromptReport remain unchanged.
- Add config parsing for contextAssembly settings.

### Phase 4: Memory Keeper + Structured Memory

- New module: src/agents/memory-keeper/ (keeper.ts, prompt.ts, events.ts, state.ts).
- New module: src/memory/ (schema.ts, store.ts, evolution.ts, queries.ts).
- Schema migration: add memories + memory_links tables to existing SQLite.
- Memory evolution workflow (extract → match → evaluate → update → link → append).
- Event queue + check_events tool for in-loop injections.
- Background run on message receipt + conclusion events.
- Markdown surface continues working as before (daily files + MEMORY.md).

### Phase 5: Overflow Handoff

- Replace auto-compaction in src/agents/pi-embedded-runner/run.ts.
- Start a new session and rerun via assembler handoff flow.
- Store handoff assembler run in internal sessions dir.

## Tests & Verification

- Unit tests:
  - Embedding pre-filter scoring and ranking
  - Retrieval Agent tool + transcript reading behavior
  - Token budget enforcement in Context Assembler
  - Context Assembler output parsing + recent window inclusion
  - Structured memory CRUD + evolution workflow
  - Memory links creation and query
  - Memory Keeper event injection flow
  - Internal session pathing + hiding
- Update existing compaction tests to assert no compaction when flag is on.
- Manual:
  - Enable context assembly; send messages on Telegram/WhatsApp/Slack.
  - Confirm retrieval agent only accesses transcripts/memory.
  - Confirm structured memories are created and evolve over sessions.
  - Confirm internal sessions are persisted but hidden.
  - Force overflow and verify handoff run.
  - Verify token budget is respected across different model context sizes.

## Future Enhancements (Not v1)

- **Graph-based retrieval**: layer a knowledge graph on transcripts for relationship-aware
  traversal. Design retrieval agent interface so graph can slot in later.
- **Fine-tuned compression model**: domain-specific model for context assembly (Cognition
  validated this approach with Devin).
- **Quality judge gate**: lightweight check between assembly and execution to verify
  context coherence and sufficiency.
- **Temporal memory hierarchy**: different retrieval strategies for different time ranges
  (recent sessions as episodes, older as compressed facts).

## Docs (Follow-up)

- New concept doc: "Context Assembly" (overview + config).
- Update Compaction docs to mention the alternative pipeline.
- Update Memory docs for stable sessionMemory (no "experimental" tag).
- Document structured memory schema and evolution behavior.
