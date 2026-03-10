# Iteration & Refinement Core

Generic refinement workflow for iteratively improving generated audio. Surgical modification over full regeneration.

## Overview

When a user requests changes to generated audio, the goal is to make the minimum necessary edits to achieve the desired result. Full regeneration discards everything that already works — timing, spectral balance, effect tuning, gain staging — and risks producing something worse overall. Surgical modification preserves what is already good and changes only what needs to change.

## What to Preserve vs Regenerate

```python
PRESERVE_RULES = {
    'ALWAYS_PRESERVE': [
        'SAMPLE_RATE',            # Core render setting — never changes
        'DSP_PRIMITIVES',         # Core synthesis functions — never edit these
        'MASTER_CHAIN',           # Mastering chain order and general settings
        'FREQUENCY_SLOT_PLAN',    # Element frequency assignments
    ],

    'REGENERATE_WHEN': [
        'User says "completely different" or "start over"',
        'User requests a fundamentally different result AND structure AND processing',
        'More than 5 independent parameters would need to change',
        'User expresses dissatisfaction with overall quality ("this sounds bad", "not what I wanted at all")',
        'Requested changes would conflict with each other in the current architecture',
        'Original script has structural issues (e.g., broken processing graph, missing sections)',
    ],

    'PARTIAL_REGENERATE': {
        'different effect chain':    'Rebuild effect chains only. Keep all builder outputs and mix levels.',
        'different arrangement':     'Rebuild structure and layout. Keep all element timbres and mix settings.',
        'different element sound':   'Regenerate the target element builder. Keep volume, pan, and effects.',
        'different section':         'Regenerate only the target section bars of all tracks. Keep the rest intact.',
    },
}
```

## Surgical Edit Workflow

Follow this exact sequence for every refinement request:

### Step 1: Read the Existing Script

Load the full `.py` file. Identify the top-level constants section, builder functions, mixing section, and master chain.

### Step 2: Map Request to Parameters

Identify which parameters the user's request maps to. If the request is compound (e.g., "brighter and drier"), combine the parameter lists from all relevant entries.

### Step 3: Show Proposed Changes

Before editing, present a summary to the user:

```python
# Example change summary format:
PROPOSED_CHANGES = {
    'request': 'make it brighter',
    'changes': [
        {'param': 'FILTER_CUTOFF',  'current': 5000, 'proposed': 7500, 'location': 'line 42'},
        {'param': 'EQ_SETTINGS',    'current': '+0dB at 10kHz', 'proposed': '+3dB at 10kHz', 'location': 'line 87'},
    ],
    'side_effects_to_check': [
        'Harshness in the 3-5kHz range',
        'Overall level increase from boosted high frequencies',
    ],
}
```

### Step 4: Make the Minimal Edit

Edit only the identified parameters. Do not touch unrelated code. Use exact line references from Step 3.

### Step 5: Re-run the Script

Execute the modified script to generate the new `.wav` output.

### Step 6: Run Quality Validation

Run standard quality checks: LUFS measurement, frequency balance, clipping detection, stereo correlation. Flag any regressions from the previous version.

### Step 7: Present and Confirm

Play or present the new version. Ask the user if the change achieved what they wanted. If not, iterate again from Step 2.

## Parameter Location Guide

```python
PARAM_LOCATIONS = {
    # -- Volume & Mix Levels (mixing section, typically lines 60-85%) ------
    'VOLUME_LEVELS': {
        'location': 'Mixing section, look in the final mix-down area',
        'pattern':  '*_vol = <float> or gain multipliers in the mix bus',
        'example':  'kick_vol = 0.8; snare_vol = 0.7; bass_vol = 0.6',
    },
    'PAN_POSITIONS': {
        'location': 'Mixing section, near volume assignments',
        'pattern':  'pan_mono(signal, <float>) or *_pan = <float>',
        'example':  'hat_L, hat_R = pan_mono(hat, 0.38)',
    },

    # -- Filter & EQ (builder functions or effect chains) ------------------
    'FILTER_CUTOFF': {
        'location': 'In builder functions or per-track processing',
        'pattern':  'lowpass(signal, <freq>) or highpass(signal, <freq>) or butter(signal, <freq>)',
        'example':  'pad = lowpass(pad, 6000, SR)',
    },
    'EQ_SETTINGS': {
        'location': 'Per-track processing or master chain',
        'pattern':  'eq_shelf(...), eq_peak(...), eq_band(...)',
        'example':  'lead = eq_peak(lead, 3000, gain_db=2.0, q=1.5, sr=SR)',
    },

    # -- Reverb & Delay (effect chains) ------------------------------------
    'REVERB': {
        'location': 'Per-track or send bus, in effect chain section',
        'pattern':  'freeverb(signal, ...) or reverb_decay = <float> or reverb_wet = <float>',
        'example':  'pad = freeverb(pad, decay=0.7, wet=0.3, sr=SR)',
    },
    'DELAY': {
        'location': 'Per-track or send bus, in effect chain section',
        'pattern':  'delay(signal, ...) or delay_time = <float> or delay_wet = <float>',
        'example':  'lead = delay(lead, time_ms=375, feedback=0.3, wet=0.2)',
    },

    # -- Dynamics (compression, sidechain) ---------------------------------
    'COMPRESSION': {
        'location': 'Per-track processing or mix bus',
        'pattern':  'compressor(signal, ...) or compress(signal, ...) with ratio, threshold',
        'example':  'drums = compressor(drums, threshold_db=-18, ratio=4.0, attack_ms=5, release_ms=80)',
    },
    'SIDECHAIN': {
        'location': 'Mixing section, applied to pads/bass',
        'pattern':  'sidechain(signal, trigger, ...)',
        'example':  'pad = sidechain(pad, kick, threshold_db=-20, ratio=10.0, attack_ms=1, release_ms=150)',
    },

    # -- Synthesis Parameters (builder functions) --------------------------
    'WAVEFORM': {
        'location': 'Inside instrument builder functions',
        'pattern':  "osc_type='<type>' or waveform='<type>' -- values: sine, saw, square, triangle, pulse",
        'example':  "bass = oscillator(freq, waveform='saw', sr=SR)",
    },
    'ENVELOPE': {
        'location': 'Inside instrument builder functions',
        'pattern':  'adsr(signal, attack, decay, sustain, release)',
        'example':  'note = adsr(note, attack=0.01, decay=0.1, sustain=0.7, release=0.2)',
    },
    'DISTORTION': {
        'location': 'Effect chain, per-track or bus',
        'pattern':  'saturate(signal, drive) or distort(signal, drive) or waveshape(signal, ...)',
        'example':  'bass = saturate(bass, drive=1.5)',
    },
}
```

## Version Management

Track iterations systematically so the user can compare and revert.

### Naming Convention

```python
# File naming for iterations:
# {track_name}_v{N}.py   -- script for version N
# {track_name}_v{N}.wav  -- rendered audio for version N
#
# Example:
#   chill_beat_v1.py   / chill_beat_v1.wav   -- original generation
#   chill_beat_v2.py   / chill_beat_v2.wav   -- "make it darker"
#   chill_beat_v3.py   / chill_beat_v3.wav   -- "more reverb on the lead"
```

### Workflow

1. Before modifying: copy the current script to `{name}_v{N}.py` as a backup.
2. Edit the working copy and render to `{name}_v{N+1}.wav`.
3. Save the modified script as `{name}_v{N+1}.py`.
4. Present both versions so the user can A/B compare.
5. If the user wants to revert: load the previous version's `.py` file and re-render.

### Change Log

Maintain a brief inline comment at the top of each versioned script documenting what changed:

```python
# Version: 3
# Based on: chill_beat_v2.py
# Changes: made it darker -- lowered cutoff 8000->5200, increased reverb wet 0.25->0.38,
#          cut high shelf -3dB at 8kHz
```
