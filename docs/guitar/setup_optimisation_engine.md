# Setup Optimisation Engine
## Sub-Feature Specification — Parameter Space Navigation & Preference-Driven Setup Targeting

**Version:** 1.0
**Status:** Specification — subject to revision after front-end and UX input
**Companion documents:** [`guitar_setup_parameters_v3.md`](./guitar_setup_parameters_v3.md) · [`string_tension_database.md`](./string_tension_database.md) · [`setup_tools_reference.md`](./setup_tools_reference.md) · [`setup_guide_architecture.md`](./setup_guide_architecture.md)

---

## 1. Purpose & Scope

### 1.1 What this feature is

The Setup Optimisation Engine is a constraint-aware, preference-targeted parameter navigation tool. It answers a specific question that the diagnostic guide and simulation tool cannot:

> *"Given this guitar, with these physical constraints, and a target feel I can describe — what is the shortest path from where I am to where I want to be, and what can actually be changed?"*

The guide diagnoses problems and explains their causes. The simulation reruns the physics for any parameter state. The optimisation engine does something different: it holds fixed constraints constant, operates only on adjustable parameters, evaluates the computed output against a preference target, and ranks the adjustments that move the setup closest to that target per unit of effort.

This is the feature that makes the physics model useful for the expert user who already understands their instrument but wants to eliminate trial and error. It is the real test for the modelling depth described across the companion documents.

### 1.2 Who uses this

Expected usage: fewer than 5% of total users. The profile is a player who:

- Has a specific guitar they know well and want to optimise, not just fix
- Can articulate what they like and dislike about how it plays — not just "it buzzes" but "the bends on the high strings feel too stiff when I drop-tune"
- Is comfortable making setup adjustments, or has access to a technician who is
- Wants to understand the tradeoff space before committing to irreversible changes (nut filing, fret work)

This is not a beginner feature. It assumes the user has already passed through the guide's skill pathways and understands the physical meaning of the parameters they are working with.

### 1.3 What this document covers

- The additional data schemas required: `SetupState`, `ConstraintSet`, `PreferenceTarget`, `OptimisationSession`, `AdjustmentCandidate`
- What each schema pulls from the companion documents and what it adds
- The constraint declaration model — how parameters are tagged as locked, adjustable, or soft
- The preference model — how a player describes a target feel in computable terms
- The scoring function — how a candidate setup state is evaluated against a preference target
- The delta-ranking algorithm — how adjustments are prioritised
- The optimisation workflow in full
- Integration points with the existing simulation and guide layers
- What remains out of scope for v1.0

---

## 2. What the Optimisation Engine Pulls from Companion Documents

The engine is not a new physics layer. It is a navigation layer over the physics that already exists. Every computation it performs runs through the existing model. This section maps the dependencies precisely.

### 2.1 From `guitar_setup_parameters_v3.md`

| Section | What the engine uses |
|---|---|
| §2.1–2.6 (Build parameters) | Source of all `locked` constraint parameters — scale length, fret geometry, board radius, fretwire dimensions, headstock angle. These are the walls of the solution space. |
| §2.7 Neck Setup | `flat_neck_confirmed`, `rod_state`, `relief_bass/treble` — the datum-based relief model is the engine's primary neck parameter. `rod_state` values determine whether the neck constraint is hard (`at_limit` → cannot move) or soft (`mid` → full range available). |
| §2.8 Bridge & Saddle | `saddle_height[1..6]`, `compensation_bass/treble` — these are the engine's highest-impact, fully reversible adjustable parameters. |
| §2.9 Fret-Plane Topology | `falloff_onset_fret`, `falloff_amount` — adjustable within PLEK scope; post-MVP for the optimisation engine since they require specialised equipment. |
| §2.10 Player Profile | `attack_intensity`, `target_action_bass/treble`, `genre_profile` — these become preference inputs in the optimisation layer rather than fixed inputs, since the engine's job is to find a setup that satisfies them. |
| §2.11 Tension Profile | All computed tension outputs (`tension[1..6]`, `balance_score`, `gauge_suitability`, `relief_target`, `rezero_required`) feed directly into the scoring function as objective constraints. |
| §3.1 Master Calculation Flow | The engine calls this calculation graph for every candidate setup state it evaluates. It is unchanged — the engine wraps it, it does not replace it. |
| §4 Problem Index (P-01–P-21, T-01–T-05) | The engine uses the problem index as a constraint violation registry. A candidate state that triggers any problem record is penalised in the scoring function. The severity of the penalty scales with the problem's skill tier (a structural problem is a hard disqualifier; a mild intonation issue is a soft penalty). |

### 2.2 From `string_tension_database.md`

| Section | What the engine uses |
|---|---|
| §2 Physics formula | `T = UW × (2 × L × f)²` — called for every candidate string set in optimisation runs that include string selection. |
| §3 Database schema | `STRING` table — the engine queries unit weights for candidate string sets when the user is exploring gauge options. |
| §4.1 Tuning presets | Used to define the tuning context for the optimisation session. The engine can run a single-tuning or multi-tuning optimisation (the latter for players who use multiple tunings on one instrument). |
| §4.3 Tension change per semitone | Used in multi-tuning optimisation to compute the tension delta across the tuning range and determine whether a single setup is viable across all tunings the player uses, or whether the tradeoffs must be acknowledged explicitly. |
| §6.4 Gauge suitability flags | The `< 7 lbs` and `> 20 lbs` thresholds are hard constraints in the engine's scoring function. A candidate setup state that places any string outside this range is flagged regardless of whether the player has requested it. |
| §6.5 Tension balance score | Feeds the `balance` dimension of the preference target. A player who wants consistent feel across all strings is specifying a high balance score target. |

### 2.3 From `setup_guide_architecture.md`

| Section | What the engine uses |
|---|---|
| §4 Data structures | `ProblemRecord.reversible` flag — the engine uses this to separate its adjustment candidates into reversible and irreversible tiers, and requires explicit user confirmation before including irreversible adjustments in a recommendation. |
| §4 SkillPathway | The engine references skill tier to surface which adjustments the user is rated to perform themselves, and which require referral. |
| §5 Skill tier table | Maps each adjustable parameter to a tier and reversibility rating. This feeds the `adjustment_cost` scoring component. |

---

## 3. New Schemas

The companion documents define the physics, the parameters, and the problem index. The optimisation engine requires four additional schemas that do not exist in any current document.

### 3.1 `SetupState`

A named, saveable snapshot of the complete parameter state for a specific guitar at a specific point in time. This is the core unit the engine operates on.

```typescript
interface SetupState {
  // Identity
  id: string;                          // UUID
  name: string;                        // User-assigned label, e.g. "Drop D with 11s — current"
  created_at: timestamp;
  session_id: string;                  // Parent OptimisationSession

  // Build parameters (locked — sourced from guitar model preset)
  scale_bass: number;                  // inches — from guitar_setup_parameters_v3.md §2.1
  scale_treble: number;                // inches
  radius_nut: number;                  // inches — from §2.2
  radius_heel: number;                 // inches
  fret_crown_height: number;           // mm — from §2.3
  fret_count: number;                  // integer
  headstock_angle: number;             // degrees — from §2.4
  neck_angle: number;                  // degrees — from §2.5

  // String set and tuning (adjustable — sourced from string_tension_database.md)
  string_set_id: string;               // FK → STRING_SET
  tuning_preset_id: string;            // FK → TUNING_PRESET
  custom_tuning_hz: number[] | null;   // 6-element array, overrides preset

  // Setup parameters (adjustable — sourced from §2.7, §2.8)
  flat_neck_confirmed: boolean;
  rod_state: RodState;                 // loose | light | mid | near_limit | at_limit
  relief_bass: number;                 // mm — measured from flat datum
  relief_treble: number;               // mm
  saddle_height: number[];             // mm × 6 strings
  nut_slot_height: number[];           // mm × 6 strings
  compensation_bass: number;           // mm saddle offset
  compensation_treble: number;         // mm saddle offset

  // Fret topology (adjustable — post-MVP)
  falloff_onset_fret: number;
  falloff_amount: number;              // mm

  // Player profile (preference inputs — sourced from §2.10)
  attack_intensity: AttackIntensity;   // light | medium | heavy
  target_action_bass: number;          // mm at 12th fret
  target_action_treble: number;        // mm at 12th fret
  genre_profile: GenreProfile;

  // Computed outputs (populated by the tension engine after state is defined)
  computed: TensionProfile | null;     // null until calculate() is called
}

interface TensionProfile {
  tension_lbs: number[];               // [6] — per string
  tension_total_lbs: number;
  tension_balance_score: number;       // 0–1
  gauge_suitability: GaugeSuitability[]; // ok | too_slack | too_tight × 6
  relief_target: number;               // mm — minimum relief for no buzz at this player profile
  relief_from_flat: number;            // mm — how far to move from confirmed flat datum
  rod_adjustment: RodDirection | null; // tighten | loosen | null (already correct)
  rezero_required: boolean;
  buzz_risk_map: BuzzRisk[][];         // [string][fret] — none | low | medium | high
  intonation_error: number[];          // cents × 6 strings
  problem_flags: string[];             // problem IDs triggered by this state (e.g. ["P-02", "T-01"])
}
```

> **Note:** `SetupState` extends the existing MVP parameter set from `guitar_setup_parameters_v3.md §6` with identity, session linkage, and the full 6-per-string resolution for saddle height and nut slot height that the MVP simplifies to 2 values. The optimisation engine requires per-string resolution to meaningfully differentiate candidates.

---

### 3.2 `ConstraintSet`

Declares which parameters are fixed, which are adjustable, and the physical limits of each adjustable parameter. This is the formal encoding of the user's constraint declaration.

```typescript
interface ConstraintSet {
  id: string;
  session_id: string;

  // Each parameter is tagged with a constraint type
  parameters: ParameterConstraint[];
}

type ConstraintType =
  | "locked"      // Cannot change — physical reality or user choice (e.g. scale length, or "I'm keeping these strings")
  | "adjustable"  // Can change within the declared range
  | "soft"        // A preference target, not a hard bound — the engine optimises toward this

interface ParameterConstraint {
  parameter_id: string;          // Matches parameter names from guitar_setup_parameters_v3.md
  constraint_type: ConstraintType;

  // For "adjustable" parameters
  min_value?: number;            // Physical or user-declared lower bound
  max_value?: number;            // Physical or user-declared upper bound
  step_size?: number;            // Smallest meaningful increment (e.g. 0.05 mm for relief)

  // For "locked" parameters
  locked_value?: number | string;

  // Metadata
  reversible: boolean;           // Sourced from guide_architecture.md §4 ProblemRecord.reversible
  skill_tier: SkillTier;         // t0–t4 — sourced from guide_architecture.md §5
  requires_confirmation: boolean; // true if irreversible — must be explicitly opted in
}
```

**Default constraint classification by parameter group:**

| Parameter group | Default constraint type | Rationale |
|---|---|---|
| Build (scale, radius, fret geometry) | `locked` | Physical — cannot be changed without luthier work |
| `string_set_id` | `adjustable` or `locked` | User choice — "I want to keep these strings" is valid |
| `tuning_preset_id` | `adjustable` or `locked` | User choice — "I play Drop D and that's not changing" |
| `relief_bass / treble` | `adjustable` | Truss rod is reversible within rod_state limits |
| `saddle_height[1..6]` | `adjustable` | Fully reversible |
| `nut_slot_height[1..6]` | `adjustable` | **Irreversible downward** — filing removes material |
| `compensation_bass/treble` | `adjustable` | Fully reversible |
| `falloff_onset_fret / amount` | `locked` (v1.0) | Requires PLEK — out of scope for this engine version |
| `attack_intensity` | `soft` | Preference — the engine can suggest the player recalibrates this |
| `target_action_bass/treble` | `soft` | Preference target — the engine optimises toward this |

---

### 3.3 `PreferenceTarget`

The computable description of what the player is trying to achieve. This is the most novel schema in the engine — it translates subjective feel into quantifiable dimensions.

```typescript
interface PreferenceTarget {
  id: string;
  session_id: string;
  name: string;                        // e.g. "Like my old Les Paul — tight and responsive"

  // Physical playability targets (hard minimums — violations are penalised heavily)
  max_buzz_risk: BuzzRiskLevel;        // none | low | medium — "high" is never a valid target
  max_intonation_error_cents: number;  // Acceptable intonation deviation, e.g. 3.0 cents
  
  // Feel dimensions (scored 0–1, weighted by player priority)
  feel: FeelTarget;

  // Dimension weights (must sum to 1.0 — player declares what matters most)
  weights: FeelWeights;
}

interface FeelTarget {
  // Tension feel — how the strings resist the player
  tension_level: TensionLevel | null;      // very_light | light | medium | firm | heavy
  tension_balance: number | null;          // 0–1 target balance score (1 = perfectly even across strings)

  // Action — how far strings sit from the frets
  action_feel: ActionFeel | null;          // very_low | low | medium | high | very_high
  // Maps to target_action_bass and target_action_treble ranges:
  //   very_low: bass ≤1.4mm, treble ≤1.0mm
  //   low:      bass 1.4–1.7mm, treble 1.0–1.2mm
  //   medium:   bass 1.7–2.0mm, treble 1.2–1.5mm
  //   high:     bass 2.0–2.3mm, treble 1.5–1.8mm
  //   very_high: bass ≥2.3mm, treble ≥1.8mm

  // Bending — how the string responds when pushed off-axis
  bend_resistance: BendResistance | null;  // very_easy | easy | medium | firm | stiff
  // This is a function of tension[n] × scale_length — lower tension or shorter scale = easier bends
  // The engine computes bend_resistance from T[n] and scale — the player names their target

  // Response — how the guitar reacts to pick attack
  dynamic_range: DynamicRange | null;      // narrow | medium | wide
  // Proxy: the gap between clean threshold (low attack) and buzz threshold (high attack)
  // A narrow dynamic range means the guitar behaves similarly at all dynamics
  // A wide range means light playing feels very different from hard playing

  // Intonation — how reliably the guitar stays in tune up the neck
  intonation_priority: IntonationPriority | null; // standard | tight | critical
  // standard: ≤5 cents across all frets
  // tight: ≤3 cents
  // critical: ≤2 cents (requires post-MVP inharmonicity compensation)

  // Reference guitar (optional) — the engine can clone a known setup's feel dimensions
  reference_guitar_id: string | null;      // FK → a stored SetupState used as the target
}

interface FeelWeights {
  tension: number;          // e.g. 0.35
  action: number;           // e.g. 0.25
  bend_resistance: number;  // e.g. 0.20
  dynamic_range: number;    // e.g. 0.10
  intonation: number;       // e.g. 0.10
  // Must sum to 1.0
}
```

> **Design note:** The `reference_guitar_id` field is the formal implementation of the "I want this guitar to feel like that guitar" use case. If the user has a saved `SetupState` for a guitar they love, the engine can extract its computed feel dimensions and use them as the `PreferenceTarget` automatically. The gap between the current guitar's feel dimensions and the reference guitar's feel dimensions becomes the optimisation objective.

---

### 3.4 `AdjustmentCandidate`

A single proposed parameter change, with its predicted impact on each feel dimension, its cost, and any dependencies or risks.

```typescript
interface AdjustmentCandidate {
  id: string;
  session_id: string;

  // What changes
  parameter_id: string;              // e.g. "relief_bass"
  current_value: number | string;
  proposed_value: number | string;
  delta: number | null;              // Numeric delta where applicable

  // Impact on the preference target dimensions
  score_delta: ScoreDelta;           // How much each feel dimension changes
  net_score_improvement: number;     // Weighted sum of score_delta × preference weights

  // Cost and feasibility
  skill_tier: SkillTier;             // t0–t4
  reversible: boolean;
  requires_confirmation: boolean;    // For irreversible changes
  estimated_time_minutes: number;
  tools_required: string[];          // Tool IDs from setup_tools_reference.md

  // Dependencies and conflicts
  depends_on: string[];              // Other adjustments that must happen first
  conflicts_with: string[];          // Adjustments that cannot be combined with this one
  prerequisite_problems: string[];   // Problem IDs that must be resolved before this is valid

  // Warnings
  warnings: AdjustmentWarning[];
}

interface ScoreDelta {
  tension: number;          // Change in tension score (positive = toward target)
  action: number;
  bend_resistance: number;
  dynamic_range: number;
  intonation: number;
  buzz_risk_change: number; // Negative = buzz risk increased
}

interface AdjustmentWarning {
  severity: "info" | "caution" | "block";
  message: string;
  // "block" severity prevents this adjustment from being recommended
  // e.g. rod_state = at_limit blocks any relief-increasing recommendation
}
```

---

### 3.5 `OptimisationSession`

The container for the full optimisation workflow — from initial state capture through constraint declaration, preference setting, and candidate ranking.

```typescript
interface OptimisationSession {
  id: string;
  created_at: timestamp;
  updated_at: timestamp;

  // The guitar being optimised
  guitar_model_id: string | null;    // Optional — if selected from preset library
  guitar_label: string;              // User-assigned name, e.g. "2019 Strat MIM"

  // Session components
  states: SetupState[];              // Snapshots — at minimum: current state + any evaluated candidates
  constraint_set: ConstraintSet;
  preference_target: PreferenceTarget;
  candidates: AdjustmentCandidate[];

  // Workflow state
  phase: OptimisationPhase;
  iteration_count: number;          // How many evaluate → adjust cycles have run

  // Output
  recommended_sequence: string[];   // Ordered adjustment_candidate IDs — the suggested path
  estimated_sessions_to_target: number; // How many physical setup sessions to reach target
}

type OptimisationPhase =
  | "state_capture"         // User is entering current setup parameters
  | "constraint_declaration" // User is tagging what can and cannot change
  | "preference_setting"     // User is describing the target feel
  | "candidate_generation"   // Engine is computing candidate adjustments
  | "review"                 // User is reviewing ranked candidates
  | "iteration"              // User has made changes and is re-entering the new state
  | "complete";              // Target reached or user has accepted the result
```

---

## 4. The Constraint Declaration Model

### 4.1 Why constraints matter

Without constraint declaration, the optimisation problem is unconstrained — theoretically solvable by replacing the guitar entirely. The constraint layer encodes physical reality (scale length cannot change), instrument permanence (the user is working with this guitar), and player preference (the strings are staying, the tuning is staying).

Constraints collapse the solution space from infinite to navigable. The engine's job is to find the best path within that space, not to identify an ideal instrument.

### 4.2 Constraint inference from context

Many constraints can be inferred without asking the user explicitly:

- **Build parameters are always locked.** Scale length, fret geometry, board radius — these are tagged `locked` automatically when the guitar model is selected.
- **`rod_state` limits relief range.** If `rod_state = near_limit`, the maximum adjustable relief is constrained accordingly. The engine reads this from the existing parameter and does not ask the user to re-enter it.
- **Irreversible parameters require opt-in.** Nut slot height filing is `locked` by default. The user must explicitly unlock it and confirm they accept the irreversibility.
- **Tension physics are always hard constraints.** The `< 7 lbs` and `> 20 lbs` gauge suitability thresholds from `string_tension_database.md §6.4` are never negotiable — the engine enforces them regardless of user preference.

### 4.3 Constraint conflict detection

Before the engine generates candidates, it validates the constraint set for internal contradictions:

| Conflict type | Example | Resolution |
|---|---|---|
| Locked parameter contradicts preference target | String set locked, but target tension requires different gauge | Warn user — target may not be achievable without changing the locked parameter |
| `rod_state = at_limit` with relief increase required | Neck cannot bow further | Block relief-increase candidates, surface P-21 |
| Irreversible parameter locked, target requires it | Nut slots locked, but action target requires nut filing | Warn that target requires unlocking nut slots |
| Multi-tuning conflict | Two tunings require contradictory relief settings | Surface tradeoff explicitly — no single setup satisfies both optimally |

---

## 5. The Preference Model

### 5.1 From feel words to computed dimensions

The preference model must translate what a player actually says ("it feels like fighting the guitar", "I want it to feel effortless like my other one", "the bends should feel like butter") into dimensions the physics model can evaluate.

The five dimensions map to computable quantities as follows:

| Feel dimension | Computed proxy | Source calculation |
|---|---|---|
| **Tension level** | Mean `tension[1..6]` in lbs | `T = UW × (2 × L × f)²` from string_tension_database.md §2 |
| **Tension balance** | `tension_balance_score` (0–1) | Variance across `tension[1..6]` relative to mean |
| **Action feel** | `target_action_bass` + `target_action_treble` combined | Direct from player profile §2.10, with attack_multiplier applied |
| **Bend resistance** | `T[n] × sin(bend_angle)` at representative fret | Bend force formula from string_tension_database.md §7 T-05 |
| **Dynamic range** | Gap between clean floor and buzz ceiling | `buzz_threshold(attack=heavy) − clean_threshold(attack=light)` at each fret |
| **Intonation** | Max intonation error in cents across all strings | Compensation model from guitar_setup_parameters_v3.md §2.8 |

### 5.2 Tension level calibration

Raw tension values in lbs are not meaningful to most players. The engine maps them to named levels:

| Tension level label | Mean tension range | Typical context |
|---|---|---|
| `very_light` | < 9 lbs average | Low-tuning light gauge; very easy bending |
| `light` | 9–11 lbs | Standard light set (.009–.042) in E standard |
| `medium` | 11–14 lbs | .010–.046 in E standard; .011–.048 in Eb |
| `firm` | 14–17 lbs | .011–.052 in E; heavier gauge or longer scale |
| `heavy` | > 17 lbs | Baritone gauges; extended scale; high tension preference |

### 5.3 Bend resistance calibration

Bend resistance describes how much physical effort a full-step bend requires on the B string at the 12th fret — a common reference point across guitar styles.

| Bend resistance label | Force range (approximate) | Character |
|---|---|---|
| `very_easy` | < 0.8 kg equivalent | Almost no resistance; suited to blues/lead players |
| `easy` | 0.8–1.1 kg | Light effort; responsive |
| `medium` | 1.1–1.4 kg | Standard feel for most players |
| `firm` | 1.4–1.8 kg | Controlled resistance; suited to rhythm players |
| `stiff` | > 1.8 kg | High tension feel; suited to strong attack players |

### 5.4 Reference guitar cloning

When a user specifies a `reference_guitar_id`, the engine:

1. Loads the reference `SetupState` and runs `calculate()` to produce its `TensionProfile`
2. Extracts the feel dimension values from that profile
3. Sets those values as the `PreferenceTarget.feel` dimensions
4. Sets the weights proportionally to how far the current guitar deviates from the reference in each dimension — the dimension with the largest gap gets the highest weight automatically (though the user can override this)

This is the implementation of "make this guitar feel like that guitar." The engine quantifies the gap and plans the path.

---

## 6. The Scoring Function

### 6.1 Overview

For any candidate `SetupState`, the scoring function produces a single composite score (0–1) that represents how close that state is to the `PreferenceTarget`. The score is used to rank `AdjustmentCandidates` by their impact on the overall optimisation objective.

### 6.2 Dimension scores

Each feel dimension is scored independently (0–1) before weighting:

```
score_tension        = 1 − |mean(tension[1..6]) − target_tension_midpoint| / tension_range_width
score_balance        = tension_balance_score                             // already 0–1
score_action         = 1 − |action_actual − action_target| / action_range_width
score_bend           = 1 − |bend_resistance_actual − bend_target| / bend_range_width
score_dynamic_range  = 1 − |dynamic_range_actual − dynamic_target| / dynamic_range_width
score_intonation     = max(0, 1 − (max_intonation_error / max_acceptable_error))
```

### 6.3 Constraint violations

Before weighting, constraint violations are applied as penalties:

| Violation | Penalty |
|---|---|
| Any string `gauge_suitability = too_slack` or `too_tight` | −0.50 (hard penalty) |
| Any fret with `buzz_risk = high` | −0.40 |
| Any fret with `buzz_risk = medium` | −0.15 per string |
| `rod_state = at_limit` and relief < `relief_target` | −0.60 (structural block) |
| Any problem flag in `ProblemRecord` with `skill_tier = t4` | −1.00 (disqualifying) |
| Intonation error > `max_intonation_error_cents` | −0.20 |

### 6.4 Composite score

```
composite_score = (
    weights.tension        × score_tension      +
    weights.action         × score_action        +
    weights.bend_resistance × score_bend         +
    weights.dynamic_range  × score_dynamic_range +
    weights.intonation     × score_intonation
  ) + (weights.tension × score_balance × 0.5)   // balance is a bonus within the tension dimension
  + sum(constraint_penalties)
```

The composite score drives the ranking of `AdjustmentCandidates`. The engine selects the candidate that produces the largest `net_score_improvement` per unit of `estimated_time_minutes`, subject to the dependency and conflict constraints in the candidate graph.

---

## 7. The Delta-Ranking Algorithm

### 7.1 Candidate generation

For each `adjustable` parameter in the `ConstraintSet`, the engine generates a set of candidate adjustments by stepping through the parameter's range in increments of `step_size`. For each candidate value, it:

1. Clones the current `SetupState`
2. Applies the candidate value
3. Runs `calculate()` to produce a new `TensionProfile`
4. Scores the new state using the scoring function
5. Computes `score_delta = new_score − current_score`
6. Records this as an `AdjustmentCandidate`

Parameters with small step sizes (e.g. relief at 0.05 mm increments) generate many candidates. The engine evaluates them all but surfaces only the top candidates per parameter to the user.

### 7.2 Ranking criteria

Candidates are ranked by:

```
rank_score = (net_score_improvement / estimated_time_minutes) × reversibility_multiplier
```

Where:
```
reversibility_multiplier = 1.0 if reversible
                         = 0.6 if irreversible (penalised but not excluded)
```

Irreversible changes are available to the user but ranked lower than equivalent reversible changes, all else equal.

### 7.3 Dependency ordering

The final recommended sequence respects physical dependencies:

1. **Datum first.** If `flat_neck_confirmed = false`, the datum step is always first, regardless of rank score. No downstream parameter can be reliably set without it.
2. **Neck before saddle.** Relief changes affect action across the whole neck. Saddle height should be set after relief is confirmed.
3. **Saddle before nut.** Nut slot height is calibrated relative to the action established at the saddle and 12th fret.
4. **Intonation last.** Saddle compensation is the final step after action geometry is fully resolved.

This ordering is hard-coded in the dependency graph regardless of rank scores. The engine respects the existing technician workflow from `guitar_setup_parameters_v3.md §3.1`.

### 7.4 Convergence detection

The engine detects convergence when:
- The top-ranked candidate produces `net_score_improvement < 0.02` (less than 2% improvement available)
- All remaining candidates are irreversible and the user has not opted in to them
- The composite score exceeds `0.90` (90% match to preference target)

At convergence, the engine reports that the target has been reached or that remaining gains require irreversible work or a change to the locked constraints.

---

## 8. Optimisation Workflow

### 8.1 Full workflow

```
PHASE 1 — State capture
  User selects guitar model (loads build parameters as locked constraints)
  User enters current setup measurements:
    flat_neck_confirmed, rod_state
    relief_bass, relief_treble
    saddle_height[1..6] (or simplified bass/treble)
    nut_slot_height[1..6] (or simplified bass/treble)
    string_set_id, tuning_preset_id
    attack_intensity, target_action_bass/treble
  Engine runs calculate() → produces current TensionProfile
  Engine flags any existing problem records triggered by current state

PHASE 2 — Constraint declaration
  Engine proposes default constraints (build = locked, setup = adjustable, profile = soft)
  User reviews and overrides:
    "Keep these strings" → string_set_id = locked
    "Drop D is non-negotiable" → tuning_preset_id = locked
    "I'm not filing the nut" → nut_slot_height = locked
  Engine validates constraint set for conflicts
  Engine surfaces conflicts with explanation (not as errors — as tradeoffs)

PHASE 3 — Preference target
  Option A: User describes target feel using dimension pickers
    → tension level, action feel, bend resistance, dynamic range, intonation priority
    → user assigns weights (or accepts defaults based on genre_profile)
  Option B: User selects a reference guitar from saved SetupStates
    → engine clones feel dimensions from reference
    → engine auto-weights by gap magnitude
  Option C: User describes a problem with current setup in plain language
    → maps to problem IDs from the guide's problem index
    → engine sets preference target to the inverse of the flagged problems

PHASE 4 — Candidate generation and ranking
  Engine generates AdjustmentCandidates for all adjustable parameters
  Engine scores each candidate against the PreferenceTarget
  Engine orders candidates by rank_score
  Engine applies dependency ordering to produce recommended_sequence
  Engine separates reversible and irreversible candidates

PHASE 5 — Review and selection
  User sees ranked candidate list:
    → Reversible candidates: presented as a sequence to try
    → Irreversible candidates: gated behind explicit confirmation
    → Locked-constraint candidates: shown as "not available — requires X to change"
  User selects which adjustments to make in this setup session

PHASE 6 — Iteration
  User makes selected adjustments physically
  User re-enters new measured values → new SetupState snapshot
  Engine re-runs scoring and re-ranks remaining candidates
  Engine updates recommended_sequence
  Repeat until convergence or user accepts current state
```

### 8.2 Multi-tuning optimisation

When a player uses multiple tunings on one instrument, the engine runs the optimisation across all declared tunings simultaneously:

1. A `SetupState` is evaluated against the `PreferenceTarget` for each tuning
2. Composite score is the weighted average across tunings (user declares which tuning is primary)
3. Candidates that improve score in one tuning but harm score in another surface the tradeoff explicitly
4. The engine identifies the Pareto-optimal setup — the state where no single adjustment improves all tunings simultaneously — and presents this as the achievable multi-tuning optimum

This is the formal answer to "I play both E standard and Drop D — what setup gives me the best of both?"

---

## 9. Integration with Existing Layers

### 9.1 Entry point from the simulation tool

The optimisation engine is accessible from Layer 4 of the guide architecture (`TensionDashboard`). A user who has run the simulation and seen their current state's outputs can initiate an optimisation session from the same interface. The current simulation state pre-populates `SetupState` Phase 1.

### 9.2 Reference to the problem index

The engine does not re-implement problem diagnosis. When the current `SetupState` triggers a problem flag (e.g. P-02, T-01), the engine surfaces the existing `ProblemRecord` from the guide — including its plain-language explanation, skill tier, and off-ramp — rather than generating new content.

### 9.3 Session persistence

`OptimisationSession` objects are persistent. A user can return to an in-progress session after physically making adjustments, re-enter their new measurements, and continue the iteration cycle without re-entering the guitar model, constraints, or preference target.

### 9.4 Export

At any point, the user can export the session as a plain-language setup sheet — a printable document listing the target values for each adjustable parameter, the order to set them, and the measurements to verify. This is the output a player would hand to a technician.

---

## 10. What Is Out of Scope for v1.0

The following are explicitly deferred. They are noted here so that the v1.0 schema design does not foreclose them.

| Deferred feature | Why deferred | Where it would plug in |
|---|---|---|
| Fret topology optimisation (`falloff_onset_fret`, `falloff_amount`) | Requires PLEK — not a user-adjustable parameter in most setups | `ConstraintSet` — currently locked; unlock when PLEK integration is added |
| Per-string inharmonicity compensation (`compensation_inharmonicity[n]`) | Post-MVP from `guitar_setup_parameters_v3.md §2.8` | Feeds `score_intonation` dimension at higher precision |
| Temperature/humidity tension correction | ~0.1% per °C — negligible for most optimisation decisions | Feeds `TensionProfile` as a correction factor |
| Custom gauge mixing across brands | Requires per-string unit weight from the DB for a non-standard set | `string_set_id` becomes nullable; per-string `unit_weight` entered manually |
| 7-string and baritone support | Schema extension — add string position 7; adjust balance score formula | `SetupState.saddle_height[]` and `nut_slot_height[]` extend to 7 |
| Acoustic guitar adaptation | Saddle and nut model carries over; neck relief model differs; top compliance adds a new variable | New `guitar_type` flag; modified `ConstraintSet` defaults |
| Community preference presets | "Import another player's preference target" — requires account system | `PreferenceTarget` is already a portable schema; persistence is the missing piece |

---

## 11. Document Relationships

```
setup_optimisation_engine.md (this document)
  ↓ reads fixed physics from
string_tension_database.md §2, §4, §6

  ↓ reads all parameter definitions from
guitar_setup_parameters_v3.md §2, §3, §4

  ↓ reads reversibility and skill tier from
setup_guide_architecture.md §4, §5

  ↓ reads tool requirements from
setup_tools_reference.md §1–§8

  ↓ produces
SetupState snapshots
ConstraintSet per guitar
PreferenceTarget per player objective
AdjustmentCandidate ranked list
OptimisationSession (persistent)

  ↓ which drives (future UI)
StateComparisonView
ConstraintDeclarationPanel
PreferenceTargetBuilder
CandidateRankingPanel
IterationTracker
SetupSheetExport
```

---

*Document version 1.0*
*Companion documents: `guitar_setup_parameters_v3.md` · `string_tension_database.md` · `setup_tools_reference.md` · `setup_guide_architecture.md`*
*Status: Initial specification — subject to revision after front-end and UX input*
