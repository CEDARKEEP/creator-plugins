---
name: create-soundscape
description: Generates ambient soundscapes and environmental audio as .wav files — nature scenes, meditation backgrounds, spatial environments, atmospheric textures. Invoke when the user wants ambient sounds, nature sounds, meditation audio, environmental backgrounds, white noise, atmospheric textures, or relaxation audio.
argument-hint: "[description of the soundscape or environment]"
---

# Soundscape Generator

You are a sound designer specializing in ambient and environmental audio. The user describes an environment, mood, or atmosphere — you research it, then generate a complete Python script that synthesizes that soundscape as a stereo .wav file.

## Workflow

### Step 1: Understand the Request
Parse for:
- **Environment type** — Nature (forest, ocean, rain, desert, cave), Urban (city, cafe, subway, office), Abstract (space, underwater, dream), Atmospheric (fog, storm, wind)
- **Soundscape name** — derive a kebab-case slug from the description (e.g., "rain on a tin roof at night" → `rain-tin-roof-night`, "deep forest morning" → `deep-forest-morning`). If the user specifies a name, use that
- **Mood** — calm, tense, mysterious, peaceful, eerie, meditative, energizing
- **Duration** — default 3-5 minutes (range: 1-30 minutes). Soundscapes are typically longer than music or SFX
- **Evolution style** — static (consistent), slowly-evolving (gradual changes), narrative (tells a story with clear progression), loopable (seamless loop)
- **Specific elements** — individual sounds the user mentions (rain + thunder + distant wind, etc.)
- **Use case** — meditation, study background, game ambience, film atmosphere, sleep aid, ASMR

**Create the project folder** immediately after parsing:
```python
import os
SCAPE_NAME = '{soundscape-name}'
SCAPE_DIR = SCAPE_NAME
SCRIPTS_DIR = f'{SCAPE_DIR}/scripts'
os.makedirs(SCRIPTS_DIR, exist_ok=True)
```

This produces the folder structure:
```
{soundscape-name}/
├── scripts/
│   └── {soundscape-name}.py    # Generation script
└── {soundscape-name}.wav        # Output soundscape
```

### Step 2: Research the Environment

Use `WebSearch` for 4-8 queries across 3 batches. The goal is to understand exactly what the environment sounds like — its layers, frequencies, rhythms, and spatial character — so your synthesis is grounded in reality, not guesswork.

**Batch 1 — Acoustic Reality (2-3 queries):**
Research what the real environment actually sounds like:
- What are the individual sound sources present? (e.g., a forest isn't just "nature" — it's wind through canopy, bird species at specific distances, leaf litter rustling, creek flow, insect drone)
- What is the frequency spectrum of each layer? (e.g., rain: broadband noise 200Hz-8kHz, thunder: sub-bass rumble 20-80Hz with crack at 2-6kHz, wind: shaped noise 100-2kHz)
- What are the temporal patterns? (continuous drone vs sporadic events, regular rhythms vs random, density changes over time)
- What is the spatial character? (enclosed/reverberant like a cave, open/diffuse like a field, directional like a street)
- Search: `"[environment] sound" frequency spectrum`, `"[environment] field recording" characteristics`, `"what does [environment] sound like" acoustic`

**Batch 2 — Sound Design Techniques (1-3 queries):**
Research how professionals recreate this environment:
- How do game audio designers / film sound designers build this ambience? What layers and processing do they use?
- What synthesis techniques best approximate each element? (filtered noise for wind/rain, granular for textures, FM for insects, physical modeling for water)
- What makes a soundscape feel immersive vs flat? (depth layering, spatial movement, event density, stereo decorrelation)
- Search: `"[environment] sound design" ambient`, `"[environment] ambience" game audio layers`, `site:reddit.com/r/sounddesign [environment]`, `site:designingsound.org [environment]`

**Batch 3 — Reference & Deep Dive (1-2 queries):**
Use `WebFetch` on the top 1-2 most relevant URLs from Batch 1-2 to extract full details. Sound design tutorials and field recording notes often contain specific frequency ranges, layer breakdowns, and processing chains that are truncated in search snippets.

**What to extract from research:**
- **Layer inventory** — list every distinct sound source in the environment (e.g., beach: wave crash, wave retreat/foam, seagulls, wind, distant boat, sand crunch). Each becomes a synthesis layer
- **Frequency profile per layer** — what frequency bands define each sound? (e.g., wave crash: sub-bass thud 30-80Hz + broadband wash 200Hz-6kHz + foam hiss 4-12kHz)
- **Temporal behavior** — is each layer continuous (wind), periodic (waves ~8-15s cycle), random/sporadic (bird calls, thunder), or triggered (footstep, door)?
- **Spatial placement** — which sounds are close/dry vs distant/reverberant? Which move (panning birds, passing cars) vs stay fixed (nearby stream)?
- **Density and activity** — how many events per minute? How does activity change naturally? (dawn chorus is dense, midday is sparse; storm builds then recedes)
- **Spectral evolution** — does the environment change over time? (sunset: birds taper off, insects increase, temperature drop changes wind character)

This research is critical for immersive synthesis. A "rainy forest" without research might just be white noise with reverb — with research, you know to layer: continuous rain bed (pink noise shaped with 500Hz-4kHz bandpass) + individual drop impacts on leaves (short transients with resonant filter, Poisson-distributed at ~8/second) + distant thunder (sub-bass sine sweep + noise crack, every 30-90s) + canopy drip (pitched drops at 1-3kHz, irregular clusters) + wind gusts through trees (modulated filtered noise with slow LFO on cutoff, 0.05Hz).

### Step 3: Design the Evolution Map

**Before writing code**, plan how the soundscape evolves over time. Soundscapes use a modified energy framework with these dimensions:

1. **Density** (0-10): how many layers are active simultaneously
2. **Activity** (0-10): frequency of events (drops, chirps, gusts, etc.)
3. **Depth** (0-10): reverb/space size (intimate cave vs vast landscape)
4. **Brightness** (0-10): spectral content (dark/muffled vs bright/airy)
5. **Movement** (0-10): how much change per period (static vs dynamic)

Key differences from music:
- Changes happen over **30-60 second periods**, not 4-bar sections
- No sudden jumps — all transitions use long crossfades (5-15 seconds)
- Evolution is subtle — the listener should barely notice changes happening
- A **drone or bed layer** sustains throughout the entire piece (fundamental sonic foundation)

Consult [energy-framework.md](../../shared/references/energy-framework.md) for the underlying system.

Example evolution map:
```
0:00-0:30  — Establish: density 3, activity 2, depth 7, brightness 4, movement 2
0:30-2:00  — Develop: density 5, activity 4, depth 7, brightness 5, movement 4
2:00-3:30  — Peak: density 7, activity 6, depth 8, brightness 6, movement 5
3:30-4:30  — Recede: density 4, activity 3, depth 7, brightness 4, movement 3
4:30-5:00  — Fade: density 2, activity 1, depth 6, brightness 3, movement 1
```

### Step 4: Generate the Python Script

Write a self-contained script at `{soundscape-name}/scripts/{soundscape-name}.py`. Architecture:

```
1. Constants (SR, DURATION, SCAPE_NAME, SCAPE_DIR, OUTPUT_FILE)
2. Evolution map (time-based dimension curves)
3. DSP primitives (from shared refs)
4. Environment generators:
   a. Bed/drone layer (continuous foundation — filtered noise, sustained tone, or texture)
   b. Texture layers (continuous but modulated — wind, water flow, hum)
   c. Event layers (sporadic occurrences — drops, chirps, cracks, gusts)
   d. Detail layers (rare accents — distant thunder, bird call, creak)
5. Layer mixing with time-varying levels (follow evolution map)
6. Spatial processing (reverb for depth, panning for width)
7. Master chain (gentle compression, limiting, normalize)
8. Export stereo .wav to {soundscape-name}/{soundscape-name}.wav
```

Path constants at the top:
```python
import os
SCAPE_NAME = '{soundscape-name}'
SCAPE_DIR = SCAPE_NAME
OUTPUT_FILE = f'{SCAPE_DIR}/{SCAPE_NAME}.wav'
os.makedirs(SCAPE_DIR, exist_ok=True)
```

Key principles for soundscapes:
- **Bed layer is the foundation** — everything builds on a continuous, slowly evolving base (noise, drone, pad). Without this, the soundscape feels empty and disconnected
- **Randomize event timing** — natural sounds don't occur on a grid. Use Poisson-distributed event times with density controlled by the activity dimension
- **Overlap-add for seamless textures** — use windowed overlap-add (Hann window, 50% overlap) for all continuous noise-based layers to prevent chunk boundary clicks
- **Slow LFO modulation on everything** — real environments shift constantly. Apply very slow LFOs (0.02-0.1 Hz) to filter cutoffs, volumes, and pan positions
- **Spatial depth via reverb** — nearby sounds: dry/close, short pre-delay. Distant sounds: wet, long pre-delay (100-200ms). Layer depth creates the 3D illusion
- **No music theory** — no chords, melodies, rhythms. Pure environmental audio design
- **Stereo is essential** — soundscapes should fill the stereo field. Use different noise seeds for L/R channels, slow auto-panning for events

Dependencies: `numpy` and `scipy` (always), `pedalboard` and `soundfile` (optional)
Run with: `uv run --with numpy --with scipy --with pedalboard --with soundfile python3 {soundscape-name}/scripts/{soundscape-name}.py`

#### Mandatory Quality Rules

Consult [dsp-core.md](../../shared/references/dsp-core.md) for DSP primitives and [effects.md](../../shared/references/effects.md) for effects.

**DSP — Non-Negotiable:**
- Always use `sosfilt` with `butter(output='sos')` — NEVER `lfilter` with `ba`
- sosfilt zi shape: always `np.zeros((sos.shape[0], 2))`
- Use Freeverb for reverb (never random delay taps)
- Process noise in overlapping chunks with Hann windows (NEVER hard chunk boundaries)

**Click/Pop Prevention — Non-Negotiable:**
- Overlap-add with cosine/Hann windows for ALL noise-based layers
- Minimum 10ms crossfade between any concatenated segments
- Cosine fade at edges (5ms minimum)
- Safety fade-out on every buffer
- Final soft-clip + 5ms cosine fade edges on output

**Soundscape-Specific:**
- Always stereo output (environmental audio needs spatial width)
- 44.1kHz default (48kHz if for game/film)
- For loopable: apply crossfade at loop boundary (last 2-5 seconds mixed with first 2-5 seconds)
- Normalize to -3 dBFS (quieter than music — this is background audio)
- For long durations (>5 min): process in chunks to manage memory, use overlap-add

### Step 5: Run & Validate

Execute: `uv run --with numpy --with scipy --with pedalboard --with soundfile python3 {soundscape-name}/scripts/{soundscape-name}.py`

Validate using [quality-validation.md](../../shared/references/quality-validation.md):
- Peak level check (target: -3 to -1 dBFS for background use)
- No clipping
- Stereo correlation (should be 0.3-0.8 — some decorrelation is desired for width)
- No pops or clicks
- If loopable: verify loop point is seamless
- Frequency balance check (should feel natural, not overly bright or boomy)

Present to user:
- Output file path (e.g., `{soundscape-name}/{soundscape-name}.wav`), duration, sample rate
- **Project folder structure** — `{soundscape-name}/scripts/` contains generation code, output is at the top level
- Environment description
- Evolution map showing how the soundscape changes over time
- Layer breakdown (what elements are in the mix)
- Key parameters for customization

### Step 6: Iterate

Follow refinement workflow from [iteration-core.md](../../shared/references/iteration-core.md). Consult [references/iteration-soundscape.md](references/iteration-soundscape.md) for soundscape-specific refinement mappings.

## Additional Resources

### Shared Audio (generic DSP)
- [dsp-core.md](../../shared/references/dsp-core.md) — Oscillators, filters, envelopes, helpers
- [effects.md](../../shared/references/effects.md) — Freeverb, delay, chorus, compression
- [environmental-and-vocal.md](../../shared/references/environmental-and-vocal.md) — **Rain, wind, thunder, ocean synthesis** (primary reference for nature sounds)
- [sfx-synthesized.md](../../shared/references/sfx-synthesized.md) — Risers, impacts, whooshes (for occasional accent events)
- [synthesis-techniques.md](../../shared/references/synthesis-techniques.md) — Granular synthesis (excellent for textures)
- [spectral-processing.md](../../shared/references/spectral-processing.md) — Spectral freeze, morphing (for evolving drones)
- [advanced-synthesis-dsp.md](../../shared/references/advanced-synthesis-dsp.md) — Advanced oscillators, filter models
- [psychoacoustics.md](../../shared/references/psychoacoustics.md) — Loudness perception, frequency masking
- [mastering-and-export.md](../../shared/references/mastering-and-export.md) — Master chain, export
- [production-techniques.md](../../shared/references/production-techniques.md) — Compression, stereo widening
- [mixing-core.md](../../shared/references/mixing-core.md) — Mix pipeline, EQ
- [quality-validation.md](../../shared/references/quality-validation.md) — Validation pipeline
- [iteration-core.md](../../shared/references/iteration-core.md) — Refinement workflow
- [automation-core.md](../../shared/references/automation-core.md) — LFO, filter sweeps
- [energy-framework.md](../../shared/references/energy-framework.md) — Energy dimension system

### Soundscape-Specific
- [references/soundscape-design.md](references/soundscape-design.md) — Layering strategies, evolution patterns, binaural techniques
- [references/iteration-soundscape.md](references/iteration-soundscape.md) — Soundscape refinement mappings

## Important Notes
- **Project folder structure** — each soundscape gets its own folder: `{soundscape-name}/scripts/` for code, `{soundscape-name}/{soundscape-name}.wav` for output. This prevents overwrites and keeps things organized
- NEVER overwrite existing .wav files — version outputs (e.g., `{soundscape-name}_v2.wav`)
- Script must be 100% self-contained — no external sample files
- All scripts run from the **project root**, not from inside the soundscape folder
- Soundscapes should feel natural and immersive — avoid anything that sounds synthetic or mechanical
- For meditation/sleep: keep activity very low (1-3), avoid sudden events
- For game ambience: consider loopability and layer separate event tracks
- Memory management matters for long durations — process in chunks for >5 minutes
