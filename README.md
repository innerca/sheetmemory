[![DOI](https://zenodo.org/badge/1227113132.svg)](https://doi.org/10.5281/zenodo.19973665)
# SheetMemory

An upstream memory intake layer for OpenClaw. SheetMemory decides what deserves to enter memory, writes high-signal items as typed records, and keeps retrieval deterministic. No embeddings. No vector database. No GPU required.

## In one sentence

SheetMemory turns agent memory from "remember to call a tool" into a rule-first admission pipeline: detect signal, decide what should enter memory, write structured records, and retrieve them deterministically.

## At a glance

- **For OpenClaw users who do not trust fuzzy memory alone.**
- **Upstream memory intake:** decide what enters memory before downstream recall begins.
- **Deterministic retrieval:** filter by fields and status, not vector similarity.
- **Local-first:** SQLite storage and optional local-model classification.
- **Rule-first ingestion:** a Perceptor catches high-signal messages before any LLM call.
- **Auditable:** memory records are typed, inspectable, and stable across runs.

## The problem

OpenClaw's native memory relies on the agent remembering to call memory tools. When context compacts or sessions reset, the agent forgets what it knew — and what it was supposed to remember. The result is silent context loss, stale recall, and no audit trail.

SheetMemory moves memory from agent-driven tool calls to **system-driven intake**: every user message passes through a rule-based Perceptor, high-signal content is classified only when needed, and durable memory enters storage as typed records instead of fuzzy recall leftovers.

## Why it is different

Most memory systems optimize for semantic similarity at retrieval time. SheetMemory focuses on the **admission step** first:

1. Detect whether a message should become memory.
2. Type it into a stable schema.
3. Store it locally with explicit fields.
4. Retrieve it with deterministic filters and decay-aware ranking.

That makes it better suited for things agents repeatedly mishandle: user rules, durable preferences, commitments, corrections, and other facts that should not disappear behind fuzzy recall.

In short:

- **SheetMemory** works upstream: memory intake, admission control, structured writes.
- **Vector-first memory systems** work downstream: semantic recall after memory has already been captured.

SheetMemory does not try to out-compete downstream retrieval systems on similarity search. It gives them better input.

## Architecture

```
[User Message] → [Perceptor] → [LLM Classify] → [SQLite] → [Weibull Decay]
                      │                │
                      │ (rule hit)     │ (fallback)
                      ▼                ▼
              [Direct Write]    [Structured Record]
                      │                │
                      └───────┬────────┘
                              ▼
                   [MemoryCorpusSupplement]
                              │
                              ▼
                     [Agent Context]
```

- **Perceptor**: pure regex + keyword engine. Detects time commitments, explicit preferences, identity info, rules/constraints, and corrections. Runs in <1ms per message on the `message_received` hook. High-confidence signals bypass the LLM entirely.
- **LLM Classifier**: maps text to the protocol's 7-type schema (entity/event/fact/rule/impression/plan/reflex). Uses `subagent.run` with `lightContext: true` — no workspace bootstrap, no tool definitions, only the classification prompt (~400 tokens) + user message.
- **SQLite**: per-agent WAL-mode database. Indexes on status, type, agent_id, and expire_at. No embedding tables, no vector store.
- **Weibull Decay**: time-based relevance scoring with configurable half-life, access-frequency boost, and automatic archival below a maintenance threshold.

## Quick install

```bash
openclaw plugins install clawhub:sheetmemory
```

Or from source during development:

```bash
openclaw plugins install . --link
```

## Configuration

```json5
// ~/.openclaw/openclaw.json
{
  plugins: {
    entries: {
      "structured-memory": {
        enabled: true,
        config: {
          classification: {
            model: "ollama:qwen2.5:3b", // recommended
            timeoutMs: 5000,
          },
          decay: {
            halfLifeDays: 14,
            minMaintenanceScore: 0.1,
          },
          recall: {
            maxResults: 15,
          },
        },
      },
    },
  },
}
```

| Field                       | Default       | Description                                                  |
| --------------------------- | ------------- | ------------------------------------------------------------ |
| `classification.model`      | agent primary | Provider/model for classification (e.g. `ollama:qwen2.5:3b`) |
| `classification.timeoutMs`  | 5000          | Max wait for a single classification call                    |
| `decay.halfLifeDays`        | 14            | Days until relevance halves                                  |
| `decay.minMaintenanceScore` | 0.1           | Records below this are auto-archived                         |
| `recall.maxResults`         | 15            | Max records returned per query                               |

## Tools

| Tool                    | Description                                                                                                        |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `memory_record_add`     | Add or update a structured memory record. Auto-classifies type, importance, and confidence.                        |
| `memory_record_find`    | Search records by type, status, importance, confidence, keywords, or text. Sorted by relevance with decay applied. |
| `memory_record_archive` | Archive a record by ID. Reason is required (user request or auto-decay).                                           |

## Benchmark

Classification tested against 4 local models on a 25-case dirty-input benchmark (noise-wrapped, implicit, fragmented, negation, multi-intent, boundary-ambiguous, code-switching, hypothetical, sarcasm, third-party, and very-short). Chinese and English, identical structure.

| Model             | ZH Acc    | EN Acc  | Parse Rate | Avg Latency |
| ----------------- | --------- | ------- | ---------- | ----------- |
| **qwen2.5:3b** ✅ | **64%**   | **64%** | **100%**   | **1.5s**    |
| qwen:7b           | 68%       | 56%     | 92%        | 3.4s        |
| llama3.2:3b       | 71% (15c) | 33%     | 47% (EN)   | 1.5s        |
| gemma-26b (GGUF)  | 71% (15c) | —       | 71%        | 30s+        |

**Recommended model: qwen2.5:3b.** It is the only model with 100% parse rate across both languages and stable performance. The 7B model scores slightly higher on Chinese but loses 8 points on English, has 8% parse failures (self-invented types), and takes 2× the latency. The 26B thinking model is unusable: its internal reasoning chain consumes 200–500 tokens before producing output, leading to 30s+ per message and frequent JSON truncation.

**Importance scoring degrades on small models.** The 3B model collapses importance to binary (4 or 7). SheetMemory's Perceptor handles importance at the rule level — the LLM classifier only polishes the summary and extracts keywords, so this limitation does not gate the system.

Run the benchmark yourself:

```bash
python3 scripts/bench-structured-memory.py --models qwen2.5:3b qwen:7b
python3 scripts/bench-structured-memory.py --lang en --models qwen2.5:3b
```

## Comparison

|                         | SheetMemory              | memory-lancedb   | supermemory      | mem0             |
| ----------------------- | ------------------------ | ---------------- | ---------------- | ---------------- |
| Retrieval engine        | Field filters + LIKE     | Vector (LanceDB) | Vector-graph     | Vector           |
| Classification          | Perceptor rules + LLM    | None             | LLM auto-capture | LLM auto-capture |
| Pre-processing          | Yes (Perceptor <1ms/msg) | No               | No               | No               |
| Embedding required      | No                       | Yes              | Yes              | Yes              |
| GPU required            | No                       | Optional         | Recommended      | Optional         |
| Deterministic retrieval | Yes                      | No               | No               | No               |
| Local-only              | Yes (SQLite + Ollama)    | Yes (LanceDB)    | Partial          | Cloud            |

SheetMemory is complementary to vector-based plugins. It handles the **input side** of the memory pipeline — detection, typing, confidence scoring, and decay — while vector plugins handle semantic similarity at retrieval time.

## Why deterministic retrieval matters

Three properties that vector search cannot provide:

1. **Auditability**: `type=rule AND confidence>=0.7` returns the same results every time. No embedding drift.
2. **Debuggability**: you can `SELECT * FROM memory_records WHERE type='fact'` and see exactly what's stored.
3. **Offline parity**: no embedding model means no version-locked binary dependencies. SQLite runs on every platform OpenClaw runs on.

These properties matter for compliance, legal review, medical workflows, and any scenario where "the AI forgot a safety rule" is more than an inconvenience.

## FAQ

**Does this replace my existing memory plugin?**
No. SheetMemory uses the `MemoryCorpusSupplement` interface and coexists with `memory-core`, `memory-lancedb`, or any other memory plugin. It adds structured, typed records alongside your existing memory pipeline.

**Why 64% accuracy?**
The benchmark uses deliberately dirty input — noise-wrapped, implicit, sarcastic, code-switched messages. Real-world chat is dirty. A clean-input benchmark would show 90%+ but would be dishonest about production performance.

**What if the LLM classifier is unavailable?**
The Perceptor's pure-rule path (RFC §6.2) handles high-confidence detection without an LLM. Rules cover ~30% of memory-worthy input at 90%+ confidence. The system degrades gracefully.

**Can I use a cloud model for classification?**
Yes. Set `classification.model` to any provider/model OpenClaw supports.

## Protocol

SheetMemory implements the [SheetMemory Structured Memory Protocol v1.1](PROTOCOL.md). Contributions should target protocol compliance first, feature additions second.

## Maintenance

This is a reference implementation of the [SheetMemory protocol](PROTOCOL.md). It solves my own OpenClaw memory problems — I'm sharing it because the protocol deserves an implementation, not because I plan to run a plugin business. Community issues and PRs are welcome; response time is not guaranteed.

If you want to take over maintenance, open an issue.

## License

MIT
