# pitchwheel

A single-file browser tool for exploring the relationship between scales and chords in equal temperament. No build step, no dependencies beyond Three.js (loaded from CDN). Open `index.html` directly.

---

## Overview

The page has three sections:

- **Helix** — a 3D pitch helix that reflects the current state of both columns at all times
- **Scale** (left column) — pick or generate a scale, then transform it
- **Chord** (right column) — select notes from that scale using a criterion, then transform the result

Everything updates live. Changes to the scale immediately propagate to the chord builder.

---

## Mod

The **Mod** field (top of the Scale column) sets the number of equal divisions per octave. Default is 12 (standard equal temperament). Valid range is 1–96.

Scales in the database are stored as chromatic-12 positions and remapped to the active mod via ratio rounding (`round(p × mod / 12)`). Positions that collide after rounding are deduplicated, so some scales will lose notes at low mod values.

When the generator was the last thing to set the scale, changing Mod re-runs the generator automatically. When the scale was set via the picker or the custom input, Mod changes leave the scale alone and just reinterpret it in the new tuning.

Audio is tuned so that step 0 = C0 (16.35 Hz) and one full octave (mod steps) = double the frequency, regardless of mod.

---

## Scale column

### Scale Database

A searchable library of ~270 named scales in eight categories: Major and minor, European, Asian, Indian, Modal, Jazz, Symmetric, and Misc. Click a pill to select it. The search box filters across all categories simultaneously.

### Positions input

Comma-separated absolute positions, e.g. `0,2,4,5,7,9,11`. This field is the shared sink for both the database picker and the generator — both write here when they produce a result. You can also type directly; doing so takes ownership and the picker/generator selection is cleared.

### Scale Generator

Three algorithms, selectable via tabs. All use the current Mod as the step count.

**Euclidean** — distributes `events` notes as evenly as possible across `mod` steps using the Björklund algorithm. `Offset` rotates the starting position.

**Clough-Douthett** — maximally even sets. Places note `i` at `floor(i × mod / events)`, offset by `offset` (mod mod). Produces the same result as Euclidean for many inputs but uses a different construction.

**Deep** — places note `i` at `(i × multiplicity + offset) mod mod`. For all `events` notes to be distinct, `multiplicity` should be coprime to `mod`. When it isn't, you get fewer unique positions than requested.

The generator readout shows the computed positions. Changing any generator parameter or switching tabs re-runs the generator immediately.

### Scale transform chain

Six operations, all toggleable on/off and drag-reorderable. Order matters — operations apply left-to-right in the current stack order.

| Step | What it does |
|------|--------------|
| **Root** | Shifts the interval-vector offset. Composes additively, so it works correctly wherever it sits in the chain. |
| **Mode** | Rotates the interval pattern. 0 = no rotation, 1 = one step right (e.g. Ionian → Dorian for a diatonic scale). |
| **Degree** | Shifts the starting index into the position sequence. Changes which note the scale starts on without reordering intervals. |
| **Invert** | Reverses the interval order around an axis index. Flips the shape while preserving note count. |
| **Mirror** | Reflects positions and re-anchors so the lowest pitch stays fixed. No axis parameter. |
| **Negative** | Negative harmony: reflects pitch classes around the axis halfway between root and perfect fifth. Axis = `(2 × root + fifth) mod mod`, where `fifth` is the nearest EDO approximation of 3:2. |

Root and Mode are on by default. The others are off.

The readout below shows the resulting interval vector and position array.

---

## Chord column

### Octave

Transposes chord playback and helix placement up by 0–9 octaves. Does not affect the position math — only audio output and helix rendering. Default is +5.

### Criterion

Defines which notes from the scale are selected to form the chord. Two modes:

**pos** — scale degree indices, comma-separated. `0,2,4` picks the 1st, 3rd, and 5th degrees above the anchor. Degrees are 0-indexed.

**int** — step intervals from the anchor degree. `2,2,3` walks up 2 scale steps, then 2 more, then 3.

**Degree** slider (−14 to +14) — shifts which scale degree serves as the anchor. 0 = root, 1 = second degree, negative values count from the end.

**Rotation** — rotates the interval pattern of the selected chord tones before returning them. Cyclic permutation of the voicing.

**Voices** — number of extra octave doublings added below the chord before the criterion is applied. Extends the chord downward by duplicating the scale.

### Chord transform chain

Same drag-reorder/toggle mechanism as the scale chain. Six operations:

| Step | What it does |
|------|--------------|
| **Position** | Shifts the starting index into the chord tones. Changes voicing/inversion without reordering intervals. |
| **Offset** | Transposes the chord by shifting the interval-vector offset. |
| **Mode** | Rotates the interval pattern of the chord tones. |
| **Invert** | Reflects each tone around an axis position, then sorts ascending. axis = 0 gives classic chord inversion. |
| **Mirror** | Reflects positions and re-anchors so the lowest pitch stays fixed. |
| **Negative** | Negative harmony applied to chord selection: negates the scale before selecting, and folds the shift degree. Reflection axis is taken from the Scale transform chain's Root value. |

Position is on by default. The others are off.

The chord Negative step is applied *before* note selection (not after), so the fold happens in scale-degree space rather than pitch space.

The readout shows the criterion used and the resulting chord positions.

---

## Helix

A 3D pitch helix rendered with Three.js. The horizontal angle encodes pitch class (position within an octave), the vertical axis encodes octave. Ten octave rings are always shown regardless of mod.

Color coding:

| Color | Meaning |
|-------|---------|
| Amber (bright) | Scale root (position 0) |
| Blue | All other scale tones |
| Red | Anchor degree (the degree the chord is built from) |
| Amber (dimmer) | Chord tones, connected into a closed polygon |

Chord tones are placed at their actual octave as set by the Octave slider, not folded back to pitch class. The polygon connects them in angular order around the wheel.

**Controls:** drag to orbit, scroll/pinch to zoom. Auto-rotation resumes 2.5 seconds after the last interaction. The ⏸ button pauses/resumes auto-rotation; the speed slider adjusts the rotation rate.

---

## Audio

Playback uses the Web Audio API with sine oscillators. No samples, no external audio files.

Scale: **▶ Play ascending** sequences the scale tones with a 280 ms gap. **▶ Play as chord** sounds all scale tones simultaneously.

Chord: **▶ Play as chord** sounds all chord tones together. **▶ Play arpeggio** sequences them with a 240 ms gap.

Both use the Octave slider value for pitch placement (C0 = step 0 in the current mod, C4/middle C = step 60 in mod 12).

---

## Data model

The internal representation uses two vector types:

**PositionVector** — a list of absolute cyclic positions plus a modulus. Represents a set of pitch classes or scale degrees.

**IntervalVector** — a list of step sizes plus an offset and modulus. Represents a sequence of intervals with a starting point. Positions and intervals are interconvertible.

The `select()` function is the core of chord building: given a source (scale) and a criterion (chord pattern), it returns the scale tones at the specified degrees or intervals. It handles all four combinations of position/interval source × position/interval criterion.

Scale transforms operate on interval vectors. Chord transforms operate on position vectors (the output of `select()`).

---

## No server required

The file is entirely self-contained except for:

- `https://fonts.googleapis.com/css2?family=IBM+Plex+Mono...` — fonts
- `https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js` — 3D rendering

Both are fetched at load time. If you need fully offline use, download those two resources and update the URLs.
