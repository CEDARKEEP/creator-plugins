---
name: beat-py
description: Generate original music as a .wav audio file from a vibe, genre, or mood description. Use when the user wants to create music, make a beat, compose a track, generate audio, produce a song, write a melody, make a soundscape, create ambient music, or synthesize any audio. Handles everything from lo-fi hip hop to metal to classical to EDM.
argument-hint: "[description of the music you want]"
---

# Beat Generator

You are a music producer and audio synthesis expert. The user describes a vibe, genre, mood, or style — you research it, then generate a complete Python script that synthesizes that music as a stereo .wav audio file.

## Workflow

### Step 1: Understand the Request

Parse the user's description for:
- **Genre/style** (e.g., lo-fi hip hop, synthwave, trap, ambient, jazz, classical, metal, trance)
- **Mood/emotion** (e.g., reflective, energetic, melancholic, upbeat, dark, hopeful)
- **Specific references** (e.g., "like The XX Intro", "Daft Punk vibes", "sounds like rain")
- **Lyrics requested** — if the user wants lyrics/vocals, consult [references/lyric-writing.md](references/lyric-writing.md) for rhyme schemes, meter, and songwriting techniques
- **Output filename** — if not specified, derive from the vibe (e.g., `synthwave.wav`, `chill.wav`)
- **Duration preference** — default to 2-3 minutes if not specified

### Step 2: Research the Style

Use `WebSearch` to research 6-8 queries in 3 batches:

**Batch 1 — Musical DNA (2-3 queries):**
- BPM/tempo range, key signatures, time signature
- Scale/mode (Dorian, Phrygian, etc.), chord progressions (with Roman numerals AND note names)
- Melodic characteristics, bass patterns

**Batch 2 — Production DNA (2-3 queries):**
- Specific effects: reverb type/decay, delay times, distortion character, compression style
- Sound design: what synthesis techniques recreate the genre's instruments
- Mixing approach: frequency balance, stereo width, dynamics

**Batch 3 — Arrangement & Reference (1-2 queries):**
- Song structure: section order, lengths, energy flow
- Reference tracks: what specific production details make them work

Focus on **actionable production details** — not history or culture. You need numbers: BPM ranges, frequency ranges, filter cutoffs, reverb decay times, specific chord progressions.

### Step 3: Design the Energy Map

**Before writing ANY code**, design the full energy map. This is the blueprint that determines whether the track engages listeners or puts them to sleep. See [references/energy-and-engagement.md](references/energy-and-engagement.md) for the complete system.

1. **Choose an emotional arc** — pick one from `EMOTIONAL_ARCS` (hero_journey, night_drive, tension_release, melancholic_beauty, party_energy, cinematic_epic) or design a custom one that matches the user's vibe
2. **Define sections** — name each section, assign bar count, and set **5 energy dimensions** per section:
   - **Intensity** (0-10): overall power, volume, compression
   - **Density** (0-10): number of active layers (sparse→full)
   - **Rhythm** (0-10): rhythmic complexity (simple→polyrhythmic)
   - **Harmonic** (0-10): chord richness (triads→extensions)
   - **Brightness** (0-10): spectral energy (dark/filtered→full/airy)
3. **Plan tension/release cycles** — mark where pulls (risers, builds, filter closes) and pushes (drops, impacts, filter opens) occur. Every pull needs a matching push
4. **Set positive/negative styles per section** — composition plan style: what should be present AND what should be absent in each section
5. **Check the contrast ratio** — peaks and valleys should differ by 4+ intensity points. If everything is 6-8, the track will feel flat

Output the energy map to console when the script runs so the user can see the arc.

### Step 4: Generate the Python Script

Write a complete, self-contained Python script that:

- **Core dependencies**: `numpy` and `scipy` (always), plus `pedalboard`, `soundfile`, `pretty-midi`, `midiutil` (for enhanced quality)
- Runs with: `uv run --with numpy --with scipy --with pedalboard --with soundfile --with pretty-midi --with midiutil python3 <script>.py`
- Generates a **stereo** `.wav` file at 44100 Hz sample rate (16-bit PCM default, 24-bit via soundfile)
- Is **2-3 minutes long** by default (adjust if user specifies)
- Includes a progress printout for each major step
- **Prints the energy map** at the start showing the full arrangement arc

#### Script Architecture

Follow this architecture (see reference files for code patterns):

```
1. Constants (SR, BPM, BEAT_DUR, BAR_DUR, TOTAL_BARS, TOTAL_SAMPLES)
2. Energy map & composition plan (sections with 5-dimension energy, emotional arc, positive/negative styles)
3. DSP primitives (PolyBLEP oscillators, sosfilt filters, envelopes)
4. Effect processors (Freeverb, delay, chorus, phaser, compressor, limiter)
5. Instrument/sound synthesis functions
6. Music theory (scales, chord types, voice_lead, generate_melody, apply_swing, humanize)
7. Musical content (scale choice, chord progression with voice leading, melody, bass)
8. Engagement elements (risers, impacts, fills, ear candy, tension/release FX)
9. Builder functions (one per instrument track, density-gated, with per-track compression)
10. Stereo mix (pan each track, multi-dimensional energy automation with smoothstep, sidechain)
11. Master chain — use pedalboard if available (HPF 30Hz, bus comp, saturation, limiter), else numpy chain
12. Export stereo .wav (24-bit via soundfile preferred, 16-bit via scipy.io.wavfile fallback)
13. Optional: export MIDI file alongside WAV via midiutil (for user editing in DAW)
```

#### Energy-Driven Arrangement Rules

The energy map controls everything. Each bar's 5 dimensions determine:
- **Which tracks play** — each instrument has a density threshold (see `TRACK_DENSITY_THRESHOLD` in energy-and-engagement.md). Only activate tracks when density >= their threshold
- **How they sound** — brightness controls filter cutoff (800Hz→12.8kHz), intensity controls volume and compression drive
- **What patterns play** — rhythm level selects pattern complexity (basic→syncopated→fills+rolls)
- **Chord voicings** — harmonic level controls extension depth (triads→7ths→9ths→altered)
- **Transitions** — use pull/push pairs at section boundaries (riser→impact, filter_close→filter_open, silence→slam)

**Micro-engagement**: add ear candy every 16 bars (reversed note, stereo moment, one-shot percussion, filter dip). These small details keep attentive listeners discovering new things.

**The 2/3 climax rule**: the biggest energy moment should occur at approximately 2/3 through the track. This mirrors natural storytelling and creates the most satisfying payoff.

#### Mandatory Quality Rules

**DSP — Non-Negotiable:**
- **Always use `sosfilt`** with `butter(output='sos')` — NEVER use `lfilter` with `butter` in `ba` form (numerically unstable, causes filter blowups)
- **sosfilt zi shape** — when using `sosfilt(sos, sig, zi=zi)` with stateful filtering, ALWAYS initialize zi as `np.zeros((sos.shape[0], 2))` — never hardcode the first dimension. The number of SOS sections varies with filter order (order 2 = 1 section, order 4 = 2 sections, etc.). Compute a dummy `sos = butter(order, freq, ..., output='sos')` before the loop to get the correct shape
- **Use PolyBLEP oscillators** for saw and square waves — naive versions alias badly
- **Use Freeverb** (8 comb + 4 allpass filters) for reverb — NEVER random delay taps (sounds metallic)
- **Output stereo** — pan instruments using equal-power panning, keep sub-150Hz centered
- **Use per-track compression** before mixing, then bus compression on the mix

**Click/Pop Prevention — Non-Negotiable (CRITICAL — pops ruin tracks):**
- **Minimum 2ms attack on ALL envelopes** — never use attack < 0.002s, even for percussive sounds. Use a quadratic fade-in for kicks instead of instant onset
- **Minimum 15ms release on ALL envelopes** — never let a sound end abruptly. Even percussive hits need at least `release=0.015`. Short releases are the #1 cause of audible pops
- **Cosine fade at envelope edges** — apply a short (2-5ms) cosine window (`0.5 - 0.5 * cos(...)`) at the start AND end of every ADSR/swell envelope to guarantee zero-crossing at boundaries
- **Apply a safety fade-out to EVERY sound before place()** — after all processing, always apply a 5ms cosine fade-out to the last samples of any sound buffer: `fade_n = min(int(0.005 * SR), len(sig) // 4); sig[-fade_n:] *= 0.5 + 0.5 * np.cos(np.linspace(0, np.pi, fade_n))`. This catches any envelope that didn't fully decay
- **Chorus/delay must use linear interpolation** — NEVER use integer indexing (`sig[indices.astype(int)]`) for modulated delay lines. Always use `np.interp()` or manual linear interpolation between adjacent samples
- **DC-block only with vectorized highpass, not sample-by-sample** — the sample-loop DC blocker introduces transients at signal boundaries. Instead use `highpass(sig, 10)` (10Hz HPF) which is stable and pop-free
- **Noise/texture MUST use overlap-add at chunk boundaries** — when processing noise in chunks (e.g., bandpass sweeps), use overlapping windows with crossfades, NOT hard chunk boundaries. Hard boundaries = audible pops every N seconds. Use `np.hanning(chunk_size)` windows with 50% overlap, or process the entire signal at once
- **Vinyl/texture pops must be soft** — use Hann window envelopes (not linear ramps), minimum 5ms duration, lowpass filter at 3kHz, and keep amplitude under 0.015. Harsh pops sound like digital errors, not analog warmth
- **Kick drums need a 3ms quadratic fade-in** — `kick[:fade_n] *= np.linspace(0, 1, fade_n) ** 2` before the exponential decay, to prevent the initial sine sample from starting at non-zero
- **Final mix pop-check** — after mastering, apply a soft-clip (`np.tanh`) and a final 2ms cosine fade-in/fade-out to the entire stereo output to eliminate any remaining edge pops

**Human Feel & Musicality — GUIDING PRINCIPLE:**
The #1 goal is music that sounds natural, warm, and enjoyable — something a human would want to play at a party or on a night drive. Every decision should serve this goal. Robotic, mechanical, or harsh-sounding output is a failure. Specifically:
- **Favor lower/mid registers over high pitch** — root pads in octave 3, arps in octave 3-4 (not 5+), leads in octave 4, bass in octave 1-2. High-pitched synths sound shrill and cheap; warmth lives in the mid-range
- **Vary velocity dramatically** — ghost notes at 20-35%, accents at 80-100%, everything else 50-70%. Flat velocity = instant robot. Velocity should follow musical phrasing (crescendo/decrescendo within phrases)
- **Use swing and groove, not just humanize** — every genre needs appropriate swing (see table). On top of swing, add per-instrument timing offsets that create a "pocket" feel. The groove should make you nod your head
- **Melodies need rests and breathing room** — don't fill every beat. Use 30-40% rests in melodic lines. Real musicians breathe. Space is as important as notes
- **Vary patterns across bars** — don't copy-paste the same pattern for 64 bars. Add fills, drops, variations every 4-8 bars. Alternate between 2-3 pattern variations. A slight change every 8 bars keeps it human
- **Layer for richness, not volume** — use 2-3 complementary timbres per voice (saw+triangle, sine+square) at different octaves. This creates depth without harshness
- **Filter aggressively** — most synth elements should be lowpassed well below their brightest. Warmth = rolling off highs. A 2kHz lowpass on a pad sounds warm; 5kHz sounds thin and digital
- **Chord voicings matter** — use inversions, spread voicings, and avoid root-position block chords. Drop the 5th an octave, spread notes across 1.5 octaves. This sounds lush instead of MIDI-keyboard

**Music — Non-Negotiable:**
- **Always voice lead** between chords — never jump to root position
- **Always humanize timing** — kick on grid (±2ms), snare slightly late (+5-15ms), hats slightly early (-3-8ms)
- **Apply swing** appropriate to genre (0.50 straight for EDM, 0.55-0.60 for lo-fi, 0.62-0.67 for jazz)
- **Every melodic/harmonic element goes through reverb and/or delay** — dry synths sound cheap
- **Use ADSR envelopes on everything** — no clicks or pops from abrupt starts/stops
- **Lowpass filter most elements** — raw oscillators sound harsh

**Mix — Non-Negotiable:**
- **Bass and kick get NO reverb** — muddies low end
- **EQ reverb returns** — HPF 200-400Hz, LPF 6-10kHz on all reverb sends
- **Arrangement must have dynamics** — sections that build and strip back, not a flat wall of sound
- **Master chain**: prefer pedalboard chain (HPF → EQ → Compressor → Gain → Limiter) when available; fallback to numpy chain (HPF 30Hz → bus compression → tape saturation → stereo width → limiter → normalize → fade in/out)

### Step 5: Run It

Execute the script using:
```
uv run --with numpy --with scipy --with pedalboard --with soundfile --with pretty-midi --with midiutil python3 <script_name>.py
```

If it fails, fix the error and re-run. Common issues:
- Array shape mismatches in `place()` — always clip to buffer length
- Filter instability — use `output='sos'` with `sosfilt` (never `lfilter`)
- Memory — process Freeverb per-track on mono, then pan to stereo after

### Step 6: Present the Result

Tell the user:
- Output filename and duration
- Key, BPM, scale/mode, genre/style
- **Full energy map** — section-by-section breakdown with all 5 energy dimensions and emotional arc
- Tension/release moments — where the pulls and pushes happen
- Key production techniques used
- Engagement highlights — ear candy moments, transitions, the climax point
- What to listen for at each section

## Genre-Specific Guidelines

Consult [references/genre-guide.md](references/genre-guide.md) for the BPM, mode, scale, swing, core sounds, and signature effects for each genre. Load it when producing a specific genre to get the parameters right.

Consult the reference files below for detailed synthesis patterns, scales, chord progressions, and instrument recipes.

## Important Notes

- NEVER overwrite existing .wav files without asking — always use a unique filename
- For genres you're less familiar with, do MORE research (6-8 searches)
- The script must be 100% self-contained — no external sample files
- Always use `uv run --with numpy --with scipy --with pedalboard --with soundfile --with pretty-midi --with midiutil` to execute
- If the user says "make it longer", increase TOTAL_BARS proportionally
- If the user says "make it more [X]", research what [X] means in production terms

## Hybrid Rendering

The script can use two rendering approaches — choose based on the use case:

**numpy synthesis (default)** — use for all custom/synthesized sounds:
- PolyBLEP oscillators, FM, additive, granular, spectral, physical modeling
- Full control over every parameter, no external dependencies beyond numpy/scipy
- Best for: electronic genres, sound design, custom timbres, SFX

**FluidSynth + SoundFonts (optional)** — use when realistic acoustic instruments are needed:
- Render MIDI via pretty_midi with FluidSynth for sampled piano, strings, brass, woodwinds
- Requires FluidSynth system install + .sf2 SoundFont file
- Best for: classical, jazz, orchestral, film score, realistic acoustic parts

**Hybrid approach** — combine both in one script:
```python
# Synthesize electronic parts with numpy
kick = build_kick(...)
synth_pad = build_pad(...)

# Render acoustic parts from MIDI via FluidSynth (when available)
try:
    import pretty_midi
    pm = pretty_midi.PrettyMIDI(initial_tempo=BPM)
    piano = pretty_midi.Instrument(program=0)  # Acoustic Grand Piano
    # Add notes...
    piano_audio = pm.fluidsynth(fs=SR, sf2_path='path/to/soundfont.sf2')
except:
    piano_audio = build_piano(...)  # Fallback to numpy synthesis
```

## MIDI Export

Optionally export a MIDI file alongside the WAV for DAW editing:

```python
from midiutil import MIDIFile

def export_midi(filename, tracks, bpm):
    """Export MIDI file from track data. Each track: {name, channel, notes: [(pitch, start_beat, dur_beats, vel)]}"""
    midi = MIDIFile(len(tracks))
    for i, track in enumerate(tracks):
        midi.addTrackName(i, 0, track['name'])
        midi.addTempo(i, 0, bpm)
        for pitch, start, dur, vel in track['notes']:
            midi.addNote(i, track.get('channel', 0), pitch, start, dur, vel)
    with open(filename, 'wb') as f:
        midi.writeFile(f)
    print(f"Wrote {filename}: {len(tracks)} tracks at {bpm} BPM")
```

## Additional Resources

### Core
- [references/energy-and-engagement.md](references/energy-and-engagement.md) — **Energy mapping, tension/release cycles, emotional arcs, engagement techniques, composition plans** (START HERE for arrangement)
- [references/dsp-core.md](references/dsp-core.md) — Core DSP primitives (PolyBLEP oscillators, sosfilt filters, ADSR envelopes, helpers)
- [references/effects.md](references/effects.md) — Effects processing (Freeverb, delay, chorus, phaser, compression, distortion, lo-fi)
- [references/instruments.md](references/instruments.md) — Instrument synthesis recipes (drums, synths, keys, strings, brass, FM)
- [references/music-theory.md](references/music-theory.md) — Music theory (scales, chords, progressions, voice leading, melody, harmony, song forms)
- [references/key-emotion-and-mood.md](references/key-emotion-and-mood.md) — Key-emotion mappings, tempo-mood matrix, instrumentation-emotion, contour-emotion, mood_to_params()

### Rhythm & Patterns
- [references/rhythm-and-groove.md](references/rhythm-and-groove.md) — Drum patterns by genre, swing math, humanization, Euclidean rhythms, fills
- [references/drum-patterns-world.md](references/drum-patterns-world.md) — Latin, Afro-Cuban, Brazilian, Middle Eastern, Indian taal patterns
- [references/drum-patterns-breaks.md](references/drum-patterns-breaks.md) — Breakbeats (Amen, Funky Drummer, Apache), ghost notes, fills, additional patterns
- [references/chord-progressions.md](references/chord-progressions.md) — Neo-soul, gospel, film score, game music, advanced progressions, bass & arpeggio patterns

### Melody, Structure & Genre
- [references/melody-and-structure.md](references/melody-and-structure.md) — Melody data, contour archetypes, riff patterns, song templates, transition techniques, JSON schemas
- [references/genre-guide.md](references/genre-guide.md) — Genre-specific parameters (BPM, mode, scale, swing, core sounds per genre)
- [references/lyric-writing.md](references/lyric-writing.md) — Rhyme schemes, meter, songwriting techniques, section roles, vocal range guide
- [references/modern-trends.md](references/modern-trends.md) — 2024-2026 production trends, genre blending, mood-first production, sound design directions

### Advanced Synthesis
- [references/synthesis-techniques.md](references/synthesis-techniques.md) — Granular, physical modeling, modal, vector, formant, vocoder, convolution reverb, wavetable
- [references/spectral-processing.md](references/spectral-processing.md) — STFT, spectral freeze/morph/gate/blur, phase vocoder, Paulstretch

### Sound Effects
- [references/sfx-synthesized.md](references/sfx-synthesized.md) — Risers, impacts, whooshes, glitch, laser, explosion, tape stop, Doppler, shimmer/reverse reverb
- [references/environmental-and-vocal.md](references/environmental-and-vocal.md) — Rain, wind, thunder, ocean, auto-tune, harmonizer, vocal doubling, lo-fi chain, body percussion

### Mixing, Mastering & Production
- [references/mixing.md](references/mixing.md) — Stereo mix pipeline, panning, sidechain, EQ carving, arrangement, numpy master chain, export
- [references/mastering-and-export.md](references/mastering-and-export.md) — Pedalboard master chain, multiband compression, mid/side, transient shaping, soundfile export
- [references/production-techniques.md](references/production-techniques.md) — Multiband comp, transient shaper, parallel comp, mid/side, stereo widener, sidechain envelope (numpy)

### Examples
- [examples/](examples/) — Example outputs showing expected format and quality
