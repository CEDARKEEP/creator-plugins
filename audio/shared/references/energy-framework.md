# Energy & Dynamics Framework

Multi-dimensional energy system for controlling audio dynamics, tension/release cycles, and engagement over time. Shared across all audio generation skills.

## The Energy Map: Blueprint Before Building

Always create an energy map BEFORE writing any audio builders. The energy map is the single most important planning step -- it determines how the output *feels* over time and whether listeners stay engaged or tune out.

### Multi-Dimensional Energy (Not Just Volume)

Energy is NOT a single number. Every segment has 5 dimensions that the script must track:

```python
ENERGY_DIMENSIONS = {
    'intensity':  'Overall power level (0-10). Controls volume, compression, distortion',
    'density':    'Number of active layers (0-10). Sparse=1-2 elements, Full=8+ layers',
    'activity':   'Temporal complexity (0-10). Simple=single pulse, Complex=layered events+variation',
    'richness':   'Tonal/textural complexity (0-10). Simple=basic tones, Rich=complex layering+harmonics',
    'brightness': 'Spectral energy (0-10). Dark=lowpassed, Bright=full spectrum+air+presence',
}

# NOTE: These five dimensions can be customized per domain.
# Music skills might rename 'activity' to 'rhythm' and 'richness' to 'harmonic'.
# Soundscape skills might rename 'activity' to 'event_density'.
# The framework functions below operate on any dict of named 0-10 dimensions.

# Each segment gets all 5 dimensions (generic template):
SEGMENT_ENERGY = {
    'opening':     {'intensity': 3, 'density': 2, 'activity': 2, 'richness': 4, 'brightness': 3},
    'development': {'intensity': 5, 'density': 5, 'activity': 5, 'richness': 5, 'brightness': 5},
    'build':       {'intensity': 6, 'density': 6, 'activity': 6, 'richness': 6, 'brightness': 7},
    'peak':        {'intensity': 9, 'density': 8, 'activity': 7, 'richness': 8, 'brightness': 9},
    'rest':        {'intensity': 3, 'density': 2, 'activity': 2, 'richness': 5, 'brightness': 3},
    'climax':      {'intensity': 10,'density': 9, 'activity': 9, 'richness': 6, 'brightness': 10},
    'resolution':  {'intensity': 3, 'density': 2, 'activity': 2, 'richness': 4, 'brightness': 2},
}
```

### How Each Dimension Controls the Mix

```python
def apply_energy_map(bar, section_energy, track_name, signal, sr=SR):
    """Apply multi-dimensional energy to a track for a given bar."""
    e = section_energy  # dict with intensity, density, rhythm, harmonic, brightness

    # 1. INTENSITY -> volume envelope + compression drive
    volume = 0.3 + 0.7 * (e['intensity'] / 10.0)  # never fully silent

    # 2. DENSITY -> element on/off gating
    #    Handled by builder: only activate track if density >= track's threshold
    #    e.g., 'strings' only active when density >= 7

    # 3. ACTIVITY -> temporal complexity selection
    #    Handled by pattern/event builder: select complexity based on activity level
    #    activity < 4: basic pattern, few events
    #    activity 4-6: moderate complexity
    #    activity 7-8: complex patterns, varied events
    #    activity 9-10: dense layered patterns, rapid variation

    # 4. RICHNESS -> tonal/textural complexity
    #    richness < 4: simple tones, minimal layering
    #    richness 4-6: moderate layering, added harmonics
    #    richness 7-8: rich textures, complex layering
    #    richness 9-10: full spectral density, maximum complexity

    # 5. BRIGHTNESS -> filter cutoff + presence EQ
    cutoff = 800 + (e['brightness'] / 10.0) * 12000  # 800Hz to 12.8kHz
    signal = lowpass(signal, cutoff, sr)
    if e['brightness'] >= 7:
        # Add air shelf boost at 10kHz
        signal = eq_shelf(signal, 10000, gain_db=2.0, sr=sr)

    return signal * volume
```

## Tension & Release: The Engine of Engagement

Tension creates anticipation. Release delivers satisfaction. Without this cycle, audio is wallpaper.

### Macro-Tension (Segment Level: 5-60 seconds)

Large-scale builds that create dramatic arcs across segments:

```python
MACRO_TENSION_TECHNIQUES = {
    'riser': {
        'description': 'Noise or pitch sweep building over 5-30 seconds',
        'energy_effect': 'intensity +2 to +4 over duration',
        'implementation': 'White noise HPF sweep 200->8000Hz, rising volume',
        'placement': 'Pre-peak, build segments',
    },
    'filter_open': {
        'description': 'LPF sweeping open from 200Hz to full spectrum',
        'energy_effect': 'brightness +6 over duration',
        'implementation': 'Exponential cutoff sweep on main bus or individual tracks',
        'placement': 'Opening, build segments, post-rest rebuilds',
    },
    'dynamic_swell': {
        'description': 'Gradual volume increase across all elements',
        'energy_effect': 'intensity +3 over 10-30 seconds',
        'implementation': 'Linear or exponential volume ramp, optionally with compression release tightening',
        'placement': 'Any build segment',
    },
    'pitch_riser': {
        'description': 'Sustained tone rising in pitch over 5-30 seconds',
        'energy_effect': 'intensity +2, brightness +3',
        'implementation': 'Sine or saw with exponential pitch bend up 1-2 octaves',
        'placement': 'Pre-peak builds',
    },
    'element_stacking': {
        'description': 'Add new element every 2-5 seconds',
        'energy_effect': 'density +1 per layer added',
        'implementation': 'Stagger element entries across build segment',
        'placement': 'Opening, post-rest rebuilds',
    },
}
```

### Micro-Tension (Within Segments: 0.2-5 seconds)

Small elements that maintain forward momentum and prevent listener fatigue:

```python
MICRO_TENSION_TECHNIQUES = {
    'filter_dip': {
        'description': 'Quick LPF dip and recovery (200ms-2s)',
        'effect': 'Breathing moment in the mix',
        'frequency': 'Periodically on sustained elements',
    },
    'stereo_moment': {
        'description': 'Brief mono->wide or wide->narrow shift',
        'effect': 'Spatial surprise that re-engages attention',
        'frequency': 'At segment transitions',
    },
    'glitch_stutter': {
        'description': 'Buffer repeat or slice at phrase boundaries',
        'effect': 'Breaks predictability, adds excitement',
        'frequency': 'Sparingly',
    },
}
```

### The Pull/Push Framework

Every tension technique is either a pull (building anticipation) or a push (delivering release). Balance is key -- if the pull overpowers the push, the climax disappoints. If there's no pull, the push has no impact.

```python
PULL_PUSH_PAIRS = {
    # Pull (tension)              ->  Push (release)
    'riser':                        'impact + drop',
    'filter_closing':               'filter_opening',
    'pitch_bend_up':                'pitch_drop + low-end entry',
    'silence_gap':                  'full_mix_slam',
    'volume_dip':                   'dynamic_swell',
    'mono_collapse':                'stereo_expansion',
    'activity_reduction':           'full_activity_return',
}

# Rule: pull duration should roughly match push intensity
# Short pull (1-3 seconds) -> moderate push
# Long pull (10-30 seconds) -> massive push (climax, peak)
# No pull -> no push impact (avoid!)
```

## Energy Interpolation: Smooth Transitions

Never jump between energy levels -- always interpolate. The script should compute per-sample energy curves:

```python
def build_energy_curve(sections, total_samples, segment_samples, sr=SR):
    """Build a smooth per-sample energy curve from section definitions.
    Returns dict of numpy arrays, one per dimension."""
    curves = {dim: np.zeros(total_samples) for dim in ENERGY_DIMENSIONS}
    transition_segments = 2  # Segments to crossfade between sections

    seg = 0
    for i, section in enumerate(sections):
        for b in range(section['segments']):
            start = int(seg * segment_samples)
            end = int((seg + 1) * segment_samples)
            if end > total_samples:
                end = total_samples

            for dim in ENERGY_DIMENSIONS:
                target = section['energy'][dim] / 10.0

                # Check if we're in a transition zone
                if b < transition_segments and i > 0:
                    prev = sections[i-1]['energy'][dim] / 10.0
                    t = b / transition_segments
                    value = prev + (target - prev) * smoothstep(t)
                elif b >= section['segments'] - transition_segments and i < len(sections) - 1:
                    next_val = sections[i+1]['energy'][dim] / 10.0
                    t = (b - (section['segments'] - transition_segments)) / transition_segments
                    value = target + (next_val - target) * smoothstep(t)
                else:
                    value = target

                curves[dim][start:end] = value

            seg += 1

    return curves

def smoothstep(t):
    """Hermite smoothstep interpolation (0->1)."""
    t = np.clip(t, 0, 1)
    return t * t * (3 - 2 * t)
```

## Per-Segment Energy Automation Effects

Apply energy curves to control these parameters per segment:

```python
ENERGY_AUTOMATION = {
    'intensity': {
        'volume':           lambda e: 0.3 + 0.7 * e,          # 0.3 to 1.0
        'compressor_ratio': lambda e: 1.5 + 2.5 * e,          # 1.5:1 to 4:1
        'saturation_drive': lambda e: 1.0 + 0.5 * e,          # 1.0x to 1.5x
    },
    'density': {
        'active_tracks':    lambda e: int(2 + 8 * e),          # 2 to 10 tracks
        'reverb_wet':       lambda e: 0.15 + 0.2 * e,         # 0.15 to 0.35
    },
    'brightness': {
        'lpf_cutoff':       lambda e: 800 + 12000 * e,        # 800Hz to 12.8kHz
        'hpf_cutoff':       lambda e: 30 + 100 * (1 - e),     # Less HPF when bright
        'air_shelf_gain':   lambda e: max(0, (e - 0.6) * 5),  # 0 to 2dB above brightness 0.6
    },
}
```

## Energy Map Visualization (Console Output)

Print a visual energy map when the script runs so the user can see the arc:

```python
def print_energy_map(sections):
    """Print a visual ASCII energy map to console."""
    print("\nENERGY MAP")
    print("=" * 70)

    max_bar_width = 40
    for section in sections:
        intensity = section['energy']['intensity']
        density = section['energy']['density']
        bar_fill = "#" * int(intensity * max_bar_width / 10)
        density_dots = "." * int(density * max_bar_width / 10)

        name = f"{section['name']:.<20}"
        segs = f"({section['segments']}seg)"
        print(f"  {name} {segs:>8}  I:{bar_fill:<{max_bar_width}}")
        print(f"  {'':20} {'':8}  D:{density_dots:<{max_bar_width}}")

    print("=" * 70)
    print("  I=Intensity  D=Density")
    print()
```
