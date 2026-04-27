# Guitar Setup Tools Reference
## Physical Tools Required for the Electric Guitar Setup Model

**Version:** 1.1
**Companion documents:** [`guitar_setup_parameters_v3.md`](./guitar_setup_parameters_v3.md) · [`setup_guide_architecture.md`](./setup_guide_architecture.md) · [`feature_layers.md`](./feature_layers.md) · [`setup_optimisation_engine.md`](./setup_optimisation_engine.md) · [`agent_access_spec.md`](./agent_access_spec.md) · [`string_tension_database.md`](./string_tension_database.md)

This document maps every physical tool required by the v3.1 setup workflow to the specific parameters, sections, and playability problems it serves. Tools are grouped by function. References use the section and problem numbering from [`guitar_setup_parameters_v3.md`](./guitar_setup_parameters_v3.md).

---

## Changelog: v1.0 → v1.1

| Section | Change |
|---|---|
| Header | Version bumped to 1.1; companion document list expanded to all six sibling documents. |
| §1–§5 (all tool sections) | Three columns added to each tool entry: `risk_if_wrong`, `learning_value`, and `cost_range`. These were flagged as missing in [`setup_guide_architecture.md §9`](./setup_guide_architecture.md#9-content-additions-required-in-companion-documents) and are required to populate `FixStep.warning`, the skill pathway learnable tool inventory, and `SetupTool.cost_range` in the guide's data structures. |
| §9 Document Relationships | New section added. Maps this document's role in the full six-document system. |

---

## 1. Measurement Tools

### 1.1 Feeler Gauge Set

The most-used tool in the entire workflow. Required at multiple stages.

| Use | Parameter / Section | Problem |
|-----|---------------------|---------|
| Establish flat-neck datum — confirm ≤ 0.02 mm gap at mid-neck frets 7–8 | [`§2.7 flat_neck_confirmed`](./guitar_setup_parameters_v3.md#27-neck-setup) | Prerequisite for all relief work |
| Measure neck relief, bass side, at 8th fret under string tension | [`§2.7 relief_bass`](./guitar_setup_parameters_v3.md#27-neck-setup) | [P-02](./guitar_setup_parameters_v3.md#p-02--buzz-across-the-whole-neck), [P-17](./guitar_setup_parameters_v3.md#p-17--neck-feels-different-bass-side-vs-treble-side) |
| Measure neck relief, treble side, at 8th fret under string tension | [`§2.7 relief_treble`](./guitar_setup_parameters_v3.md#27-neck-setup) | [P-02](./guitar_setup_parameters_v3.md#p-02--buzz-across-the-whole-neck), [P-17](./guitar_setup_parameters_v3.md#p-17--neck-feels-different-bass-side-vs-treble-side) |
| Confirm open-string nut clearance above first fret crown | [`§2.6 nut_slot_height[1..6]`](./guitar_setup_parameters_v3.md#26-nut) | [P-01](./guitar_setup_parameters_v3.md#p-01--buzz-on-open-strings-only), [P-08](./guitar_setup_parameters_v3.md#p-08--action-too-high-only-in-first-position), [P-13](./guitar_setup_parameters_v3.md#p-13--open-string-sharp-12th-fret-correct) |

| Property | Value |
|---|---|
| `cost_range` | $5–25 USD (feeler gauge set) |
| `learning_value` | High — this tool is used at every setup session; developing fluency with it is foundational |
| `risk_if_wrong` | Over-deflection at the datum step produces a false `flat_neck_confirmed`. All downstream relief and action values are calibrated from a wrong zero. The resulting setup will have more relief than intended and action may be higher than targeted. Recoverable by re-establishing the datum. |

> **Workflow note ([§3.3](./guitar_setup_parameters_v3.md#33-setup-workflow-sequence)):** The feeler gauge is used at step 2 (confirming flat before setting `flat_neck_confirmed = true`) and again at step 4 (measuring `relief_bass` / `relief_treble` after loosening the rod). Both readings feed the computed `relief_from_flat` output in [`§2.11`](./guitar_setup_parameters_v3.md#211-tension-profile-updated-in-v30).

---

### 1.2 Precision Straight Edge

Laid across fret crowns along the length of the neck to confirm flat. Required before the datum can be established.

| Use | Parameter / Section | Problem |
|-----|---------------------|---------|
| Verify zero gap at mid-neck frets under full string tension | [`§2.7 flat_neck_confirmed`](./guitar_setup_parameters_v3.md#27-neck-setup) | [T-03](./guitar_setup_parameters_v3.md#t-03--guitar-buzzes-after-retuning-but-was-fine-before), [P-21](./guitar_setup_parameters_v3.md#p-21--truss-rod-at-or-near-limit) |
| Detect neck twist — uneven contact bass vs. treble side | [`§2.7 relief_bass / relief_treble`](./guitar_setup_parameters_v3.md#27-neck-setup) | [P-17](./guitar_setup_parameters_v3.md#p-17--neck-feels-different-bass-side-vs-treble-side) |

| Property | Value |
|---|---|
| `cost_range` | $20–80 USD (machinist's straight edge, 450–600 mm) |
| `learning_value` | High — required for every datum-based relief workflow; also the primary tool for detecting neck twist |
| `risk_if_wrong` | A warped or incorrectly applied straight edge produces a false flat reading. The datum is set incorrectly and all subsequent relief measurements reference a wrong zero. Twist may be missed entirely, leaving P-17 undiagnosed. Not reversible without re-establishing the datum at the correct string tension and guitar orientation. |

> **Specification:** Long enough to span the full fretboard — typically 450–600 mm. A machinist's straight edge is preferred over a guitar-specific tool for the accuracy required by the ≤ 0.02 mm flat datum.

> **Re-zero trigger:** When tuning changes by `|ΔT_total| ≥ 8 lbs`, the model sets `rezero_required = true` ([`§2.11`](./guitar_setup_parameters_v3.md#211-tension-profile-updated-in-v30)). The straight edge is used again at the new tuning to re-establish the datum before any relief work proceeds.

---

### 1.3 Fret Rocker (Three-Fret Rocker)

A short straight edge that spans exactly three frets. Rocks on a high fret; drops on a low one. Used to locate individual proud or sunken frets without a full fret height map.

| Use | Parameter / Section | Problem |
|-----|---------------------|---------|
| Isolate high frets in first position causing positional buzz | [`§2.9 fret_height_map[2..5]`](./guitar_setup_parameters_v3.md#29-fret-plane-topology) | [P-03](./guitar_setup_parameters_v3.md#p-03--buzz-only-in-first-position-frets-15) |
| Isolate high frets above fret 12 causing upper-register buzz | [`§2.9 fret_height_map[12..N]`](./guitar_setup_parameters_v3.md#29-fret-plane-topology) | [P-04](./guitar_setup_parameters_v3.md#p-04--buzz-only-at-high-frets-above-fret-12) |
| Identify fret causing pitch deviation at a specific position | [`§2.9 fret_height_map[n]`](./guitar_setup_parameters_v3.md#29-fret-plane-topology) | [P-12](./guitar_setup_parameters_v3.md#p-12--intonation-correct-at-12th-fret-wrong-at-other-frets) |

| Property | Value |
|---|---|
| `cost_range` | $10–30 USD |
| `learning_value` | Medium — useful diagnostic tool but only needed when fret-specific buzz is suspected; not part of every setup session |
| `risk_if_wrong` | No material removal — purely diagnostic. Misreading a rocking result (confusing vibration for rock) may produce a false positive and lead to unnecessary fret work. The tool itself causes no harm if misread. |

> **Diagnostic technique for [P-03](./guitar_setup_parameters_v3.md#p-03--buzz-only-in-first-position-frets-15):** First capo at fret 1 and play — if first-position buzz clears, the nut is the cause and the fret rocker is not needed. If buzz persists with the capo, work the fret rocker across frets 2–5 to find the offender.

---

### 1.4 String Action Gauge / Notched Ruler

Measures string height above the 12th fret crown for setting and verifying action targets.

| Use | Parameter / Section | Problem |
|-----|---------------------|---------|
| Measure action at 12th fret, bass side | [`§2.10 target_action_bass`](./guitar_setup_parameters_v3.md#210-player-profile) | [P-07](./guitar_setup_parameters_v3.md#p-07--action-too-high-across-the-whole-neck), [P-09](./guitar_setup_parameters_v3.md#p-09--action-too-low-buzzing-at-normal-playing) |
| Measure action at 12th fret, treble side | [`§2.10 target_action_treble`](./guitar_setup_parameters_v3.md#210-player-profile) | [P-07](./guitar_setup_parameters_v3.md#p-07--action-too-high-across-the-whole-neck), [P-14](./guitar_setup_parameters_v3.md#p-14--notes-choke-or-die-when-bending) |
| Verify first-position action is approximately half of 12th-fret action | [`§2.6 nut_slot_height[1..6]`](./guitar_setup_parameters_v3.md#26-nut) | [P-08](./guitar_setup_parameters_v3.md#p-08--action-too-high-only-in-first-position) |

| Property | Value |
|---|---|
| `cost_range` | $10–40 USD (dedicated notched ruler) |
| `learning_value` | High — action measurement is part of every setup; a notched ruler is significantly more accurate than improvising with a standard ruler and worth owning |
| `risk_if_wrong` | Measurement error produces incorrect saddle height targets. Saddle adjustment is reversible if shims are used; saddle sanding is not. Confirm readings at least twice before committing to any material removal. |

> **Typical ranges:** `target_action_bass` 1.5–2.5 mm; `target_action_treble` 1.0–2.0 mm. A dedicated notched ruler (which spans the string and sits on the fret crown) is more accurate than a standard ruler for this measurement.

---

### 1.5 Digital Calipers

Precise measurement of small dimensions — nut geometry, string gauges, and saddle heights.

| Use | Parameter / Section | Problem |
|-----|---------------------|---------|
| Measure nut slot depth and width per string | [`§2.6 nut_slot_height[1..6]`](./guitar_setup_parameters_v3.md#26-nut) | [P-01](./guitar_setup_parameters_v3.md#p-01--buzz-on-open-strings-only), [P-08](./guitar_setup_parameters_v3.md#p-08--action-too-high-only-in-first-position), [P-18](./guitar_setup_parameters_v3.md#p-18--tuning-instability--strings-drift-after-bending) |
| Confirm string gauge before loading `unit_weight` from DB | [`§2.4 gauge[1..6]`](./guitar_setup_parameters_v3.md#24-string-set) | — |
| Measure saddle height per string at contact point | [`§2.8 saddle_height[1..6]`](./guitar_setup_parameters_v3.md#28-bridge--saddle) | [P-07](./guitar_setup_parameters_v3.md#p-07--action-too-high-across-the-whole-neck), [P-10](./guitar_setup_parameters_v3.md#p-10--sharp-intonation-at-every-fret) |
| Measure nut thickness and string spacing at nut | [`§2.6 nut_thickness`, `nut_string_spacing`](./guitar_setup_parameters_v3.md#26-nut) | — |

---

### 1.6 Capo

Used as a diagnostic tool, not a playing aid, during the setup workflow.

| Use | Parameter / Section | Problem |
|-----|---------------------|---------|
| Capo at fret 1 to isolate whether first-position buzz is nut or fret | [`§2.6 nut_slot_height[1..6]`](./guitar_setup_parameters_v3.md#26-nut) | [P-03](./guitar_setup_parameters_v3.md#p-03--buzz-only-in-first-position-frets-15) |

> **Method:** With capo at fret 1, the nut is bypassed. If buzz in first position clears, the nut slots are causing it. If buzz persists, use the fret rocker across frets 2–5.

---

### 1.7 Electronic Tuner (Chromatic, High Precision)

Required for tuning to pitch before the flat-neck datum and for all intonation work. A strobe tuner or high-accuracy clip tuner is needed — typical app tuners lack the resolution for compensation adjustment.

| Use | Parameter / Section | Problem |
|-----|---------------------|---------|
| Tune to pitch at target tuning before establishing flat-neck datum | [`§2.7 flat_neck_confirmed`](./guitar_setup_parameters_v3.md#27-neck-setup), [`§2.5 tuning_preset_id`](./guitar_setup_parameters_v3.md#25-tuning) | Prerequisite for all setup |
| Set intonation — compare open vs. 12th fret pitch per string | [`§2.8 compensation_bass / compensation_treble`](./guitar_setup_parameters_v3.md#28-bridge--saddle) | [P-10](./guitar_setup_parameters_v3.md#p-10--sharp-intonation-at-every-fret), [P-11](./guitar_setup_parameters_v3.md#p-11--flat-intonation-at-every-fret), [P-13](./guitar_setup_parameters_v3.md#p-13--open-string-sharp-12th-fret-correct) |
| Detect intonation error at mid-fret positions | [`§2.9 fret_height_map[n]`](./guitar_setup_parameters_v3.md#29-fret-plane-topology) | [P-12](./guitar_setup_parameters_v3.md#p-12--intonation-correct-at-12th-fret-wrong-at-other-frets), [T-04](./guitar_setup_parameters_v3.md#t-04--intonation-shifts-after-changing-tuning-not-covered-by-compensation) |

> **Tuning change and intonation ([T-04](./guitar_setup_parameters_v3.md#t-04--intonation-shifts-after-changing-tuning-not-covered-by-compensation)):** When tuning changes, string inharmonicity shifts, requiring micro-adjustment of saddle compensation. The tuner is the primary instrument for detecting this — the model flags `|ΔT_total| ≥ 8 lbs` changes as likely to require compensation re-check.

---

## 2. Truss Rod Tools

### 2.1 Truss Rod Wrench (Guitar-Specific)

The actuator for all truss rod adjustments. The correct size and type must match the instrument — using the wrong tool risks stripping the nut or rod.

| Use | Parameter / Section | Problem |
|-----|---------------------|---------|
| Tighten rod to reach flat-neck datum from forward bow | [`§2.7 flat_neck_confirmed`, `rod_state`](./guitar_setup_parameters_v3.md#27-neck-setup) | Prerequisite for all neck work |
| Loosen rod to add `relief_from_flat` after datum is confirmed | [`§2.7 rod_direction`, `§2.11 relief_from_flat`](./guitar_setup_parameters_v3.md#211-tension-profile-updated-in-v30) | [P-02](./guitar_setup_parameters_v3.md#p-02--buzz-across-the-whole-neck), [T-03](./guitar_setup_parameters_v3.md#t-03--guitar-buzzes-after-retuning-but-was-fine-before) |
| Assess rod state — feel for resistance to categorise `rod_state` | [`§2.7 rod_state`](./guitar_setup_parameters_v3.md#27-neck-setup) | [P-21](./guitar_setup_parameters_v3.md#p-21--truss-rod-at-or-near-limit) |

> **Common wrench types by guitar:**
> - Hex (Allen) key — most common; sizes vary (typically 4 mm, 5 mm, or 3/16")
> - Gibson-style spoke wheel — accessed at the headstock, no wrench needed
> - Fender bullet nut — requires a socket or open-end wrench at the headstock

> **Rod state assessment ([§2.7](./guitar_setup_parameters_v3.md#27-neck-setup)):** Before turning, the technician assesses and records `rod_state` (`loose` / `light` / `mid` / `near_limit` / `at_limit`). At `near_limit`, proceed in very small increments only. At `at_limit`, stop — do not force — and flag [P-21](./guitar_setup_parameters_v3.md#p-21--truss-rod-at-or-near-limit) for luthier referral.

> **Turn increments and settling time ([§2.7](./guitar_setup_parameters_v3.md#27-neck-setup)):** The model targets relief as a measured mm value confirmed by feeler gauge, not a turn count. Turn counts are not modelled — they vary between rod types and are non-linear near limits. Allow 10–15 minutes between increments for the neck to settle before remeasuring.

---

## 3. Nut Work Tools

### 3.1 Nut Files (Gauged Set)

For cutting or widening nut slots to the correct depth and profile per string. Files come matched to string gauges — using the wrong width leaves the string either binding or floating loose in the slot.

| Use | Parameter / Section | Problem |
|-----|---------------------|---------|
| Deepen slots when clearance above fret 1 is too high | [`§2.6 nut_slot_height[1..6]`](./guitar_setup_parameters_v3.md#26-nut) | [P-08](./guitar_setup_parameters_v3.md#p-08--action-too-high-only-in-first-position) |
| Widen slots to match current string gauge and splay angle | [`§2.6 nut_slot_height[1..6]`, `nut_splay_angle[1..6]`](./guitar_setup_parameters_v3.md#26-nut) | [P-18](./guitar_setup_parameters_v3.md#p-18--tuning-instability--strings-drift-after-bending) |
| Re-cut slots when switching to a heavier string set | [`§2.4 gauge[1..6]`](./guitar_setup_parameters_v3.md#24-string-set), [`§2.6 nut_slot_height[1..6]`](./guitar_setup_parameters_v3.md#26-nut) | [P-01](./guitar_setup_parameters_v3.md#p-01--buzz-on-open-strings-only), [P-18](./guitar_setup_parameters_v3.md#p-18--tuning-instability--strings-drift-after-bending) |

> **Tension link:** The required slot profile is driven by `unit_weight[n]` loaded from the String Tension DB ([`§2.4`](./guitar_setup_parameters_v3.md#24-string-set)). A heavier string set demands wider and deeper slots. The model updates nut slot height recommendations automatically when the string set changes — the nut files are the physical implementation of that change.

> **Slot width matters for [P-18](./guitar_setup_parameters_v3.md#p-18--tuning-instability--strings-drift-after-bending):** A slot cut with the wrong-width file will bind the string even if the depth is correct. Each string needs a file matched to its gauge.

---

### 3.2 Nut Slot Lubricant

Reduces friction in nut slots to allow the string to return to exact pitch after bending or vibrato use.

| Use | Parameter / Section | Problem |
|-----|---------------------|---------|
| Lubricate slots after cutting to reduce binding friction | [`§2.6 nut_slot_height[1..6]`, `nut_splay_angle[1..6]`](./guitar_setup_parameters_v3.md#26-nut) | [P-18](./guitar_setup_parameters_v3.md#p-18--tuning-instability--strings-drift-after-bending) |

> **Options:** Graphite powder (pencil lead), Big Bends Nut Sauce, or purpose-made nut lubricant. Applied to the slot after filing and before restringing. Higher-tension strings (heavier gauge / higher tuning) require a cleaner, better-lubricated slot — see the tension link in [`§2.6`](./guitar_setup_parameters_v3.md#26-nut).

---

### 3.3 Nut Blanks and Adhesive

Required when a nut must be replaced rather than modified — typically when slots have been cut too low to recover ([P-13](./guitar_setup_parameters_v3.md#p-13--open-string-sharp-12th-fret-correct)) or slot geometry is unrecoverable ([P-18](./guitar_setup_parameters_v3.md#p-18--tuning-instability--strings-drift-after-bending)).

| Use | Parameter / Section | Problem |
|-----|---------------------|---------|
| Replace nut when slots are too low to re-cut | [`§2.6 nut_slot_height[1..6]`](./guitar_setup_parameters_v3.md#26-nut) | [P-13](./guitar_setup_parameters_v3.md#p-13--open-string-sharp-12th-fret-correct) |
| Replace nut when slot angles are unrecoverable | [`§2.6 nut_splay_angle[1..6]`](./guitar_setup_parameters_v3.md#26-nut) | [P-18](./guitar_setup_parameters_v3.md#p-18--tuning-instability--strings-drift-after-bending) |

> **Materials:** Bone, TUSQ, or synthetic nut blank material appropriate to the guitar. The blank must be fitted to the nut slot in the neck, then shaped and slotted from scratch using sandpaper, shaping files, and gauged nut files (§3.1 above).

---

### 3.4 Sandpaper and Shaping Files

Used when fitting a new nut blank — material is removed from the base to achieve correct height before slots are cut.

| Use | Parameter / Section |
|-----|---------------------|
| Shape and fit a new nut blank to correct height | [`§2.6 nut_thickness`, `nut_slot_height[1..6]`](./guitar_setup_parameters_v3.md#26-nut) |

---

## 4. Saddle and Bridge Tools

The specific tool required depends on the bridge type fitted to the guitar. The `saddle_height[1..6]` and `compensation_bass/treble` parameters ([`§2.8`](./guitar_setup_parameters_v3.md#28-bridge--saddle)) are the targets; the tools below are the actuators.

### 4.1 Saddle Height Adjustment — By Bridge Type

| Bridge Type | Tool | Parameter |
|-------------|------|-----------|
| Tune-o-matic (Nashville/ABR) | Small flathead screwdriver or thumbwheel (per post) | [`saddle_height[1..6]`](./guitar_setup_parameters_v3.md#28-bridge--saddle) |
| Fender hardtail / vintage trem | 1.5 mm or 2 mm hex key (per saddle block) | [`saddle_height[1..6]`](./guitar_setup_parameters_v3.md#28-bridge--saddle) |
| Floyd Rose / locking trem | 3 mm hex key (post height) + 2.5 mm for fine saddle | [`saddle_height[1..6]`](./guitar_setup_parameters_v3.md#28-bridge--saddle) |
| Wraparound / fixed | Overall post height only — Phillips or Allen per bridge | [`saddle_height[1..6]`](./guitar_setup_parameters_v3.md#28-bridge--saddle) |

**Problems addressed:** [P-07](./guitar_setup_parameters_v3.md#p-07--action-too-high-across-the-whole-neck), [P-09](./guitar_setup_parameters_v3.md#p-09--action-too-low-buzzing-at-normal-playing), [P-05](./guitar_setup_parameters_v3.md#p-05--buzz-only-on-wound-strings) (bass-side saddle adjustment only).

---

### 4.2 Saddle Compensation (Intonation) Adjustment — By Bridge Type

| Bridge Type | Tool | Parameter |
|-------------|------|-----------|
| Fender-style | Phillips screwdriver (intonation screw per saddle) | [`compensation_bass / treble`](./guitar_setup_parameters_v3.md#28-bridge--saddle) |
| Tune-o-matic | Small flathead screwdriver (saddle fore/aft) | [`compensation_bass / treble`](./guitar_setup_parameters_v3.md#28-bridge--saddle) |
| Floyd Rose | 3 mm hex key (saddle lock screw + fore/aft adjust) | [`compensation_bass / treble`](./guitar_setup_parameters_v3.md#28-bridge--saddle) |

**Problems addressed:** [P-10](./guitar_setup_parameters_v3.md#p-10--sharp-intonation-at-every-fret), [P-11](./guitar_setup_parameters_v3.md#p-11--flat-intonation-at-every-fret), [T-04](./guitar_setup_parameters_v3.md#t-04--intonation-shifts-after-changing-tuning-not-covered-by-compensation).

> **Post-MVP note:** The `compensation_inharmonicity[n]` parameter ([`§2.8`](./guitar_setup_parameters_v3.md#28-bridge--saddle)) — which adjusts compensation for tension-dependent string stiffness — is calculated by the model but requires the same physical adjustment tools as standard compensation.

---

## 5. Fret Work Tools

The model distinguishes between fret issues that are within setup scope (dressable) and those requiring escalation. [P-03](./guitar_setup_parameters_v3.md#p-03--buzz-only-in-first-position-frets-15), [P-04](./guitar_setup_parameters_v3.md#p-04--buzz-only-at-high-frets-above-fret-12), and high-fret fall-off work ([`§2.9 falloff_amount`](./guitar_setup_parameters_v3.md#29-fret-plane-topology)) are within scope with the tools below. [P-12](./guitar_setup_parameters_v3.md#p-12--intonation-correct-at-12th-fret-wrong-at-other-frets) (intonation from uneven frets) and [P-21](./guitar_setup_parameters_v3.md#p-21--truss-rod-at-or-near-limit) are escalation flags — no setup tool addresses them.

### 5.1 Fret Levelling Beam / Levelling File

A long flat file or diamond-surfaced beam used to level high frets by removing material from the crown until all frets are at equal height.

| Use | Parameter / Section | Problem |
|-----|---------------------|---------|
| Level a proud fret causing first-position buzz | [`§2.9 fret_height_map[2..5]`](./guitar_setup_parameters_v3.md#29-fret-plane-topology) | [P-03](./guitar_setup_parameters_v3.md#p-03--buzz-only-in-first-position-frets-15) |
| Level high frets in upper register | [`§2.9 fret_height_map[12..N]`](./guitar_setup_parameters_v3.md#29-fret-plane-topology) | [P-04](./guitar_setup_parameters_v3.md#p-04--buzz-only-at-high-frets-above-fret-12) |
| Create or extend `falloff_amount` from `falloff_onset_fret` onward | [`§2.9 falloff_onset_fret`, `falloff_amount`](./guitar_setup_parameters_v3.md#29-fret-plane-topology) | [P-04](./guitar_setup_parameters_v3.md#p-04--buzz-only-at-high-frets-above-fret-12), [P-14](./guitar_setup_parameters_v3.md#p-14--notes-choke-or-die-when-bending) |

> **Fall-off and tension ([P-14](./guitar_setup_parameters_v3.md#p-14--notes-choke-or-die-when-bending)):** Lower tunings reduce string tension, causing strings to deflect farther laterally during bends — increasing the risk of fretting out. A `falloff_amount` sufficient at E standard may be marginal at Drop C. When `tuning_preset_id` changes to a lower tuning and bend choking is reported, the levelling beam is the tool used to increase fall-off from `falloff_onset_fret` forward.

---

### 5.2 Crowning File

After levelling removes the apex of a fret crown, a crowning file re-rounds the top of the fret to restore a precise contact point. Without this step, a levelled fret produces intonation errors.

| Use | Parameter / Section | Problem |
|-----|---------------------|---------|
| Restore fret crown profile after any levelling work | [`§2.9 fret_height_map[n]`](./guitar_setup_parameters_v3.md#29-fret-plane-topology) | [P-12](./guitar_setup_parameters_v3.md#p-12--intonation-correct-at-12th-fret-wrong-at-other-frets) (prevention) |

> A fret left flat-topped after levelling will produce the same symptom as [P-12](./guitar_setup_parameters_v3.md#p-12--intonation-correct-at-12th-fret-wrong-at-other-frets) — intonation error at specific fret positions. Crowning is a mandatory follow-on to any levelling operation.

---

### 5.3 Fret Erasers / Polishing Papers

Progressively finer abrasive papers or rubber erasers used to polish fret crowns after levelling and crowning. Rough fret surfaces accelerate string wear and produce inconsistent feel and sustain.

| Use | Parameter / Section |
|-----|---------------------|
| Polish fret crowns after levelling and crowning | [`§2.9 fret_height_map[n]`](./guitar_setup_parameters_v3.md#29-fret-plane-topology) |

> Typical sequence: 400 grit → 600 grit → 1000 grit → fret eraser (ultra-fine). Used with masking tape protecting the fretboard surface (§5.4 below).

---

### 5.4 Masking Tape

Protects the fretboard surface during any fret work. Applied to the board between frets before levelling, crowning, or polishing begins.

| Use | Parameter / Section |
|-----|---------------------|
| Fretboard protection during all fret work | [`§2.9`](./guitar_setup_parameters_v3.md#29-fret-plane-topology) — any fret-plane topology work |

---

## 6. General Workshop Tools

### 6.1 String Winder

Speeds up string removal and installation. Required whenever the string set changes ([`§2.4`](./guitar_setup_parameters_v3.md#24-string-set)) or when the nut must be accessed for replacement or re-cutting.

---

### 6.2 Wire / String Cutters

Trim string ends after stringing. Required with every string change.

---

### 6.3 Guitar Support / Neck Rest

Holds the guitar stable in a consistent position during measurement and adjustment. Accurate feeler gauge readings at the 8th fret require the guitar not to shift between measurements.

> **Critical for datum work ([§2.7](./guitar_setup_parameters_v3.md#27-neck-setup)):** The flat-neck confirmation requires the guitar to be held horizontally as it would be in playing position, or flat on the bench. A neck cradle or padded workbench prevents positional inconsistency in relief measurements (`relief_bass`, `relief_treble`).

---

### 6.4 Magnification and Task Lighting

Fret rocker readings, nut slot inspection, and string contact point verification at the nut and saddle are difficult at scale without magnification. A loupe or magnified lamp is particularly useful for:

- Checking nut slot width against string gauge (§3.1)
- Verifying fret rocker contact during high-fret localisation ([P-04](./guitar_setup_parameters_v3.md#p-04--buzz-only-at-high-frets-above-fret-12))
- Inspecting string contact angle in the nut slot (`nut_splay_angle[1..6]`, [P-18](./guitar_setup_parameters_v3.md#p-18--tuning-instability--strings-drift-after-bending))

---

## 7. Tools Out of Scope — Luthier Escalation

The following conditions in the setup model cannot be resolved with setup tools and require luthier assessment. The model surfaces these as escalation flags rather than setup adjustments.

| Problem | Condition | Escalation |
|---------|-----------|------------|
| [P-21](./guitar_setup_parameters_v3.md#p-21--truss-rod-at-or-near-limit) | `rod_state = at_limit` and neck not flat | Heat straightening, fretboard planing, neck reset, or neck replacement |
| [P-17](./guitar_setup_parameters_v3.md#p-17--neck-feels-different-bass-side-vs-treble-side) | `\|relief_bass − relief_treble\| > 0.15 mm` (severe twist) | Neck reset or refret |
| [P-12](./guitar_setup_parameters_v3.md#p-12--intonation-correct-at-12th-fret-wrong-at-other-frets) | Intonation error at specific frets from uneven fret height | Full fret dress or PLEK |
| [P-19](./guitar_setup_parameters_v3.md#p-19--dead-spots-weak-sustain-on-specific-notes) | Dead spots from body/neck resonance | Outside setup scope entirely |

> **P-21 early detection ([§4.7](./guitar_setup_parameters_v3.md#47-truss-rod-problems-new-in-v30)):** The flat-neck datum step naturally surfaces rod-limit problems before any downstream parameters are set. If the rod reaches `at_limit` before the neck reaches flat, [P-21](./guitar_setup_parameters_v3.md#p-21--truss-rod-at-or-near-limit) is flagged immediately — preventing the technician from building a setup on a neck that cannot hold its geometry.

---

## 8. Quick-Reference Tool Map

| Tool | Sections | Problems |
|------|----------|----------|
| Feeler gauge | [§2.6](./guitar_setup_parameters_v3.md#26-nut), [§2.7](./guitar_setup_parameters_v3.md#27-neck-setup), [§2.11](./guitar_setup_parameters_v3.md#211-tension-profile-updated-in-v30) | [P-01](./guitar_setup_parameters_v3.md#p-01--buzz-on-open-strings-only), [P-02](./guitar_setup_parameters_v3.md#p-02--buzz-across-the-whole-neck), [P-08](./guitar_setup_parameters_v3.md#p-08--action-too-high-only-in-first-position), [P-13](./guitar_setup_parameters_v3.md#p-13--open-string-sharp-12th-fret-correct), [P-17](./guitar_setup_parameters_v3.md#p-17--neck-feels-different-bass-side-vs-treble-side) |
| Precision straight edge | [§2.7](./guitar_setup_parameters_v3.md#27-neck-setup) | [P-17](./guitar_setup_parameters_v3.md#p-17--neck-feels-different-bass-side-vs-treble-side), [P-21](./guitar_setup_parameters_v3.md#p-21--truss-rod-at-or-near-limit), [T-03](./guitar_setup_parameters_v3.md#t-03--guitar-buzzes-after-retuning-but-was-fine-before) |
| Fret rocker | [§2.9](./guitar_setup_parameters_v3.md#29-fret-plane-topology) | [P-03](./guitar_setup_parameters_v3.md#p-03--buzz-only-in-first-position-frets-15), [P-04](./guitar_setup_parameters_v3.md#p-04--buzz-only-at-high-frets-above-fret-12), [P-12](./guitar_setup_parameters_v3.md#p-12--intonation-correct-at-12th-fret-wrong-at-other-frets) |
| String action gauge | [§2.10](./guitar_setup_parameters_v3.md#210-player-profile) | [P-07](./guitar_setup_parameters_v3.md#p-07--action-too-high-across-the-whole-neck), [P-08](./guitar_setup_parameters_v3.md#p-08--action-too-high-only-in-first-position), [P-09](./guitar_setup_parameters_v3.md#p-09--action-too-low-buzzing-at-normal-playing), [P-14](./guitar_setup_parameters_v3.md#p-14--notes-choke-or-die-when-bending) |
| Digital calipers | [§2.4](./guitar_setup_parameters_v3.md#24-string-set), [§2.6](./guitar_setup_parameters_v3.md#26-nut), [§2.8](./guitar_setup_parameters_v3.md#28-bridge--saddle) | [P-01](./guitar_setup_parameters_v3.md#p-01--buzz-on-open-strings-only), [P-07](./guitar_setup_parameters_v3.md#p-07--action-too-high-across-the-whole-neck), [P-08](./guitar_setup_parameters_v3.md#p-08--action-too-high-only-in-first-position), [P-10](./guitar_setup_parameters_v3.md#p-10--sharp-intonation-at-every-fret), [P-18](./guitar_setup_parameters_v3.md#p-18--tuning-instability--strings-drift-after-bending) |
| Capo | [§2.6](./guitar_setup_parameters_v3.md#26-nut) | [P-03](./guitar_setup_parameters_v3.md#p-03--buzz-only-in-first-position-frets-15) |
| Electronic tuner | [§2.5](./guitar_setup_parameters_v3.md#25-tuning), [§2.7](./guitar_setup_parameters_v3.md#27-neck-setup), [§2.8](./guitar_setup_parameters_v3.md#28-bridge--saddle) | [P-10](./guitar_setup_parameters_v3.md#p-10--sharp-intonation-at-every-fret), [P-11](./guitar_setup_parameters_v3.md#p-11--flat-intonation-at-every-fret), [P-12](./guitar_setup_parameters_v3.md#p-12--intonation-correct-at-12th-fret-wrong-at-other-frets), [P-13](./guitar_setup_parameters_v3.md#p-13--open-string-sharp-12th-fret-correct), [T-04](./guitar_setup_parameters_v3.md#t-04--intonation-shifts-after-changing-tuning-not-covered-by-compensation) |
| Truss rod wrench | [§2.7](./guitar_setup_parameters_v3.md#27-neck-setup), [§2.11](./guitar_setup_parameters_v3.md#211-tension-profile-updated-in-v30) | [P-02](./guitar_setup_parameters_v3.md#p-02--buzz-across-the-whole-neck), [T-03](./guitar_setup_parameters_v3.md#t-03--guitar-buzzes-after-retuning-but-was-fine-before), [P-21](./guitar_setup_parameters_v3.md#p-21--truss-rod-at-or-near-limit) |
| Nut files | [§2.6](./guitar_setup_parameters_v3.md#26-nut) | [P-01](./guitar_setup_parameters_v3.md#p-01--buzz-on-open-strings-only), [P-08](./guitar_setup_parameters_v3.md#p-08--action-too-high-only-in-first-position), [P-18](./guitar_setup_parameters_v3.md#p-18--tuning-instability--strings-drift-after-bending) |
| Nut lubricant | [§2.6](./guitar_setup_parameters_v3.md#26-nut) | [P-18](./guitar_setup_parameters_v3.md#p-18--tuning-instability--strings-drift-after-bending) |
| Nut blanks + adhesive | [§2.6](./guitar_setup_parameters_v3.md#26-nut) | [P-13](./guitar_setup_parameters_v3.md#p-13--open-string-sharp-12th-fret-correct), [P-18](./guitar_setup_parameters_v3.md#p-18--tuning-instability--strings-drift-after-bending) |
| Saddle height tools | [§2.8](./guitar_setup_parameters_v3.md#28-bridge--saddle) | [P-05](./guitar_setup_parameters_v3.md#p-05--buzz-only-on-wound-strings), [P-07](./guitar_setup_parameters_v3.md#p-07--action-too-high-across-the-whole-neck), [P-09](./guitar_setup_parameters_v3.md#p-09--action-too-low-buzzing-at-normal-playing) |
| Compensation tools | [§2.8](./guitar_setup_parameters_v3.md#28-bridge--saddle) | [P-10](./guitar_setup_parameters_v3.md#p-10--sharp-intonation-at-every-fret), [P-11](./guitar_setup_parameters_v3.md#p-11--flat-intonation-at-every-fret), [T-04](./guitar_setup_parameters_v3.md#t-04--intonation-shifts-after-changing-tuning-not-covered-by-compensation) |
| Fret levelling beam | [§2.9](./guitar_setup_parameters_v3.md#29-fret-plane-topology) | [P-03](./guitar_setup_parameters_v3.md#p-03--buzz-only-in-first-position-frets-15), [P-04](./guitar_setup_parameters_v3.md#p-04--buzz-only-at-high-frets-above-fret-12), [P-14](./guitar_setup_parameters_v3.md#p-14--notes-choke-or-die-when-bending) |
| Crowning file | [§2.9](./guitar_setup_parameters_v3.md#29-fret-plane-topology) | [P-12](./guitar_setup_parameters_v3.md#p-12--intonation-correct-at-12th-fret-wrong-at-other-frets) (prevention) |
| Fret erasers / polish | [§2.9](./guitar_setup_parameters_v3.md#29-fret-plane-topology) | Post-levelling polish |
| Masking tape | [§2.9](./guitar_setup_parameters_v3.md#29-fret-plane-topology) | Fretboard protection during all fret work |

---

## 9. Document Relationships

This document maps physical tools to the parameter model. It is consumed by the guide and optimisation engine to populate fix procedures and adjustment costs.

```
setup_tools_reference.md (this document)
  ↓ maps tools to parameters defined in
guitar_setup_parameters_v3.md §2.x (all parameter groups) + §4.x (all problems)

  ↓ provides tool data consumed by
setup_guide_architecture.md §4 — SetupTool schema + FixStep.tools_required
setup_optimisation_engine.md §2.3 — adjustment_cost scoring component
agent_access_spec.md §4.1 — tools_required field per problem record

  ↓ provides learning_value and cost_range consumed by
setup_guide_architecture.md §7.2 — SkillTierFilter (learnable tool inventory)
feature_layers.md — Layer 2 skill pathways (tool selection guidance)
```

| Column added in v1.1 | Consumed by |
|---|---|
| `risk_if_wrong` | [`setup_guide_architecture.md §4`](./setup_guide_architecture.md#4-data-structures) — `FixStep.warning` |
| `learning_value` | [`setup_guide_architecture.md §7.3`](./setup_guide_architecture.md#73-skilltierfiler-component) — learnable tool inventory; [`feature_layers.md`](./feature_layers.md) — Layer 2 skill pathways |
| `cost_range` | [`setup_guide_architecture.md §4`](./setup_guide_architecture.md#4-data-structures) — `SetupTool.cost_range`; [`agent_access_spec.md §4.1`](./agent_access_spec.md#41-problem-records) — `tool_cost_range` field |

---

*Document version 1.1 — derived from [`guitar_setup_parameters_v3.md`](./guitar_setup_parameters_v3.md).*
*v1.1 changes: `risk_if_wrong`, `learning_value`, and `cost_range` columns added to all tool sections; §9 Document Relationships added; companion document list expanded.*
*All section and problem references link to v3.1 of the setup parameter model.*
