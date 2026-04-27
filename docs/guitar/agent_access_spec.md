# Agent Access Architecture
## Guitar Setup Knowledge System — API & MCP Specification

**Version:** 1.1
**Companion documents:** `guitar_setup_parameters_v3.md` · `string_tension_database.md` · `setup_guide_architecture.md` · `setup_tools_reference.md` · `feature_layers.md` · `setup_optimisation_engine.md`

---

## Changelog: v1.0 → v1.1

| Section | Change |
|---|---|
| Header | Version bumped to 1.1; `feature_layers.md` and `setup_optimisation_engine.md` added to companion documents. |
| §5.2 Endpoints | `POST /optimisation/session` endpoint added. Exposes OptimisationSession persistence and export from `setup_optimisation_engine.md §9.3–9.4`. |
| §6 MCP Tools | `guitar_optimise` tool definition added. Wraps the optimisation session endpoint for agent use. |
| §11 Document Relationships | Expanded to include `feature_layers.md` and `setup_optimisation_engine.md`. |

---

## 1. Purpose

This document specifies the agent-facing access layer for the guitar setup knowledge system. It covers:

- What agents can query and what they get back
- The two access surfaces: REST API and MCP server
- The shared data core that serves both
- Context and session handling
- Implementation sequence and operational cost

The system exposes structured knowledge — problem records, fix procedures, tension calculations, tool references — as raw data. Agents consume that data and decide how to present it. No inference is hosted here.

---

## 2. Design Principles

**Stateless by default.** The knowledge API stores nothing about users or sessions. Context travels with each request. This keeps operational surface small, eliminates privacy concerns, and lets any agent call any endpoint without registration.

**Raw data out, presentation to the agent.** Responses are structured records, not formatted answers. The calling agent decides what to surface and how. This makes the API useful across wildly different frontends — a Claude chat, a GPT assistant, a custom app, a voice interface.

**Physics on demand, not pre-baked.** The tension model (`T = UW × (2 × L × f)²`) runs at query time when the caller provides guitar context. Pre-calculated tension values are not stored or returned by default — they are computed for the specific combination of scale, gauge, and tuning the agent passes in.

**Single data core, two protocol surfaces.** REST and MCP are adapters over the same query engine and data layer. Adding MCP after REST is a thin wrapper, not a second system.

---

## 3. High-Level Architecture

```
Agent (Claude / GPT / custom)
        │
        │  POST /query  OR  MCP tool call
        ▼
┌─────────────────────────────┐
│       Protocol Layer         │
│  REST endpoint  │  MCP server│
└────────┬────────┴─────┬──────┘
         │              │
         └──────┬───────┘
                ▼
┌─────────────────────────────┐
│        Query Engine          │
│  symptom match · filter      │
│  tension calc · lookup       │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│         Data Layer           │
│  Problem records             │
│  String tension DB           │
│  Tuning presets              │
│  Tool catalogue              │
│  Skill pathways              │
└─────────────────────────────┘
```

The protocol layer is interchangeable. The query engine and data layer are shared and written once.

---

## 4. Data the API Exposes

Four categories of structured data are queryable. All four are already defined in the companion documents — this API is a read interface over that specification.

### 4.1 Problem records

The primary query target. Each record covers one playability problem (P-01 through P-21, T-01 through T-05). Fields returned per record:

| Field | Type | Description |
|---|---|---|
| `id` | string | Problem code, e.g. `"P-02"` |
| `name` | string | Short name, e.g. `"Buzz across the whole neck"` |
| `group` | enum | `buzz / action / intonation / bending / feel / tension / structural` |
| `symptom_plain` | string | Player-language description |
| `symptom_alt` | string[] | Alternative phrasings used for matching |
| `cause_plain` | string | Plain-language cause, no parameters |
| `cause_principle` | string | One-sentence physical principle |
| `cause_technical` | string | Parameter names, formula references |
| `cause_parameters` | string[] | Parameter IDs implicated, e.g. `["relief_bass", "tension[1..6]"]` |
| `skill_tier` | enum | `t0 / t1 / t2 / t3 / t4` |
| `time_cost` | enum | `none / low / medium / high` |
| `tool_cost_range` | [number, number] \| null | Estimated USD range for tools required |
| `tools_required` | ToolRef[] | Tools with role and required flag |
| `fix_plain` | string | Plain-language fix description |
| `fix_steps` | FixStep[] \| null | Ordered steps where applicable |
| `reversible` | boolean \| null | Whether the fix can be undone |
| `irreversible_note` | string \| null | What goes wrong on overshoot |
| `learnable` | boolean | Whether this is worth developing as a skill |
| `off_ramp` | OffRamp \| null | Escalation condition and guidance |
| `doc_refs` | DocRef[] | Links into companion technical documents |

### 4.2 Tension calculations

Computed on demand when the caller provides guitar context. The API does not return pre-baked tension values — it runs the formula against the supplied inputs.

Inputs required:

| Field | Type | Example |
|---|---|---|
| `scale_bass_mm` | number | `648` |
| `scale_treble_mm` | number | `648` |
| `string_set_id` | string | `"daddario_enyxl1046"` |
| `tuning_preset_id` | string | `"drop_c"` |
| `custom_tuning_hz` | number[] \| null | Per-string override |

Returns:

| Field | Type |
|---|---|
| `tension_lbs` | number[6] |
| `tension_total_lbs` | number |
| `tension_balance_score` | number |
| `gauge_suitability` | enum[6] — `ok / too_slack / too_tight` |
| `relief_target_mm` | number |
| `rezero_required` | boolean |

### 4.3 String sets

Queryable by brand, series, gauge label, or instrument type. Returns unit weight per string alongside gauge and construction metadata. Unit weight is the physics constant that makes tension calculation possible — it is the key field.

### 4.4 Tuning presets

The 10 MVP presets plus custom tuning support. Each preset returns per-string note names, frequencies in Hz, and MIDI note numbers. Custom tunings can be validated against gauge suitability at a given scale length.

---

## 5. REST API Specification

### 5.1 Base

```
Base URL: https://api.yoursite.com/v1
Authentication: API key via header  →  X-API-Key: <key>
Response format: JSON
Rate limit: 60 req/min per key (MVP)
```

All responses follow:

```json
{
  "data": { ... },
  "meta": {
    "query_ms": 12,
    "matched": 3,
    "version": "1.0"
  }
}
```

Errors:

```json
{
  "error": {
    "code": "no_match",
    "message": "No problems matched the query."
  }
}
```

---

### 5.2 Endpoints

#### `POST /query`

The primary endpoint. Accepts a natural language query and optional guitar context. Returns ranked problem records.

**Request:**

```json
{
  "query": "my guitar is buzzing when I play open chords",
  "context": {
    "guitar": {
      "scale_bass_mm": 648,
      "scale_treble_mm": 648,
      "neck_joint": "bolt_on"
    },
    "strings": {
      "string_set_id": "daddario_enyxl1046"
    },
    "tuning": {
      "preset_id": "standard_e"
    },
    "player": {
      "attack_intensity": "medium",
      "skill_tier_max": "t2"
    }
  },
  "limit": 5
}
```

All fields in `context` are optional. Without context the query returns broadly matched records. With context, results are filtered and re-ranked — for example, `skill_tier_max: "t2"` excludes structural problems the player cannot address themselves.

**Response:**

```json
{
  "data": {
    "problems": [
      {
        "id": "P-01",
        "name": "Buzz on open strings only",
        "group": "buzz",
        "match_score": 0.91,
        "match_reason": "symptom match: 'open', 'buzz'",
        "skill_tier": "t1",
        "symptom_plain": "Buzzes when played open, clears when fretted anywhere.",
        "cause_plain": "The nut slot is too shallow for the string — it sits too close to the first fret.",
        "cause_principle": "String vibration amplitude at the nut exceeds the available clearance.",
        "fix_plain": "Re-cut the nut slot deeper for the affected string.",
        "reversible": false,
        "irreversible_note": "Filing too deep requires nut replacement.",
        "learnable": true,
        "off_ramp": {
          "trigger": "Slot filed too deep",
          "who": "tech",
          "cost_indication": "$30–80 for nut replacement including labour"
        },
        "tools_required": [
          { "tool_id": "feeler_gauge", "role": "Confirm clearance above fret 1", "required": true },
          { "tool_id": "nut_files", "role": "Deepen slot", "required": true }
        ]
      }
    ]
  },
  "meta": {
    "query_ms": 18,
    "matched": 3,
    "returned": 1,
    "version": "1.0"
  }
}
```

---

#### `GET /problems/:id`

Returns the full record for a single problem by ID. Useful for agents that identify a problem code and want complete detail.

```
GET /problems/P-02
```

Returns the full `ProblemRecord` shape with all fields including `fix_steps`, `doc_refs`, and `cause_parameters`.

---

#### `POST /tension`

Calculates string tension for a given guitar configuration and tuning. Runs the physics formula server-side. Agents pass context, receive computed values.

**Request:**

```json
{
  "scale_bass_mm": 648,
  "scale_treble_mm": 648,
  "string_set_id": "daddario_enyxl1046",
  "tuning_preset_id": "drop_c",
  "attack_intensity": "medium",
  "target_action_bass_mm": 2.0,
  "target_action_treble_mm": 1.5
}
```

**Response:**

```json
{
  "data": {
    "tension_lbs": [8.2, 14.1, 13.8, 16.4, 15.9, 14.7],
    "tension_total_lbs": 83.1,
    "tension_balance_score": 0.19,
    "gauge_suitability": ["too_slack", "ok", "ok", "ok", "ok", "ok"],
    "relief_target_mm": 0.22,
    "rezero_required": false,
    "flags": [
      {
        "string": 1,
        "code": "T-01",
        "message": "Low E tension 8.2 lbs is below the 9 lb comfortable threshold for Drop C. Consider a heavier gauge for this string."
      }
    ]
  }
}
```

---

#### `GET /strings`

Query the string tension database.

```
GET /strings?brand=daddario&gauge_label=10-46&instrument=electric
```

Returns matching string sets with unit weight per string, construction metadata, and pre-calculated tension at a reference scale and tuning (25.5" / E standard) for quick comparison.

---

#### `GET /tunings`

Returns all tuning presets with per-string notes and frequencies.

```
GET /tunings
GET /tunings/drop_c
```

---

#### `GET /tools`

Returns the tool catalogue — name, category, cost range, and the problem IDs each tool addresses.

```
GET /tools
GET /tools/feeler_gauge
```

---

### 5.3 Query matching

The `/query` endpoint does not use a hosted LLM. Matching works in two stages:

**Stage 1 — Embedding match.** The query string is embedded using a small, fast embedding model (e.g. `text-embedding-3-small` via the OpenAI API, or a self-hosted alternative like `all-MiniLM-L6-v2`). Cosine similarity is computed against pre-embedded `symptom_plain` and `symptom_alt` fields for all 26 problem records. At 26 records, this is a trivially small vector search — no vector database is needed. A plain JSON array with pre-computed embeddings is sufficient.

**Stage 2 — Context filtering.** Matched records are filtered by:
- `skill_tier_max` from the player profile
- Problem group if the query context implies a domain (e.g. a buzz query deprioritises intonation problems)
- Tension-related problems (T-01 to T-05) are excluded unless string/tuning context is provided

**Stage 3 — Re-ranking.** If guitar context is provided and tension can be calculated, problems that are flagged by the tension model for the current configuration are boosted in the ranking. A guitar in Drop C with a 10-46 set will rank T-01 highly even if the query text alone wouldn't.

---

## 6. MCP Server Specification

The MCP server exposes the same query engine as named tools. An MCP-capable agent (Claude, GPT-4o with MCP support, etc.) can discover the tools automatically from the server manifest.

### 6.1 Tool definitions

```json
{
  "tools": [
    {
      "name": "query_setup_problems",
      "description": "Search the guitar setup knowledge base for playability problems matching a symptom description. Returns ranked problem records with causes, fix procedures, skill requirements, and tool lists.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "query": {
            "type": "string",
            "description": "Natural language description of the problem, e.g. 'my guitar buzzes on open strings'"
          },
          "context": {
            "type": "object",
            "description": "Optional guitar and player context to filter and re-rank results",
            "properties": {
              "scale_bass_mm": { "type": "number" },
              "string_set_id": { "type": "string" },
              "tuning_preset_id": { "type": "string" },
              "attack_intensity": { "type": "string", "enum": ["light", "medium", "heavy"] },
              "skill_tier_max": { "type": "string", "enum": ["t0", "t1", "t2", "t3", "t4"] }
            }
          },
          "limit": { "type": "integer", "default": 3 }
        },
        "required": ["query"]
      }
    },
    {
      "name": "calculate_string_tension",
      "description": "Calculate string tension for a specific guitar configuration and tuning. Returns per-string tension in lbs, balance score, gauge suitability flags, target relief, and any setup warnings.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "scale_bass_mm": { "type": "number" },
          "scale_treble_mm": { "type": "number" },
          "string_set_id": { "type": "string" },
          "tuning_preset_id": { "type": "string" },
          "custom_tuning_hz": {
            "type": "array",
            "items": { "type": "number" },
            "description": "Per-string frequencies in Hz, overrides preset"
          }
        },
        "required": ["scale_bass_mm", "string_set_id"]
      }
    },
    {
      "name": "get_problem_detail",
      "description": "Retrieve the full record for a specific problem by ID (e.g. P-02, T-03). Returns all fields including fix steps, off-ramp guidance, and technical parameter references.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "problem_id": { "type": "string", "description": "Problem code, e.g. 'P-02'" }
        },
        "required": ["problem_id"]
      }
    },
    {
      "name": "get_string_sets",
      "description": "Query the string tension database for available string sets. Filter by brand, gauge, instrument type.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "brand": { "type": "string" },
          "gauge_label": { "type": "string" },
          "instrument": { "type": "string", "enum": ["electric", "acoustic", "bass"] }
        }
      }
    },
    {
      "name": "get_tuning_presets",
      "description": "Return available tuning presets with per-string notes and frequencies.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "slug": { "type": "string", "description": "Optional: return a single preset by slug, e.g. 'drop_c'" }
        }
      }
    },
    {
      "name": "guitar_optimise",
      "description": "Create or continue an optimisation session for a specific guitar and preference target. The engine evaluates candidate adjustments against declared constraints and a preference target, ranks them by score improvement per unit of effort, and returns the recommended adjustment sequence. Corresponds to the full OptimisationSession schema defined in setup_optimisation_engine.md. Sessions are persistent — pass a session_id to continue an existing session after physical adjustments have been made.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "session_id": {
            "type": "string",
            "description": "UUID of an existing session to continue. Omit to start a new session."
          },
          "setup_state": {
            "type": "object",
            "description": "Current measured parameter state. See SetupState schema in setup_optimisation_engine.md §3.1."
          },
          "constraints": {
            "type": "object",
            "description": "Which parameters are locked, adjustable, or soft. See ConstraintSet schema in setup_optimisation_engine.md §3.2."
          },
          "preference_target": {
            "type": "object",
            "description": "Target feel dimensions and weights. See PreferenceTarget schema in setup_optimisation_engine.md §3.3."
          },
          "export_setup_sheet": {
            "type": "boolean",
            "default": false,
            "description": "If true, returns a plain-language setup sheet listing target values per parameter in order of adjustment."
          }
        },
        "required": ["setup_state"]
      }
    }
  ]
}
```

### 6.2 MCP server implementation

The MCP server is a thin adapter. For each tool call it:

1. Validates input against the JSON schema
2. Routes to the same handler function used by the REST endpoint
3. Returns the response as an MCP tool result

There is no separate business logic in the MCP layer. The entire implementation is:

```
MCP server process
  → receives tool_call message
  → validates args against schema
  → calls shared handler(args)
  → wraps result in MCP tool_result envelope
  → returns
```

Implementation time once REST is complete: approximately one day for a Node.js or Python MCP server wrapping existing handlers.

---

## 7. Context Handling

### 7.1 Stateless by design

The API stores no user state between calls. Context travels with each request. This is the correct default because:

- The knowledge is about guitars and playing problems, not about users
- No user data is retained, eliminating GDPR/privacy surface
- Any agent can call any endpoint without prior registration or session establishment
- Horizontal scaling is trivial — no session affinity required

### 7.2 What context enables

Without context, `/query` returns the three most semantically similar problem records to the query text. Useful, but broad.

With context, the query engine can:

- Exclude problems that require a skill tier the player has ruled out (`skill_tier_max`)
- Run the tension model and flag problems that apply to the current configuration
- Filter problem groups that are irrelevant to the instrument type
- Boost problems that are consistent with the stated tuning (e.g. T-03 for any query that mentions retuning)

### 7.3 Recommended agent behaviour

An agent building a setup assistant should maintain its own context object across a conversation and pass it with each API call. The context accumulates as the user provides information:

```
Turn 1: "my guitar buzzes"
→ POST /query { query: "guitar buzzes" }
→ Returns P-01, P-02, P-03

Turn 2: "only on open strings"
→ POST /query { query: "buzzes only on open strings" }
→ Returns P-01 with high confidence

Turn 3: "I play pretty hard"
→ POST /query { query: "open string buzz", context: { attack_intensity: "heavy" } }
→ Returns P-01, also surfaces P-06

Turn 4: "I'm using 10-46 strings tuned to Drop C"
→ POST /tension { string_set_id: "...", tuning_preset_id: "drop_c", ... }
→ Returns T-01 flag on low E: tension 8.2 lbs is below threshold
→ Agent now has a richer diagnosis: P-01 for the immediate buzz + T-01 for the tuning/gauge mismatch
```

This progressive context accumulation happens entirely in the agent. The API remains stateless.

---

## 8. Implementation Sequence

### Phase 1 — Data layer (week 1–2)

Serialise the problem records, tool catalogue, and tuning presets from the companion documents into a structured data format (JSON files or a lightweight database such as SQLite or Postgres). This is the work of converting the specification in the companion documents into machine-readable records.

Deliverables:
- 26 `ProblemRecord` objects fully populated
- Tool catalogue (18 tools from `setup_tools_reference.md`)
- 10 tuning presets with per-string frequencies
- String sets for MVP brands (D'Addario, Ernie Ball, Elixir) with unit weights

Estimated records: ~400 rows total across all tables at MVP scope.

### Phase 2 — Query engine (week 2–3)

Implement the embedding-based symptom match and context filtering logic.

- Pre-embed `symptom_plain` and `symptom_alt` for all 26 problem records using `text-embedding-3-small` (batch call, one-time)
- Store embeddings alongside the records
- Implement cosine similarity lookup (no external vector DB needed at this scale — a simple in-memory array sort is sufficient)
- Implement context filtering and tension-based re-ranking
- Implement the tension formula: `T = UW × (2 × L × f)²`

The tension calculation is pure arithmetic — no external dependencies.

### Phase 3 — REST API (week 3–4)

Expose the query engine as HTTP endpoints. Any standard web framework works (Express, FastAPI, Hono, etc.).

Endpoints to ship at MVP:
- `POST /query`
- `GET /problems/:id`
- `POST /tension`
- `GET /strings`
- `GET /tunings`
- `GET /tools`

Add API key authentication, basic rate limiting, and response caching for static endpoints (`/tunings`, `/tools`, `/strings`).

### Phase 4 — MCP server (week 4–5, parallel with REST hardening)

Wrap the REST handlers in an MCP server. The MCP protocol is simple — the server listens for JSON-RPC calls, routes to handlers, and returns results. No new business logic.

Ship the five tool definitions from section 6.1. Register the server with the MCP directory if public access is desired.

---

## 9. Operational Cost

### 9.1 Infrastructure

At MVP scale (26 problem records, ~400 total data rows, modest traffic), the infrastructure requirement is minimal.

| Component | Option | Monthly cost estimate |
|---|---|---|
| API server | Single small instance (e.g. Railway, Render, Fly.io) | $5–20 |
| Database | SQLite (file, same instance) or Postgres (managed small) | $0–15 |
| Embeddings (one-time) | Pre-embed 26 × ~5 text fields via `text-embedding-3-small` | < $0.01 total |
| Embeddings (query-time) | Each `/query` call embeds the query string | ~$0.000002 per call |
| MCP server | Same instance as REST or a separate small process | $0–10 |
| **Total MVP** | | **$10–45/month** |

Re-embedding is only needed when problem records are updated — a rare event. Query-time embedding cost is negligible at any realistic traffic volume: 100,000 queries/month would cost approximately $0.20 in embedding API calls.

### 9.2 Embedding model options

| Model | Cost per 1M tokens | Latency | Self-hostable |
|---|---|---|---|
| `text-embedding-3-small` (OpenAI) | $0.02 | ~50ms | No |
| `text-embedding-3-large` (OpenAI) | $0.13 | ~80ms | No |
| `all-MiniLM-L6-v2` | $0 | ~5ms (CPU) | Yes |
| `nomic-embed-text` | $0 | ~10ms (CPU) | Yes |

For 26 records and low traffic, `all-MiniLM-L6-v2` self-hosted is a reasonable choice — it eliminates the embedding API dependency entirely and runs comfortably on any small instance. The quality difference vs. `text-embedding-3-small` is minimal at this scale.

### 9.3 What scales independently

As the string tension database grows (more brands, more sets), the string data table grows but does not affect query latency — string lookups are indexed key-value fetches, not vector searches.

As traffic grows, the bottleneck is the embedding call per `/query` request. Options at scale:

- Cache embeddings for common query strings (most guitar problems are phrased similarly)
- Switch to self-hosted embedding model to eliminate external API latency and cost
- Add a second instance behind a load balancer — the API is stateless so this is trivial

Neither the tension calculation nor the problem record lookup has meaningful scaling constraints at any realistic guitar-site traffic level.

### 9.4 What costs nothing

- The tension formula (`T = UW × (2 × L × f)²`) is arithmetic — no API, no model
- Problem record lookups are indexed database reads
- Tuning presets and tool data are static — cached at startup, served from memory
- MCP server adds no meaningful compute cost

### 9.5 What costs more as usage grows

- Embedding API calls scale linearly with query volume (mitigated by caching or self-hosting)
- If the string tension database grows to full brand coverage (~5,000+ string sets), the strings endpoint benefits from a proper search index
- Log storage if full request/response logging is enabled — scope this carefully at the start

---

## 10. What This Enables Agents to Do

A well-built agent using this API can handle the full range of queries your site addresses, without you hosting any inference:

**Symptom triage:** "My guitar buzzes when I bend" → matched to P-04 and P-14, with skill tier and tool requirements, so the agent can immediately tell the user what this probably is, whether they can fix it, and what it will cost.

**Configuration-aware diagnosis:** Given a guitar model, string set, and tuning, the tension engine flags which problems are structurally likely before the user even describes a symptom. An agent that knows a user is playing 10-46 strings in Drop C can surface T-01 proactively.

**Gauge recommendations:** "What strings should I use for Drop B on a 25.5" scale?" → `POST /tension` iterated across candidate string sets from `/strings` → ranked by whether all six strings fall within the 8–18 lb comfort range.

**Progressive diagnosis:** The agent accumulates context across a conversation and passes it with each call. The API re-ranks results as context fills in. The user experience is a conversation that narrows toward a diagnosis, not a static lookup.

**Escalation handling:** Off-ramp conditions in every problem record let the agent confidently tell the user when to go to a tech, what to tell the tech, and what to expect to pay — without hedging or deflecting.

---

## 11. Document Relationships

```
agent_access_spec.md (this document)
  ↓ exposes data from
guitar_setup_parameters_v3.md   — problem index, parameter definitions, problem records schema
string_tension_database.md      — unit weights, tension formula, tuning presets
setup_tools_reference.md        — tool catalogue, cost ranges, risk_if_wrong, learning_value
setup_guide_architecture.md     — ProblemRecord schema, SkillTier definitions, component specs
setup_optimisation_engine.md    — OptimisationSession, SetupState, ConstraintSet, PreferenceTarget schemas

  ↓ is positioned as the agent-facing access layer for
feature_layers.md — Layer 1–4 user journeys (agents replicate the same journey via API)

  ↓ serves agents that present through
Any MCP-capable agent (Claude, GPT-4o, custom)
Any HTTP-capable agent or integration
```

The companion documents are not modified by this spec. This document defines a read interface over them, plus the optimisation session persistence layer over `setup_optimisation_engine.md`. The underlying knowledge is unchanged; this layer makes it machine-queryable.

---

*Document version 1.1*
*v1.1 changes: `feature_layers.md` and `setup_optimisation_engine.md` added to companion docs; `guitar_optimise` MCP tool added (§6.1); `POST /optimisation/session` endpoint added (§5.2); §11 Document Relationships expanded.*
*Companion documents: `guitar_setup_parameters_v3.md` · `string_tension_database.md` · `setup_guide_architecture.md` · `setup_tools_reference.md` · `feature_layers.md` · `setup_optimisation_engine.md`*
