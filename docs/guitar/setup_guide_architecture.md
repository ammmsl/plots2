# Setup Guide Architecture
## Forward-Facing Layer — Content Model, Data Structures & Document Interplay

**Version:** 1.0
**Companion documents:** [`guitar_setup_parameters_v3.md`](./guitar_setup_parameters_v3.md) · [`string_tension_database.md`](./string_tension_database.md) · [`setup_tools_reference.md`](./setup_tools_reference.md)

---

## 1. Purpose & Design Philosophy

The setup guide is the public face of the site. It is the layer that most users will ever see, and for most of them it will be enough. The simulation tool — the full parameter model with its tension engine, buzz risk map, and intonation diagnostics — is the destination for a small minority of users who want to fully understand and optimise their instrument. The guide is what brings them there.

The journey the site creates is a deliberate funnel:

```
Symptom recognition
  → Cause understanding
    → Fix execution (or confident referral)
      → Skill development
        → Physics comprehension
          → Simulation use
```

Each stage is a genuine resting place. A user who enters because their guitar buzzes and exits knowing how to adjust their truss rod has gotten full value from the site. A user who exits having run a complete tension-driven setup simulation and understood why every parameter sits where it does has gotten something different but not more legitimate. The guide must serve both without condescending to either.

### What this document covers

- The content architecture of the forward-facing layer and how it maps to user journey stages
- The data structures that drive the setup guide component
- How those structures connect to the existing technical documentation
- What the guide renders, what it links to, and what it hands off to the simulation tool
- The progressive disclosure model — what is visible at each stage and what is behind the next door

---

## 2. The User Journey in Four Stages

### Stage 1 — Symptom recognition (entry point)

The user arrives with a problem, not a concept. They know their guitar buzzes. They know it's hard to play. They don't know the word "relief" or the concept of vibration amplitude.

The entry surface is a symptom picker — plain language, no jargon. Users select what they are experiencing, not what they think the cause is. The guide meets them exactly there.

**What this stage requires:**
- A symptom vocabulary mapped to problem IDs from the parameter model
- No assumptions about prior knowledge
- No tools, no measurements — just recognition

**What this stage produces:**
- A ranked list of probable causes for the selected symptom
- A skill tier assessment for each likely fix
- An immediate answer to: *can I fix this myself?*

---

### Stage 2 — Cause understanding

The user now knows what the probable cause is. This stage explains why. Not how to fix it yet — why it happens. The explanation is written in plain language first, with physics available one layer deeper for the curious.

This is where the tension model's explanatory power becomes visible to a non-technical user. The fact that downtuning reduces neck load, which straightens the neck, which reduces mid-neck clearance, which causes buzz — this chain is expressible in plain English without a single formula. The formula is available behind a toggle for the user who wants it.

**What this stage requires:**
- Plain-language cause description per problem
- A one-sentence physical principle linking cause to effect
- Optional: the formula or parameter reference that makes it precise
- A visual indication of where in the guitar the problem lives (fret zone, nut, saddle, neck)

**What this stage produces:**
- User understanding of the causal chain
- Enough context to evaluate the fix options
- First encounter with the physics framing (optional, never forced)

---

### Stage 3 — Fix execution (or confident referral)

The user understands the cause and is ready to act. This stage tells them exactly what to do, what tools they need, what it will cost in time and money, what the risk of getting it wrong is, and whether it's worth learning to do themselves.

This is the most operationally dense stage. It maps directly to the categorisation developed in this project: skill tier, reversibility, tool cost, off-ramp condition, and learning value. Every problem has a complete record in the data structure that drives this stage.

A key design principle: the off-ramp is presented without embarrassment. Some problems should go to a tech. The site says so clearly and helps the user prepare for that conversation — which tools the tech will use, what the fix involves, what a reasonable price range looks like. A user who leaves this stage to book a setup has still been served well.

**What this stage requires:**
- Full problem record (see §4 data structures)
- Step-by-step fix description (where applicable)
- Tool list with cost indication
- Explicit off-ramp condition with suggested action
- Learning pathway signal — is this worth developing as a skill?

**What this stage produces:**
- Either: a user who successfully makes the adjustment
- Or: a user who makes an informed decision to seek a tech, with context for that conversation
- In both cases: a user who understands what happened and why

---

### Stage 4 — Skill development and simulation

Users who reach this stage have fixed one or more problems and are now building a mental model of how setup works. They understand that their guitar is a physical system, that its parameters interact, and that tuning and string gauge are not cosmetic choices — they cascade through the whole instrument.

This stage has two exits:

1. **Skill building** — the site provides learning pathways for the skills that are worth developing (truss rod work, nut filing, intonation, saddle adjustment). Each pathway is self-contained but linked to the relevant section of the technical documentation for the user who wants to go deeper.

2. **Simulation entry** — the user is introduced to the full parameter model. They already understand the concepts from the earlier stages. The simulation is now the logical next step — not a wall of physics, but a tool they are prepared to use.

**What this stage requires:**
- Skill pathway content per learnable skill
- A clear on-ramp to the simulation tool
- The simulation tool itself — the full v3.0 parameter model with tension engine

---

## 3. Content Architecture

### 3.1 Layer map

```
LAYER 0 — Entry surface
  Symptom picker (plain language)
  → routes to: Problem record

LAYER 1 — Problem record (the guide's core unit)
  Problem ID, name, symptom description
  Cause (plain language + optional physics)
  Skill tier badge
  Cost summary (time / tools / money)
  Reversibility flag
  → links to: Fix detail (Layer 2)
  → links to: Cause deep-dive (Layer 2)
  → links to: Related problems

LAYER 2 — Fix detail
  Step-by-step procedure
  Tool list (links to setup_tools_reference.md data)
  Off-ramp condition and suggested action
  Learning pathway signal
  → links to: Skill pathway (Layer 3)
  → links to: Technical parameter reference (companion docs)

LAYER 3 — Skill pathway
  What this skill is
  When you need it (problem list)
  What to practise on first
  What to measure, what tools, what targets
  Common mistakes and how to recover
  → links to: Parameter reference in v3.0

LAYER 4 — Simulation entry
  Guitar model selector
  String set selector (tension DB)
  Tuning selector (tension DB)
  Player profile inputs
  → runs: Full tension engine + parameter model
  → outputs: Buzz risk map, intonation map, setup recommendations
```

### 3.2 Progressive disclosure rules

Each layer is accessible from the one above it — never required. A user who wants only Layer 1 (cause and tier assessment) never sees a formula. A user who wants Layer 4 (simulation) passes through the earlier layers first, accumulating the conceptual framework they need.

The physics is never hidden — it is always reachable one step further in. But it is never the first thing a user sees.

The simulation tool requires no prior knowledge of the technical documentation to use — the UI guides parameter entry, the tension engine runs invisibly, and the outputs are in plain language with optional detail. But a user who has read through the layers will get substantially more from it.

---

## 4. Data Structures

### 4.1 ProblemRecord

The core unit. One record per problem ID. These are derived from `guitar_setup_parameters_v3.md §4` (Problem Index) and the categorisation session.

```typescript
interface ProblemRecord {
  // Identity
  id: string;                    // "P-01" through "P-21", "T-01" through "T-05"
  name: string;                  // "Open-string buzz"
  group: ProblemGroup;           // "buzz" | "action" | "intonation" | "bending" | "feel" | "tension" | "structural"

  // Entry surface — plain language
  symptom_plain: string;         // What the player experiences, no jargon
  symptom_alt: string[];         // Alternative phrasings for the symptom picker

  // Cause — two depths
  cause_plain: string;           // Plain English. No parameters, no formulas.
  cause_principle: string;       // One-sentence physical principle. Still no formula.
  cause_technical: string;       // Parameter names, formula references. Links to companion docs.
  cause_parameters: string[];    // ["relief_bass", "tension[1..6]"] — links to v3.0 §2

  // Skill & cost
  skill_tier: SkillTier;         // "t0" | "t1" | "t2" | "t3" | "t4"
  time_cost: TimeCost;           // "none" | "low" | "medium" | "high"
  tool_cost_range: [number, number] | null;  // [min, max] in USD; null if no tools needed
  tools_required: ToolRef[];     // References to SetupTool records (see §4.3)

  // Fix
  fix_plain: string;             // What to do, plain language
  fix_steps: FixStep[] | null;   // Null for structural/luthier problems
  off_ramp: OffRamp | null;      // Null if no escalation path

  // Reversibility
  reversible: boolean | null;    // null = N/A (no physical change)
  irreversible_note: string | null;  // What goes wrong if you overshoot

  // Learning
  learnable: boolean;
  learn_note: string | null;     // Why it's worth learning, or why it isn't
  skill_pathway_id: string | null;  // FK to SkillPathway record (see §4.4)

  // Cross-references to companion docs
  doc_refs: DocRef[];            // Links into v3.0, string_tension_database.md, tools ref
}
```

```typescript
type ProblemGroup =
  | "tension"      // T-01 to T-05 — require tension engine
  | "buzz"         // P-01 to P-06
  | "action"       // P-07 to P-09
  | "intonation"   // P-10 to P-13
  | "bending"      // P-14 to P-15
  | "feel"         // P-16 to P-20
  | "structural";  // P-21

type SkillTier =
  | "t0"   // No skill — string swap or model input change
  | "t1"   // Basic — reversible mechanical adjustment
  | "t2"   // Intermediate — cuts material, learnable with care
  | "t3"   // Advanced — irreversible, significant consequences
  | "t4";  // Luthier only — structural, no setup tool addresses this

type TimeCost = "none" | "low" | "medium" | "high";
```

### 4.2 SkillTier (display metadata)

Used by the UI to render tier badges consistently.

```typescript
interface SkillTierMeta {
  id: SkillTier;
  label: string;              // "No skill" | "Basic" | "Intermediate" | "Advanced" | "Luthier only"
  color_token: string;        // Design token for badge color
  description: string;        // One sentence — what this tier means to the user
  reversibility_note: string; // "All adjustments can be undone" / "Material cuts are permanent" etc.
}

const SKILL_TIERS: Record<SkillTier, SkillTierMeta> = {
  t0: {
    id: "t0",
    label: "No skill needed",
    description: "A string swap or a change in how you use the tool. No physical work on the guitar.",
    reversibility_note: "Fully reversible.",
  },
  t1: {
    id: "t1",
    label: "Basic",
    description: "Mechanical adjustments that can be reversed if you overshoot. Learn as you go.",
    reversibility_note: "Reversible — you can go back.",
  },
  t2: {
    id: "t2",
    label: "Intermediate",
    description: "Involves cutting or filing material. Learnable, but you cannot add material back.",
    reversibility_note: "Partially irreversible — overshoot requires replacement.",
  },
  t3: {
    id: "t3",
    label: "Advanced",
    description: "Irreversible work with meaningful consequences if done incorrectly. Off-ramp recommended.",
    reversibility_note: "Irreversible.",
  },
  t4: {
    id: "t4",
    label: "Luthier only",
    description: "Outside setup parameter scope. No adjustment tool addresses this. Structural assessment required.",
    reversibility_note: "Structural — cannot be resolved with setup tools.",
  },
};
```

### 4.3 ToolRef and SetupTool

Tools referenced in problem records. These are the same tools catalogued in `setup_tools_reference.md` — the component reads from a structured version of that document's data.

```typescript
interface ToolRef {
  tool_id: string;            // FK to SetupTool.id
  role: string;               // How this tool is used for this specific problem
  required: boolean;          // True if the fix cannot proceed without it
}

interface SetupTool {
  id: string;                 // "feeler_gauge" | "truss_rod_wrench" | "nut_files" | etc.
  name: string;               // "Feeler gauge set"
  category: ToolCategory;
  description: string;        // What it does
  cost_range: [number, number]; // [min, max] in USD
  notes: string | null;       // Specification notes (e.g. required precision)
  problems: string[];         // Problem IDs this tool is used in
  // Sourced from setup_tools_reference.md §1–§6
}

type ToolCategory =
  | "measurement"       // Feeler gauge, straight edge, fret rocker, action gauge, calipers, tuner, capo
  | "truss_rod"         // Truss rod wrenches
  | "nut_work"          // Nut files, lubricant, blanks, adhesive, sandpaper
  | "saddle_bridge"     // Bridge-type-specific adjustment tools
  | "fret_work"         // Levelling beam, crowning file, fret erasers, masking tape
  | "general";          // String winder, cutters, guitar support, magnification
```

The tool data is the structured equivalent of `setup_tools_reference.md §1–§6`. The tools reference document remains the canonical prose description; this structure is the machine-readable version that the component queries.

### 4.4 FixStep

Ordered steps within a fix procedure. Only present for problems where the user is actually doing the work (t1, t2 tier problems with `fix_steps` populated).

```typescript
interface FixStep {
  order: number;
  instruction: string;        // Plain-language instruction
  measurement?: {             // Present when the step requires measurement
    parameter: string;        // The parameter being measured, e.g. "relief_bass"
    tool_id: string;          // Which tool to use
    target: string;           // What to aim for, e.g. "0.10–0.30 mm"
    how_to_measure: string;   // Where to place the tool, what to read
  };
  warning?: string;           // What to watch out for — risk of this step
  settling_time?: string;     // e.g. "Wait 10–15 minutes before measuring again" (truss rod)
  doc_ref?: DocRef;           // Link to parameter definition if step references a model parameter
}
```

### 4.5 OffRamp

When the fix should go to a tech or luthier, or when the user risks crossing into irreversible territory.

```typescript
interface OffRamp {
  trigger: string;            // Condition that makes this the right call
  who: "tech" | "luthier";    // Tech = setup work; luthier = structural
  why: string;                // Why this is the off-ramp, not a failure
  what_to_tell_them: string;  // What the user should communicate to the professional
  cost_indication: string;    // Rough cost range or time range
  prepare: string[];          // What the user can do to prepare (e.g. "note your current tuning and string gauge")
}
```

### 4.6 SkillPathway

For learnable skills (problems where `learnable = true` and `skill_pathway_id` is set). One pathway can cover multiple problems — truss rod work covers P-02, P-07, T-03.

```typescript
interface SkillPathway {
  id: string;                 // "truss_rod" | "nut_filing" | "intonation" | "saddle_height" | "nut_replacement" | "fret_levelling"
  name: string;               // "Truss rod adjustment"
  covers_problems: string[];  // Problem IDs this skill addresses
  tier: SkillTier;            // The tier of work this pathway teaches
  
  what_is_it: string;         // Plain explanation of what the skill involves
  when_you_need_it: string;   // Conditions that call for this skill
  
  before_you_start: string[]; // Prerequisites — what to have, what to confirm
  practice_advice: string;    // What to practise on first, how to build confidence
  
  steps: SkillStep[];         // The learning progression
  common_mistakes: Mistake[]; // What goes wrong and how to recover
  
  tools_required: ToolRef[];  // The tools this skill needs
  
  // Links into companion docs
  parameter_refs: string[];   // Parameters from v3.0 that this skill directly controls
  doc_refs: DocRef[];         // Section references in companion docs
}

interface SkillStep {
  order: number;
  name: string;
  description: string;
  what_you_are_learning: string;  // The underlying concept, not just the action
  success_criterion: string;      // How to know you've done this right
}

interface Mistake {
  what_happens: string;
  why: string;
  recovery: string;           // Can this be recovered from, and how?
}
```

### 4.7 DocRef

Cross-references into the companion technical documents. These are used throughout the data structures wherever a concept connects to its formal definition.

```typescript
interface DocRef {
  doc: "guitar_setup_parameters_v3" | "string_tension_database" | "setup_tools_reference";
  section: string;            // e.g. "§2.7", "§6.3", "§1.1"
  anchor: string;             // URL fragment for deep-linking
  label: string;              // Display text for the link
  note?: string;              // Why this reference is relevant
}
```

### 4.8 SymptomEntry

The entry point. Maps plain-language symptoms to problem IDs. This is what the symptom picker queries.

```typescript
interface SymptomEntry {
  phrase: string;             // "My guitar buzzes when I play open chords"
  keywords: string[];         // ["buzz", "open", "chords", "first position"]
  problems: {
    id: string;
    probability: "primary" | "secondary";  // Primary = most likely cause
    condition?: string;       // Optional qualifier — "if buzz clears above fret 5"
  }[];
}
```

The symptom entry table is intentionally large — it covers many phrasings of the same experience because users describe the same problem in different words. Keyword matching is fuzzy; probability ranking is based on frequency in real setup contexts.

---

## 5. Skill Tier Reference Table

This table is the master reference for categorising problems. It is the source of truth for every `skill_tier`, `learnable`, `reversible`, and `off_ramp` field in the `ProblemRecord` data.

| ID | Name | Tier | Time | Tool cost | Reversible | Learnable | Off-ramp condition |
|----|------|------|------|-----------|------------|-----------|-------------------|
| T-01 | String feels floppy | t0 | None | $0 (string set) | Yes | Yes | — |
| T-02 | Guitar hard to play | t0 | None | $0 (string set) | Yes | Yes | — |
| T-03 | Buzz after retuning | t1 | Medium | $30–60 | Yes | Yes | — |
| T-04 | Intonation shifts after retuning | t1 | Low | $50–120 | Yes | Yes | — |
| T-05 | Bending feel changes with tuning | t0 | None | $0 | Yes | Yes | — |
| P-01 | Open-string buzz | t1 | Low | $20–40 | No | Yes | Slot filed too deep → nut replacement |
| P-02 | Buzz across the whole neck | t1 | Low–Medium | $30–60 | Yes | Yes | — |
| P-03 | Buzz in first position only | t1 | Low | $30–50 | No | Yes | High fret confirmed → fret dress |
| P-04 | Buzz at high frets only | t2 | Medium | $80–150 | No | Yes | Not confident with levelling → tech |
| P-05 | Buzz on wound strings only | t1 | Low | $30–60 | Yes | Yes | — |
| P-06 | Buzz when playing hard | t0 | None | $0 | Yes | Yes | — |
| P-07 | Action too high everywhere | t1 | Medium | $40–80 | Yes | Yes | Neck angle root cause → luthier |
| P-08 | Action high in first position only | t1 | Low | $30–50 | No | Yes | Slot filed too deep → nut replacement |
| P-09 | Action too low | t1 | Low | $20–40 | Yes | Yes | — |
| P-10 | Sharp intonation everywhere | t1 | Low–Medium | $50–120 | Yes | Yes | — |
| P-11 | Flat intonation everywhere | t1 | Low | $50–120 | Yes | Yes | — |
| P-12 | Intonation off at specific frets | t3 | Medium–High | $80–200 | No | No | Fret dress or PLEK → tech/luthier |
| P-13 | Open sharp, 12th fret correct | t2 | Medium | $60–100 | No | Yes | Not confident fitting blank → tech |
| P-14 | Notes choke when bending | t2 | Medium | $80–150 | No | Yes | Not confident with levelling → tech |
| P-15 | Bends require unexpected force | t0 | None | $0 (string set) | Yes | Yes | — |
| P-16 | Uneven feel across strings | t0 | None | $0 (string set) | Yes | Yes | — |
| P-17 | Neck feels different bass vs treble | t2–t4 | Medium–High | $50–80 | No | No | Twist > 0.15 mm → luthier |
| P-18 | Tuning instability after bending | t1 | Low–Medium | $30–60 | No | Yes | Unrecoverable slots → nut replacement |
| P-19 | Dead spots / weak sustain | t4 | N/A | $0 | N/A | No | Outside setup scope entirely |
| P-20 | String rattle behind fretted note | t1 | Low | $20–50 | No | Yes | — |
| P-21 | Truss rod at or near limit | t4 | High | Luthier cost | No | No | Mandatory luthier referral |

### Learnable skills inventory

The following skills are worth developing as repeatable competencies. Each covers multiple problems and pays dividends across future setups.

| Skill | Pathway ID | Problems covered | Tier | Key risk |
|-------|-----------|-----------------|------|----------|
| Truss rod adjustment | `truss_rod` | P-02, P-07, T-03 | t1 | Over-tightening; going past the flat datum |
| Saddle height adjustment | `saddle_height` | P-05, P-07, P-09 | t1 | None — fully reversible |
| Intonation (saddle compensation) | `intonation` | P-10, P-11, T-04 | t1 | None — fully reversible |
| Nut slot filing | `nut_filing` | P-01, P-03, P-08, P-18, P-20 | t2 | Filing too deep — requires nut replacement |
| Nut replacement | `nut_replacement` | P-01, P-08, P-13, P-18 | t2 | Fitting errors; slot angles |
| Fret levelling & fall-off | `fret_levelling` | P-03, P-04, P-14 | t2 | Removing too much crown height |

---

## 6. Interplay with Companion Documents

The forward-facing guide is not a separate content system — it is a presentation layer over the existing technical documentation. Every field in the data structures above traces back to a specific section of a companion document. This section makes those connections explicit.

### 6.1 From `guitar_setup_parameters_v3.md`

| v3.0 section | Guide usage |
|---|---|
| §2.1 Scale & Fret Geometry | Referenced in tension-related problem causes (T-01 to T-05). Scale length explained in plain language as "how long the vibrating string is" — a longer scale produces higher tension at the same gauge and tuning. |
| §2.4 String Set | The string selector in Layer 4 (simulation). In the guide, the string set concept is introduced via T-01/T-02 (floppy or stiff strings). `unit_weight` is never shown to the user — only its effect (tension). |
| §2.5 Tuning | The tuning selector in Layer 4. In the guide, introduced via T-03/T-04/T-05 — the effects of tuning change on neck geometry and bend feel. |
| §2.6 Nut | Direct source for P-01, P-08, P-13, P-18, P-20 problem records. `nut_slot_height[n]` is the parameter behind all of these; the guide describes it as "how deep the string sits in the nut slot". |
| §2.7 Neck Setup | The flat-neck datum workflow is the procedural backbone of the truss rod skill pathway. `flat_neck_confirmed`, `relief_from_flat`, and `rod_state` map directly to FixStep records in P-02 and T-03. `rod_state` values (`near_limit`, `at_limit`) are the trigger for P-21 escalation. |
| §2.8 Bridge & Saddle | Source for P-10, P-11, P-13 (compensation) and P-07, P-09 (saddle height). `compensation_inharmonicity` (post-MVP) is referenced in T-04. |
| §2.9 Fret-Plane Topology | Source for P-03 (high fret in first position), P-04 (high frets, insufficient fall-off), P-12 (intonation from uneven frets), P-14 (bend choking). `falloff_amount` and `falloff_onset_fret` are described in the guide as "the planned drop in fret height past the 12th fret that gives your bends room to breathe". |
| §2.10 Player Profile | `attack_intensity` is the source for P-06. `target_action_bass/treble` feed into P-07, P-09, P-14. In the guide, attack intensity is introduced as "how hard you typically play" in the player profile questionnaire. |
| §2.11 Tension Profile | The computed outputs here are what the guide's tension-specific problems (T-01 to T-05) explain in plain language. `gauge_suitability[n]` flags are the mechanism behind T-01/T-02 recommendations. `rezero_required` is the trigger for the tuning-change prompt in T-03. |
| §4.7 Problem P-21 | The `rod_state` field and its five values (`loose`, `light`, `mid`, `near_limit`, `at_limit`) are directly encoded in the P-21 off-ramp record. The diagnostic table in §4.7 maps to the `OffRamp.trigger` and `OffRamp.what_to_tell_them` fields. |

### 6.2 From `string_tension_database.md`

| Tension DB section | Guide usage |
|---|---|
| §2 — Physics | The guide's plain-language cause descriptions for T-01 to T-05 are all derived from the tension formula `T = UW × (2 × L × f)²`. The formula itself appears only in Layer 2 (optional physics toggle) and Layer 4 (simulation). |
| §4.1 — Tuning presets | The tuning selector in the symptom picker and simulation. Used when a user identifies "I changed tuning" as the context for a problem. |
| §4.3 — Tension change per semitone | The 12.2% per semitone / 20.6% per whole step figures appear in the plain-language explanation of T-03 (buzz after retuning). Presented as "dropping a whole step reduces the pull on the neck by about a fifth". |
| §6.4 — Gauge suitability flagging | The < 7 lbs and > 20 lbs thresholds are the diagnostic basis for T-01 and T-02. In the guide, these are presented as "this string is too light for this tuning" and "these strings are too heavy" without surfacing the pound values unless the user requests the physics detail. |
| §6.5 — Tension balance score | The balance score formula feeds P-16. The guide presents this as "your strings don't all pull with the same force — some feel stiff, others floppy" and recommends a balanced set. |

### 6.3 From `setup_tools_reference.md`

The tools reference document is the canonical description of every physical tool. The guide's `SetupTool` data structure is the machine-readable version of the same content. The interplay is:

- `setup_tools_reference.md §8` (quick-reference tool map) is the source for `SetupTool.problems[]` — which tools address which problems.
- `setup_tools_reference.md §7` (tools out of scope) is the source for `OffRamp.who = "luthier"` cases — these are problems where the tools reference explicitly states no setup tool addresses the issue.
- The per-tool `Workflow note` callouts (feeler gauge, straight edge) map to `FixStep.measurement` fields — they define where to place the tool, what to read, and what settling time to allow.

### 6.4 What the guide adds that the companion docs do not contain

The companion documents are complete technical references. They define parameters, calculate tension, catalogue tools, and diagnose problems precisely. What they do not contain — and what this document adds — is:

- **Skill tier assessment** — no companion doc rates problems by the skill level required to fix them.
- **Reversibility classification** — no companion doc distinguishes between adjustments that can be undone and material cuts that cannot.
- **Learning value judgement** — no companion doc advises on which skills are worth developing vs. which should go to a tech.
- **Off-ramp content** — the companion docs flag escalation (P-21, P-17) but do not describe how to communicate with a professional or what to expect.
- **Symptom vocabulary** — the companion docs name problems formally. The guide translates those names into the language players actually use.
- **Progressive disclosure structure** — the companion docs are flat. The guide layers the same information so that each user sees only as much as they are ready for.

---

## 7. Component Specification

### 7.1 ProblemCard component

The primary rendering unit for the guide. One card per problem, with progressive disclosure built in.

```
ProblemCard
├── Header
│   ├── problem.id (small, secondary)
│   ├── problem.name (primary heading)
│   └── Badges
│       ├── SkillTierBadge (tier label + color)
│       ├── ReversibilityBadge (if !reversible)
│       └── LearnableBadge (if learnable)
│
├── Symptom (always visible)
│   └── problem.symptom_plain
│
├── Cause summary (always visible)
│   └── problem.cause_plain
│
├── Cost row (always visible)
│   ├── TimeCostIndicator
│   ├── ToolCostIndicator (or "no tools needed")
│   └── SkillLabel
│
└── [Expandable section — hidden by default]
    ├── Fix detail
    │   ├── problem.fix_plain
    │   └── FixStepList (if fix_steps present)
    │
    ├── Tool list
    │   └── ToolRef[] (with links to SetupTool)
    │
    ├── Physics toggle (optional)
    │   ├── problem.cause_principle
    │   └── problem.cause_technical (with DocRef links)
    │
    ├── LearnNote (if learnable)
    │   └── problem.learn_note + link to SkillPathway
    │
    └── OffRamp (if present)
        ├── off_ramp.trigger
        ├── off_ramp.why
        ├── off_ramp.what_to_tell_them
        └── off_ramp.cost_indication
```

### 7.2 SymptomPicker component

Entry point. No guitar knowledge required.

```
SymptomPicker
├── Free-text search (fuzzy match against SymptomEntry.keywords)
├── Category filter
│   ├── "It buzzes"
│   ├── "It's hard to play"
│   ├── "It sounds out of tune"
│   ├── "Bending feels wrong"
│   ├── "It feels uneven or inconsistent"
│   └── "I changed something and now it's worse"
│
└── Results
    └── ProblemCard[] (ordered by probability for selected symptom)
```

The category "I changed something and now it's worse" is a direct entry point for T-03 (buzz after retuning), T-04 (intonation shift after retuning), and P-16/T-01/T-02 (feel changes after string swap). This is a high-frequency real-world scenario that the tension model now explains precisely.

### 7.3 SkillTierFilter component

Allows users to filter the full problem index by what they are willing to take on themselves.

```
SkillTierFilter
├── "Show me what I can fix right now" → [t0, t1]
├── "Show me what's worth learning" → [t1, t2, learnable = true]
├── "Show me what needs a tech" → [t3, t4, or off_ramp present]
└── "Show everything" → all
```

### 7.4 TensionDashboard component (Layer 4 — simulation)

The bridge between the guide and the full simulation. This component is the first thing a user sees when they enter the simulation — it is designed to be immediately legible from the guide's conceptual framework, not from the technical documentation.

```
TensionDashboard
├── Guitar selector → loads scale_bass, scale_treble, neck_joint, etc. from guitar model preset
├── String set selector → loads unit_weight[1..6], gauge[1..6] from String Tension DB
├── Tuning selector → loads frequency[1..6] from tuning preset library
│
├── [Live computed output — updates on every change]
│   ├── TensionBar × 6 (one per string)
│   │   ├── bar width = tension in lbs
│   │   ├── color = gauge_suitability (ok / too_slack / too_tight)
│   │   └── label = string note name + tension value
│   ├── BalanceScore indicator
│   ├── TotalTensionReadout
│   └── RezeroCue (if rezero_required)
│
└── [Setup parameters panel — expands from here]
    ├── flat_neck_confirmed checkbox
    ├── rod_state selector (5-point)
    ├── relief_bass / relief_treble sliders
    ├── saddle_height sliders
    ├── nut_slot_height sliders
    └── [Outputs]
        ├── BuzzRiskMap (all frets × all strings)
        ├── IntonationMap (per string)
        └── AdjustmentRecommendations (plain language)
```

The `TensionBar` component is the visual equivalent of the plain-language explanation the user has already read in the guide: "your string tension determines how your guitar responds to everything else." By the time a user reaches this component, they already understand what they are looking at.

---

## 8. The Pedagogical Funnel — How the Layers Work Together

The site's central claim is that understanding physics eliminates trial and error. This is true, but only if the physics is introduced at the right moment in the right form. The funnel is the mechanism that ensures this.

### Entry: problem-first, not concept-first

A user does not arrive thinking "I need to understand string tension." They arrive thinking "my guitar sounds like a typewriter." The guide meets them there. The symptom picker asks nothing about the instrument, the setup, or the physics — it asks what the player is experiencing.

### First payoff: the cause

Once the guide has identified the probable cause, it delivers a plain-language explanation that gives the user a working model of what is happening. This is the first payoff — the user didn't know this before, and now they do. The experience of understanding is more motivating than the solution alone.

### Second payoff: the fix

The fix section gives the user agency. They can act. For t0 and t1 tier problems, they can act immediately with minimal investment. For t2, they can choose to learn or to refer. The off-ramp is presented as a reasonable choice, not a failure.

### Third payoff: skill development

Users who fix their own problems discover that they are capable of more than they thought. The skill pathways capitalise on this discovery. Each pathway is framed not as a tutorial but as a progression — from understanding the concept to executing it confidently. The common mistakes section is particularly important here: it tells the user what can go wrong and how to recover, which converts fear of irreversibility into informed caution.

### Fourth payoff: the simulation

By the time a user is ready for the simulation, they have a working vocabulary for every input they will encounter. `relief_bass` is not a mysterious parameter — it's the thing they just measured with a feeler gauge. `tension[1..6]` is not a formula — it's the force they now understand is different for every string and changes when they retune. The simulation is a natural extension of what they already know, not a leap into unfamiliar territory.

This is the core value proposition: the guide teaches the physics through use, so that by the time a user reaches the simulation, they are using it fluently rather than entering numbers they don't understand into a black box.

---

## 9. Content Additions Required in Companion Documents

The following additions to the companion documents would improve the interplay with the guide and should be tracked as future revisions.

### `setup_tools_reference.md` additions

The current tools reference maps tools to problems and sections accurately. Two columns are missing that the guide data structures require:

| Addition | What it enables |
|---|---|
| `risk_if_wrong` per tool | Populates `FixStep.warning` — the user needs to know what the risk of misusing each tool is |
| `learning_value` rating per tool | Informs the learnable skill inventory — which tools are worth buying and developing vs. which are specialist-only |
| Cost range per tool | Populates `SetupTool.cost_range` — currently implied but not stated explicitly |

### `guitar_setup_parameters_v3.md` additions

| Addition | What it enables |
|---|---|
| Plain-language symptom description per problem (alongside the formal parameter diagnostic) | Reduces translation work in the guide — the formal description and the plain-language version would be co-located |
| `rod_state` to `rod_adjustment` mapping table for common rod types | Populates `FixStep` content for truss rod skill pathway more precisely |

### `string_tension_database.md` additions

| Addition | What it enables |
|---|---|
| Note that `§6.3` (compliance coefficient model) is superseded by v3.0 flat-neck datum approach | Removes a stale cross-reference flagged in v3.0 §7 |
| Plain-language summary of the tension formula for each tuning preset | Enables guide to surface "in Drop C, your low string produces about X lbs of tension" without requiring the user to run the formula |

---

## 10. Document Relationships

```
setup_guide_architecture.md (this document)
  ↓ draws problem records from
guitar_setup_parameters_v3.md §4 (Problem Index)

  ↓ draws tension physics and tuning data from
string_tension_database.md §2, §4, §6

  ↓ draws tool data from
setup_tools_reference.md §1–§8

  ↓ produces data structures that drive
ProblemCard component
SymptomPicker component
SkillTierFilter component
SkillPathway pages
TensionDashboard component (simulation entry)

  ↓ which guides users toward
guitar_setup_parameters_v3.md (full simulation — 5% of users)
string_tension_database.md (tension engine reference — 2% of users)
```

The companion documents are not changed by this architecture — they remain complete technical references. This document defines how the guide reads from them and what it adds. The guide is a view over the technical layer, not a replacement for it.

---

*Document version 1.0*
*Companion documents: `guitar_setup_parameters_v3.md` · `string_tension_database.md` · `setup_tools_reference.md`*
