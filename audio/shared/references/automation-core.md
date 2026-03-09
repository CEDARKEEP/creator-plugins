# Automation & Modulation Core

Within-section parameter automation: filter sweeps, LFO systems, modulation depth scaling. Shared across all audio generation skills.

## The Three Modulation Levels

```python
MODULATION_LEVELS = {
    'section': {
        'description': 'Energy map - changes at segment boundaries',
        'timescale': 'Entire segment (5-60 seconds)',
        'controlled_by': 'Segment energy definitions in the energy framework',
        'example': 'One segment at intensity=5 transitions to the next at intensity=9',
        'note': 'Already covered by the energy framework. Do NOT duplicate section-level changes here.',
    },
    'phrase': {
        'description': 'Slow sweeps within a segment - filter opens, pan drifts, volume swells',
        'timescale': '2-30 seconds',
        'controlled_by': 'FILTER_SWEEP_TEMPLATES and phrase-level automation plans',
        'example': 'Filter cutoff rises from 400Hz to 12kHz over 10 seconds during a build',
        'note': 'This is where most "movement" lives. Every segment needs at least one phrase-level sweep.',
    },
    'micro': {
        'description': 'LFO-rate modulation - amplitude, filter, pan, pitch modulation',
        'timescale': 'Sub-second to a few seconds (0.05Hz - 30Hz)',
        'controlled_by': 'LFO_ASSIGNMENTS and generate_lfo()',
        'example': 'Sustained sound filter cutoff oscillates at 0.2Hz with depth 0.3 for gentle breathing',
        'note': 'Makes sustained sounds feel organic. Without micro-level LFOs, sustained tones sound frozen.',
    },
}
```

## Filter Sweep Templates

```python
FILTER_SWEEP_TEMPLATES = {
    'open_build': {
        'description': 'Filter opens gradually (classic build effect)',
        'cutoff_start': 400,
        'cutoff_end': 12000,
        'curve_type': 'exponential',
        'duration_seconds': 10,
        'typical_use': 'Pre-peak builds, opening reveals, post-quiet rebuilds',
    },
    'close_intimate': {
        'description': 'Filter closes for subdued, intimate feel',
        'cutoff_start': 8000,
        'cutoff_end': 1200,
        'curve_type': 'linear',
        'duration_seconds': 5,
        'typical_use': 'Transition into quieter segments, wind-down, fade-out',
    },
    'breathe': {
        'description': 'Open and close cyclically, creates breathing feel',
        'cutoff_start': 2000,
        'cutoff_end': 8000,
        'curve_type': 'sine_cycle',
        'duration_seconds': 10,
        'typical_use': 'Sustained sounds, ambient backgrounds, drones',
    },
    'wobble': {
        'description': 'Rapid open/close at rhythmic rate',
        'cutoff_start': 300,
        'cutoff_end': 6000,
        'curve_type': 'triangle_lfo',
        'duration_seconds': 5,
        'typical_use': 'Aggressive bass sounds, wobble effects, intense segments',
    },
    'resonance_peak': {
        'description': 'Resonance sweeps up while cutoff stays fixed',
        'cutoff_start': 1500,
        'cutoff_end': 1500,
        'resonance_start': 0.3,
        'resonance_end': 0.9,
        'curve_type': 'linear',
        'duration_seconds': 10,
        'typical_use': 'Resonant sweeps, squelchy textures, sequenced patterns',
    },
    'sidechain_filter': {
        'description': 'Cutoff follows sidechain envelope (ducks filter on trigger)',
        'cutoff_start': 6000,
        'cutoff_end': 800,
        'curve_type': 'sidechain_env',
        'duration_seconds': 2,
        'typical_use': 'Rhythmic pumping on sustained sounds, trigger-driven filtering',
    },
    'spectral_tilt': {
        'description': 'Gradually shift spectral balance from dark to bright across segment',
        'cutoff_start': 1500,
        'cutoff_end': 14000,
        'curve_type': 'exponential',
        'duration_seconds': 20,
        'typical_use': 'Full-segment transitions, slow builds, cinematic swells',
    },
    'lo_fi_drift': {
        'description': 'Slow random walk on cutoff (tape degradation feel)',
        'cutoff_start': 3000,
        'cutoff_end': 5000,
        'curve_type': 'random_walk',
        'duration_seconds': 20,
        'typical_use': 'Lo-fi textures, tape-style degradation, vintage ambience',
    },
    'notch_sweep': {
        'description': 'Notch filter sweeps through spectrum (phaser-like)',
        'cutoff_start': 400,
        'cutoff_end': 6000,
        'curve_type': 'linear',
        'duration_seconds': 10,
        'typical_use': 'Phaser-like movement on sustained sounds, retro textures',
    },
    'bandpass_narrow': {
        'description': 'Narrow bandpass opens to wideband (telephone to full)',
        'cutoff_start': 1000,
        'cutoff_end': 16000,
        'bandwidth_start': 0.5,
        'bandwidth_end': 4.0,
        'curve_type': 'exponential',
        'duration_seconds': 5,
        'typical_use': 'Opening reveals, processing transitions, radio effects',
    },
}
```

## LFO Assignment Table

```python
LFO_ASSIGNMENTS = {
    'sustained_filter': {
        'rate_hz': (0.1, 0.3),
        'depth': (0.2, 0.4),
        'shape': 'sine',
        'target': 'filter_cutoff',
        'notes': 'Gentle breathing on sustained sounds. Essential for any held tone or drone.',
    },
    'sustained_volume': {
        'rate_hz': (0.05, 0.15),
        'depth': (0.1, 0.2),
        'shape': 'sine',
        'target': 'amplitude',
        'notes': 'Slow volume swell on sustained sounds. Combine with sustained_filter for organic movement.',
    },
    'pitch_vibrato': {
        'rate_hz': (4.0, 6.0),
        'depth': (0.015, 0.03),
        'shape': 'sine',
        'target': 'pitch',
        'notes': 'Depth in semitones. Apply after sound onset (delay 100-200ms for natural feel).',
    },
    'filter_wah': {
        'rate_hz': (1.0, 3.0),
        'depth': (0.2, 0.4),
        'shape': 'triangle',
        'target': 'filter_cutoff',
        'notes': 'Wah-like filter movement. Triangle avoids the click of square.',
    },
    'filter_wobble': {
        'rate_hz': (0.5, 2.0),
        'depth': (0.3, 0.5),
        'shape': 'sine',
        'target': 'filter_cutoff',
        'notes': 'Aggressive filter wobble on low-frequency content. Use selectively — too intense for most contexts.',
    },
    'high_element_pan': {
        'rate_hz': (0.25, 0.5),
        'depth': (0.3, 0.5),
        'shape': 'sine',
        'target': 'pan',
        'notes': 'Subtle stereo movement on high-frequency elements. Keep depth moderate to avoid distraction.',
    },
    'tremolo': {
        'rate_hz': (2.0, 4.0),
        'depth': (0.15, 0.3),
        'shape': 'sine',
        'target': 'amplitude',
        'notes': 'Classic tremolo effect. Works on any sustained sound.',
    },
    'auto_pan': {
        'rate_hz': (0.1, 0.25),
        'depth': (0.3, 0.6),
        'shape': 'sine',
        'target': 'pan',
        'notes': 'Slow stereo sweep. Effective on repeating patterns, delays, and textural elements.',
    },
    'ring_mod': {
        'rate_hz': (8.0, 30.0),
        'depth': (0.1, 0.8),
        'shape': 'sine',
        'target': 'amplitude',
        'notes': 'Metallic texture. Depth varies hugely by context. Use sparingly.',
    },
    'resonance_mod': {
        'rate_hz': (0.3, 1.0),
        'depth': (0.2, 0.4),
        'shape': 'sine',
        'target': 'filter_q',
        'notes': 'Modulate filter resonance for squelchy, liquid movement. Pair with filter LFO.',
    },
    'stereo_width_mod': {
        'rate_hz': (0.08, 0.2),
        'depth': (0.1, 0.3),
        'shape': 'sine',
        'target': 'stereo_width',
        'notes': 'Very slow width breathing. Subtle but adds life to stereo field.',
    },
}
```

## Implementation: generate_lfo()

```python
def generate_lfo(t, rate, shape, phase_offset=0.0):
    """Generate an LFO waveform over time array t.

    Args:
        t: numpy array of time values in seconds
        rate: frequency in Hz
        shape: 'sine', 'triangle', 'square', or 'sample_hold'
        phase_offset: starting phase in radians (0 to 2*pi)

    Returns:
        numpy array of values in range [-1, 1]
    """
    phase = 2 * np.pi * rate * t + phase_offset

    if shape == 'sine':
        return np.sin(phase)

    elif shape == 'triangle':
        # Triangle from phase: 2*|sawtooth| - 1
        saw = (phase / (2 * np.pi)) % 1.0
        return 4.0 * np.abs(saw - 0.5) - 1.0

    elif shape == 'square':
        return np.sign(np.sin(phase))

    elif shape == 'sample_hold':
        # New random value each LFO cycle
        rng = np.random.RandomState(42)
        cycle_index = (phase / (2 * np.pi)).astype(int)
        values = np.zeros_like(t)
        unique_cycles = np.unique(cycle_index)
        random_vals = rng.uniform(-1, 1, size=len(unique_cycles))
        for i, c in enumerate(unique_cycles):
            values[cycle_index == c] = random_vals[i]
        return values

    else:
        raise ValueError(f"Unknown LFO shape: {shape}. Use sine, triangle, square, or sample_hold.")
```

## Implementation: apply_bar_automation()

```python
def apply_segment_automation(signal, segment_index, total_segments, automation_plan, sr=44100, segment_duration=2.0):
    """Apply within-section automation to a signal for one segment.

    Args:
        signal: numpy array, mono audio for this segment
        segment_index: which segment within the section (0-based)
        total_segments: total segments in this section
        automation_plan: dict from build_automation_plan()
        sr: sample rate
        segment_duration: duration of each segment in seconds

    Returns:
        numpy array (same length as signal), automated signal
        dict of metadata (pan offsets, etc.) for downstream mixing
    """
    from scipy.signal import butter, sosfilt

    progress = segment_index / max(total_segments - 1, 1)  # 0.0 to 1.0 across section
    t = np.arange(len(signal)) / sr  # time array for this bar
    metadata = {}

    # --- Filter sweep (bar-level) ---
    if 'filter_sweep' in automation_plan:
        sweep = automation_plan['filter_sweep']
        f_start = sweep['cutoff_start']
        f_end = sweep.get('cutoff_end', f_start)
        curve = sweep.get('curve_type', 'linear')

        if curve == 'exponential':
            # Exponential interpolation in log space
            cutoff = f_start * (f_end / max(f_start, 1)) ** progress
        elif curve == 'linear':
            cutoff = f_start + (f_end - f_start) * progress
        elif curve == 'sine_cycle':
            # Full sine cycle over the section for breathing effect
            cutoff = (f_start + f_end) / 2 + (f_end - f_start) / 2 * np.sin(2 * np.pi * progress)
            cutoff = float(cutoff)
        elif curve == 'random_walk':
            # Deterministic random walk seeded by bar_index
            rng_local = np.random.RandomState(bar_index + 7)
            walk_step = rng_local.uniform(-0.15, 0.15)
            center = (f_start + f_end) / 2
            span = (f_end - f_start) / 2
            cutoff = center + span * np.clip(walk_step * (bar_index + 1), -1, 1)
        else:
            cutoff = f_start + (f_end - f_start) * progress

        # Clamp cutoff to safe range
        cutoff = np.clip(cutoff, 20, sr / 2 - 100)

        # Apply lowpass filter
        sos = butter(2, float(cutoff), btype='low', fs=sr, output='sos')
        signal = sosfilt(sos, signal).astype(signal.dtype)
        metadata['filter_cutoff'] = float(cutoff)

        # Handle resonance sweep if present
        if 'resonance_start' in sweep and 'resonance_end' in sweep:
            q_start = sweep['resonance_start']
            q_end = sweep['resonance_end']
            q = q_start + (q_end - q_start) * progress
            # Apply resonant peak with bandpass
            bw = max(50, float(cutoff) * (1.0 - q))
            f_low = max(20, float(cutoff) - bw / 2)
            f_high = min(sr / 2 - 100, float(cutoff) + bw / 2)
            if f_low < f_high:
                sos_bp = butter(1, [f_low, f_high], btype='band', fs=sr, output='sos')
                resonance_signal = sosfilt(sos_bp, signal)
                signal = signal + resonance_signal * q * 0.5
            metadata['filter_q'] = float(q)

    # --- LFO modulation (beat-level) ---
    pan_offset = np.zeros(len(signal))
    width_offset = np.zeros(len(signal))

    if 'lfo' in automation_plan:
        for lfo_cfg in automation_plan['lfo']:
            rate = lfo_cfg['rate']
            depth = lfo_cfg['depth']
            shape = lfo_cfg.get('shape', 'sine')
            target = lfo_cfg['target']
            phase = lfo_cfg.get('phase_offset', 0.0)

            mod = depth * generate_lfo(t, rate, shape, phase)

            if target == 'amplitude':
                # Modulate volume: signal * (1 + mod), clamp to avoid negative
                signal = signal * np.clip(1.0 + mod, 0.0, 2.0)

            elif target == 'filter_cutoff':
                # Modulate filter cutoff per-sample (use time-varying filter approx)
                # For efficiency: apply at bar level using average modulation
                avg_mod = float(np.mean(np.abs(mod)))
                base_cutoff = metadata.get('filter_cutoff', 4000)
                mod_cutoff = base_cutoff * (1.0 + avg_mod)
                mod_cutoff = np.clip(mod_cutoff, 20, sr / 2 - 100)
                sos_lfo = butter(2, float(mod_cutoff), btype='low', fs=sr, output='sos')
                signal = sosfilt(sos_lfo, signal).astype(signal.dtype)

            elif target == 'pan':
                pan_offset += mod

            elif target == 'pitch':
                # Pitch modulation via resampling (vibrato)
                # depth is in semitones; convert to ratio
                pitch_mod = 2.0 ** (mod * depth / 12.0) if depth > 0 else np.ones_like(mod)
                # Phase accumulator for variable-rate playback
                phase_acc = np.cumsum(pitch_mod)
                phase_acc = phase_acc / phase_acc[-1] * (len(signal) - 1)
                indices = np.clip(phase_acc, 0, len(signal) - 1)
                signal = np.interp(np.arange(len(signal)),
                                   indices, signal)

            elif target == 'filter_q':
                metadata['filter_q_mod'] = mod

            elif target == 'stereo_width':
                width_offset += mod

    if np.any(pan_offset != 0):
        metadata['pan_offset'] = pan_offset
    if np.any(width_offset != 0):
        metadata['width_offset'] = width_offset

    return signal, metadata
```

## Modulation Depth by Energy Level

How modulation parameters should scale with the section energy map:

```python
MODULATION_DEPTH_SCALING = {
    'low': {
        'energy_range': (1, 3),
        'lfo_rate_scale': 0.6,
        'lfo_depth_scale': 0.5,
        'filter_sweep_range_scale': 0.4,
        'description': 'Subtle, slow modulation. Rates below 1Hz for most targets. '
                       'Small filter sweep ranges (1-2kHz). Depth kept minimal so '
                       'the listener feels gentle movement without obvious wobble.',
        'guidelines': [
            'Sustained filter LFO: rate 0.05-0.15Hz, depth 0.1-0.2',
            'No aggressive wobble modulation',
            'Volume LFOs barely perceptible (depth < 0.1)',
            'Auto-pan very slow (0.05-0.1Hz) and narrow (depth 0.15-0.25)',
            'Filter sweeps: narrow range, linear curves preferred',
        ],
    },
    'medium': {
        'energy_range': (4, 6),
        'lfo_rate_scale': 1.0,
        'lfo_depth_scale': 1.0,
        'filter_sweep_range_scale': 1.0,
        'description': 'Moderate modulation at default rates and depths. '
                       'This is the baseline — LFO_ASSIGNMENTS values are calibrated for this range.',
        'guidelines': [
            'Use LFO_ASSIGNMENTS values directly (no scaling needed)',
            'Filter sweeps can cover 2-6kHz range',
            'Pitch vibrato at natural rates (4-6Hz)',
            'Auto-pan and stereo width mod at normal depth',
            'Can introduce rhythmic elements (tremolo, gating)',
        ],
    },
    'high': {
        'energy_range': (7, 9),
        'lfo_rate_scale': 1.3,
        'lfo_depth_scale': 1.4,
        'filter_sweep_range_scale': 1.6,
        'description': 'Aggressive, energetic modulation. Faster rates, deeper depths. '
                       'Filter sweeps can cover full spectrum. Wobble and gating effects welcome.',
        'guidelines': [
            'Filter LFOs can go faster (1-4Hz) and deeper (0.3-0.6)',
            'Filter sweeps: wide range (400Hz-12kHz+), exponential curves',
            'Tremolo and gating at synced rates',
            'Filter wobble active where appropriate',
            'Multiple simultaneous LFOs on the same sound OK',
            'Auto-pan can be wider (depth 0.4-0.7)',
        ],
    },
    'peak': {
        'energy_range': (10, 10),
        'lfo_rate_scale': 1.0,
        'lfo_depth_scale': 1.0,
        'filter_sweep_range_scale': 1.0,
        'description': 'Context-dependent. Either maximum modulation (heavy bass drops, wobble effects) '
                       'or strip back to raw power (distorted, aggressive, hard-hitting). '
                       'The decision depends on the target style and use case.',
        'guidelines': [
            'Heavy/aggressive contexts: max filter movement, synced gates, full-range sweeps',
            'Raw/powerful contexts: LESS modulation — raw power from gain and saturation',
            'Build-oriented contexts: maximum gate depth, widest filter sweep, longest risers resolve here',
            'Transparent contexts: biggest climax should feel natural, not over-processed',
            'Spatial contexts: max spatial width and tonal density',
            'Consider removing LFOs and using static maximum values for raw impact',
        ],
    },
}
```
