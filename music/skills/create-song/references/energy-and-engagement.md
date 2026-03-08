# Energy Mapping & Listener Engagement

Multi-dimensional energy system, tension/release cycles, emotional arcs, and techniques for creating music that grabs and holds attention. Inspired by professional music production, composition planning, and film scoring techniques.

## The Energy Map: Blueprint Before Building

**Always create an energy map BEFORE writing any instrument builders.** The energy map is the single most important planning step — it determines how the track *feels* over time and whether listeners stay engaged or tune out.

### Multi-Dimensional Energy (Not Just Volume)

Energy is NOT a single number. Every bar has **5 dimensions** that the script must track:

```python
ENERGY_DIMENSIONS = {
    'intensity':  'Overall power level (0-10). Controls volume, compression, distortion',
    'density':    'Number of active layers (0-10). Sparse=1-2 instruments, Full=8+ layers',
    'rhythm':     'Rhythmic complexity (0-10). Simple=kick only, Complex=syncopation+fills+rolls',
    'harmonic':   'Harmonic richness (0-10). Simple=root+5th, Rich=7ths+9ths+extensions+inversions',
    'brightness': 'Spectral energy (0-10). Dark=lowpassed, Bright=full spectrum+air+presence',
}

# Each section gets all 5 dimensions:
SECTION_ENERGY = {
    'intro':       {'intensity': 3, 'density': 2, 'rhythm': 2, 'harmonic': 4, 'brightness': 3},
    'verse':       {'intensity': 5, 'density': 5, 'rhythm': 5, 'harmonic': 5, 'brightness': 5},
    'pre_chorus':  {'intensity': 6, 'density': 6, 'rhythm': 6, 'harmonic': 6, 'brightness': 7},
    'chorus':      {'intensity': 9, 'density': 8, 'rhythm': 7, 'harmonic': 8, 'brightness': 9},
    'bridge':      {'intensity': 5, 'density': 4, 'rhythm': 4, 'harmonic': 7, 'brightness': 5},
    'drop':        {'intensity': 10,'density': 9, 'rhythm': 9, 'harmonic': 6, 'brightness': 10},
    'breakdown':   {'intensity': 3, 'density': 2, 'rhythm': 2, 'harmonic': 5, 'brightness': 3},
    'outro':       {'intensity': 3, 'density': 2, 'rhythm': 2, 'harmonic': 4, 'brightness': 2},
}
```

### How Each Dimension Controls the Mix

```python
def apply_energy_map(bar, section_energy, track_name, signal, sr=SR):
    """Apply multi-dimensional energy to a track for a given bar."""
    e = section_energy  # dict with intensity, density, rhythm, harmonic, brightness

    # 1. INTENSITY → volume envelope + compression drive
    volume = 0.3 + 0.7 * (e['intensity'] / 10.0)  # never fully silent

    # 2. DENSITY → element on/off gating
    #    Handled by builder: only activate track if density >= track's threshold
    #    e.g., 'strings' only active when density >= 7

    # 3. RHYTHM → pattern complexity selection
    #    Handled by drum builder: select pattern variant based on rhythm level
    #    rhythm < 4: basic pattern (kick+hat)
    #    rhythm 4-6: full pattern (kick+snare+hat+ride)
    #    rhythm 7-8: syncopated pattern + ghost notes
    #    rhythm 9-10: fills, rolls, polyrhythmic layers

    # 4. HARMONIC → chord voicing richness
    #    harmonic < 4: triads, root position
    #    harmonic 4-6: 7th chords, inversions
    #    harmonic 7-8: 9ths, 11ths, spread voicings
    #    harmonic 9-10: extensions, altered chords, chromatic passing

    # 5. BRIGHTNESS → filter cutoff + presence EQ
    cutoff = 800 + (e['brightness'] / 10.0) * 12000  # 800Hz to 12.8kHz
    signal = lowpass(signal, cutoff, sr)
    if e['brightness'] >= 7:
        # Add air shelf boost at 10kHz
        signal = eq_shelf(signal, 10000, gain_db=2.0, sr=sr)

    return signal * volume
```

### Track Density Thresholds

Each instrument has a minimum density level to be active:

```python
TRACK_DENSITY_THRESHOLD = {
    # Always active (density 1+)
    'kick':        1,
    'bass':        1,
    'pad':         1,

    # Medium density (3+)
    'hihat':       3,
    'snare':       3,
    'chord':       3,

    # Higher density (5+)
    'melody':      5,
    'arp':         5,
    'ride':        5,

    # Full density (7+)
    'strings':     7,
    'lead':        7,
    'counter_mel': 7,
    'perc':        7,

    # Peak only (9+)
    'fx':          9,
    'doubled':     9,
    'harmony':     9,
}

def is_track_active(track_name, bar, section_energy):
    """Check if a track should play in this bar based on density."""
    threshold = TRACK_DENSITY_THRESHOLD.get(track_name, 5)
    return section_energy['density'] >= threshold
```

## Tension & Release: The Engine of Engagement

Tension creates anticipation. Release delivers satisfaction. Without this cycle, music is wallpaper.

### Macro-Tension (Section Level: 4-32 bars)

Large-scale builds that create dramatic arcs across sections:

```python
MACRO_TENSION_TECHNIQUES = {
    'riser': {
        'description': 'Noise or pitch sweep building over 4-16 bars',
        'energy_effect': 'intensity +2 to +4 over duration',
        'implementation': 'White noise HPF sweep 200→8000Hz, rising volume',
        'placement': 'Pre-chorus, pre-drop, build sections',
    },
    'snare_roll': {
        'description': 'Accelerating snare hits from 8th→16th→32nd notes',
        'energy_effect': 'rhythm +3, intensity +2',
        'implementation': 'Exponential density increase over 4-8 bars',
        'placement': 'Last 4-8 bars before drop/chorus',
    },
    'filter_open': {
        'description': 'LPF sweeping open from 200Hz to full spectrum',
        'energy_effect': 'brightness +6 over duration',
        'implementation': 'Exponential cutoff sweep on main bus or individual tracks',
        'placement': 'Intro, build sections, post-breakdown',
    },
    'harmonic_build': {
        'description': 'Progressively add chord extensions and layers',
        'energy_effect': 'harmonic +3, density +2',
        'implementation': 'Bar 1: triads → Bar 4: 7ths → Bar 8: 9ths + inversions',
        'placement': 'Verse→chorus transitions',
    },
    'dynamic_swell': {
        'description': 'Gradual volume increase across all elements',
        'energy_effect': 'intensity +3 over 8-16 bars',
        'implementation': 'Linear or exponential volume ramp, optionally with compression release tightening',
        'placement': 'Any build section',
    },
    'pitch_riser': {
        'description': 'Sustained tone rising in pitch over 4-16 bars',
        'energy_effect': 'intensity +2, brightness +3',
        'implementation': 'Sine or saw with exponential pitch bend up 1-2 octaves',
        'placement': 'Pre-drop, pre-chorus builds',
    },
    'element_stacking': {
        'description': 'Add new instrument every 2-4 bars',
        'energy_effect': 'density +1 per layer added',
        'implementation': 'Stagger instrument entries across build section',
        'placement': 'Intro, post-breakdown rebuilds',
    },
    'rhythmic_acceleration': {
        'description': 'Drum pattern subdivision increases (4th→8th→16th)',
        'energy_effect': 'rhythm +4 over build',
        'implementation': 'Hi-hat doubles, then triples, then rolls',
        'placement': 'Last 4 bars before peak sections',
    },
}
```

### Micro-Tension (Within Sections: 1-4 bars)

Small elements that maintain forward momentum and prevent listener fatigue:

```python
MICRO_TENSION_TECHNIQUES = {
    'kick_drop': {
        'description': 'Remove kick for 1-2 beats, return with impact',
        'effect': 'Creates momentary void that makes return satisfying',
        'frequency': 'Every 8-16 bars, before phrase boundaries',
    },
    'reverse_crash': {
        'description': 'Reversed cymbal crash into forward crash (pull→push)',
        'effect': 'Signals upcoming change, creates anticipation',
        'frequency': 'Before section transitions, every 8 bars',
    },
    'drum_fill': {
        'description': 'Varied fills (NOT the same fill every time)',
        'effect': 'Breaks pattern monotony, signals phrase boundary',
        'frequency': 'Every 4-8 bars, varied types',
        'variants': ['snare_roll', 'tom_cascade', 'hat_flurry', 'ghost_buildup'],
    },
    'filter_dip': {
        'description': 'Quick LPF dip and recovery (200ms-1bar)',
        'effect': 'Breathing moment in the mix',
        'frequency': 'Every 4-8 bars on pads/chords',
    },
    'glitch_stutter': {
        'description': 'Buffer repeat or beat slice on last beat of phrase',
        'effect': 'Breaks predictability, adds excitement',
        'frequency': 'Sparingly, every 8-16 bars',
    },
    'ghost_notes': {
        'description': 'Very quiet anticipatory hits before main hits',
        'effect': 'Adds groove depth and human feel',
        'frequency': 'Continuously in drum patterns at vel 20-35%',
    },
    'harmonic_surprise': {
        'description': 'Brief chromatic passing chord or borrowed chord',
        'effect': 'Ear candy, prevents harmonic predictability',
        'frequency': 'Once per 8-16 bar section, tastefully',
    },
    'stereo_moment': {
        'description': 'Brief mono→wide or wide→narrow shift',
        'effect': 'Spatial surprise that re-engages attention',
        'frequency': 'At section transitions',
    },
}
```

### The Pull/Push Framework

Every tension technique is either a **pull** (building anticipation) or a **push** (delivering release). Balance is key — if the pull overpowers the push, the drop disappoints. If there's no pull, the push has no impact.

```python
PULL_PUSH_PAIRS = {
    # Pull (tension)              →  Push (release)
    'riser':                        'impact + drop',
    'filter_closing':               'filter_opening',
    'snare_roll':                   'crash + full beat',
    'kick_removal':                 'kick_return + sub boom',
    'harmonic_suspension':          'chord_resolution',
    'pitch_bend_up':                'pitch_drop + bass entry',
    'silence_gap':                  'full_mix_slam',
    'rhythmic_simplification':      'full_groove_return',
    'mono_collapse':                'stereo_expansion',
    'volume_dip':                   'dynamic_swell',
}

# Rule: pull duration should roughly match push intensity
# Short pull (1-2 bars) → moderate push
# Long pull (8-16 bars) → massive push (drop, chorus)
# No pull → no push impact (avoid!)
```

## Emotional Arc Templates

Beyond energy levels, the **emotional journey** determines whether music connects with listeners. Based on film scoring's valence/arousal model and modern composition planning techniques.

### Emotional Dimensions

```python
EMOTIONAL_DIMENSIONS = {
    'valence':    'Positive↔Negative (happy↔sad, major↔minor)',
    'arousal':    'High↔Low energy (excited↔calm)',
    'tension':    'Tight↔Relaxed (dissonant↔consonant)',
    'movement':   'Static↔Dynamic (sustained↔rhythmic)',
    'space':      'Intimate↔Epic (dry/close↔reverb/wide)',
}
```

### Arc Templates

```python
EMOTIONAL_ARCS = {
    'hero_journey': {
        # Classic: calm → challenge → triumph → reflection
        'description': 'Start peaceful, build through struggle to victory, resolve gently',
        'arc': [
            {'section': 'intro',    'valence': 6, 'arousal': 2, 'tension': 1, 'space': 5},
            {'section': 'verse',    'valence': 5, 'arousal': 4, 'tension': 3, 'space': 5},
            {'section': 'build',    'valence': 4, 'arousal': 7, 'tension': 8, 'space': 7},
            {'section': 'chorus',   'valence': 9, 'arousal': 9, 'tension': 3, 'space': 9},
            {'section': 'verse2',   'valence': 6, 'arousal': 5, 'tension': 4, 'space': 5},
            {'section': 'chorus2',  'valence': 10,'arousal': 10,'tension': 2, 'space': 10},
            {'section': 'outro',    'valence': 8, 'arousal': 3, 'tension': 1, 'space': 7},
        ],
        'genres': ['EDM', 'trance', 'pop', 'orchestral', 'metal'],
    },

    'night_drive': {
        # Sustained mood: steady groove with subtle shifts
        'description': 'Consistent vibe with gentle undulations, hypnotic flow',
        'arc': [
            {'section': 'intro',    'valence': 5, 'arousal': 3, 'tension': 2, 'space': 6},
            {'section': 'groove',   'valence': 6, 'arousal': 6, 'tension': 3, 'space': 7},
            {'section': 'drift',    'valence': 5, 'arousal': 5, 'tension': 2, 'space': 8},
            {'section': 'peak',     'valence': 7, 'arousal': 7, 'tension': 4, 'space': 8},
            {'section': 'cruise',   'valence': 6, 'arousal': 6, 'tension': 3, 'space': 7},
            {'section': 'fade',     'valence': 5, 'arousal': 3, 'tension': 1, 'space': 9},
        ],
        'genres': ['synthwave', 'lo-fi', 'deep house', 'chillwave', 'ambient'],
    },

    'tension_release': {
        # Dramatic: build unbearable tension then explode
        'description': 'Slow suffocating build to cathartic release, repeat with higher stakes',
        'arc': [
            {'section': 'intro',      'valence': 3, 'arousal': 2, 'tension': 3, 'space': 4},
            {'section': 'build1',     'valence': 3, 'arousal': 5, 'tension': 7, 'space': 5},
            {'section': 'drop1',      'valence': 7, 'arousal': 10,'tension': 2, 'space': 9},
            {'section': 'breakdown',  'valence': 2, 'arousal': 2, 'tension': 5, 'space': 3},
            {'section': 'build2',     'valence': 2, 'arousal': 7, 'tension': 9, 'space': 6},
            {'section': 'drop2',      'valence': 8, 'arousal': 10,'tension': 1, 'space': 10},
            {'section': 'outro',      'valence': 5, 'arousal': 3, 'tension': 1, 'space': 7},
        ],
        'genres': ['dubstep', 'drum & bass', 'techno', 'trance', 'trap'],
    },

    'melancholic_beauty': {
        # Bittersweet: sadness with moments of beauty and hope
        'description': 'Wistful, contemplative, with brief moments of warmth breaking through',
        'arc': [
            {'section': 'intro',    'valence': 3, 'arousal': 2, 'tension': 2, 'space': 7},
            {'section': 'verse',    'valence': 3, 'arousal': 4, 'tension': 4, 'space': 6},
            {'section': 'chorus',   'valence': 6, 'arousal': 6, 'tension': 3, 'space': 8},
            {'section': 'verse2',   'valence': 2, 'arousal': 3, 'tension': 5, 'space': 5},
            {'section': 'chorus2',  'valence': 7, 'arousal': 7, 'tension': 2, 'space': 9},
            {'section': 'bridge',   'valence': 4, 'arousal': 5, 'tension': 6, 'space': 7},
            {'section': 'outro',    'valence': 5, 'arousal': 2, 'tension': 1, 'space': 8},
        ],
        'genres': ['indie', 'post-rock', 'ambient', 'classical', 'jazz', 'lo-fi'],
    },

    'party_energy': {
        # High energy: relentless groove with strategic breathers
        'description': 'Immediate groove, short breaks for breathing, escalating peaks',
        'arc': [
            {'section': 'intro',      'valence': 7, 'arousal': 5, 'tension': 3, 'space': 6},
            {'section': 'groove1',    'valence': 8, 'arousal': 8, 'tension': 3, 'space': 7},
            {'section': 'break1',     'valence': 6, 'arousal': 4, 'tension': 2, 'space': 5},
            {'section': 'groove2',    'valence': 9, 'arousal': 9, 'tension': 4, 'space': 8},
            {'section': 'break2',     'valence': 5, 'arousal': 3, 'tension': 5, 'space': 4},
            {'section': 'peak',       'valence': 10,'arousal': 10,'tension': 2, 'space': 9},
            {'section': 'groove3',    'valence': 9, 'arousal': 9, 'tension': 3, 'space': 8},
            {'section': 'outro',      'valence': 7, 'arousal': 5, 'tension': 1, 'space': 6},
        ],
        'genres': ['house', 'techno', 'disco', 'funk', 'dancehall', 'afrobeats'],
    },

    'cinematic_epic': {
        # Film score: slow establishment → escalating drama → massive climax → resolution
        'description': 'Grandiose emotional journey with multiple escalating peaks',
        'arc': [
            {'section': 'establish',  'valence': 5, 'arousal': 1, 'tension': 1, 'space': 8},
            {'section': 'theme',      'valence': 6, 'arousal': 4, 'tension': 3, 'space': 7},
            {'section': 'develop',    'valence': 4, 'arousal': 6, 'tension': 6, 'space': 8},
            {'section': 'conflict',   'valence': 2, 'arousal': 8, 'tension': 9, 'space': 9},
            {'section': 'climax',     'valence': 8, 'arousal': 10,'tension': 5, 'space': 10},
            {'section': 'resolve',    'valence': 9, 'arousal': 4, 'tension': 1, 'space': 9},
            {'section': 'epilogue',   'valence': 7, 'arousal': 2, 'tension': 0, 'space': 7},
        ],
        'genres': ['orchestral', 'film score', 'post-rock', 'epic', 'trailer'],
    },
}
```

### Mapping Emotions to Production Parameters

```python
def emotion_to_production(emotion_state):
    """Convert emotional dimensions to concrete production parameters."""
    v = emotion_state  # dict with valence, arousal, tension, movement, space

    return {
        # VALENCE → mode, chord quality
        'scale': 'major' if v['valence'] > 6 else 'minor' if v['valence'] < 4 else 'dorian',
        'chord_extensions': v['valence'] > 7,  # Add 9ths/11ths for warmth
        'bright_voicings': v['valence'] > 5,

        # AROUSAL → tempo feel, dynamics, rhythmic drive
        'velocity_range': (60 + v['arousal'] * 4, 80 + v['arousal'] * 2),
        'rhythmic_density': v['arousal'] / 10.0,  # 0=sparse, 1=dense
        'sidechain_depth': 0.3 + v['arousal'] * 0.05,  # Stronger pump at high arousal

        # TENSION → dissonance, harmonic instability, FX intensity
        'add_suspensions': v['tension'] > 6,
        'use_tritones': v['tension'] > 7,
        'reverb_predelay': 10 + v['tension'] * 5,  # Longer = more unsettled
        'delay_feedback': 0.2 + v['tension'] * 0.05,  # More feedback = more tension

        # SPACE → reverb size, stereo width, distance
        'reverb_room': 0.3 + v['space'] * 0.07,  # 0.3 to 1.0
        'reverb_wet': 0.1 + v['space'] * 0.03,   # 0.1 to 0.4
        'stereo_width': 0.7 + v['space'] * 0.06,  # 0.7 to 1.3
        'pad_octave': 2 if v['space'] > 7 else 3,  # Lower = more expansive
    }
```

## Composition Plan

Structure your script's arrangement as a detailed composition plan — each section gets explicit style directives (what to include AND what to exclude):

```python
COMPOSITION_PLAN = {
    'global_styles': {
        'positive': ['warm analog character', 'head-nodding groove', 'rich stereo field'],
        'negative': ['harsh digital artifacts', 'flat dynamics', 'thin low end'],
    },
    'sections': [
        {
            'name': 'Intro',
            'bars': 8,
            'duration_target_ms': 16000,  # ~16s at 120bpm
            'energy': {'intensity': 3, 'density': 2, 'rhythm': 2, 'harmonic': 4, 'brightness': 3},
            'emotion': {'valence': 5, 'arousal': 2, 'tension': 2, 'space': 7},
            'positive_styles': ['atmospheric', 'spacious reverb', 'gentle filter sweep opening'],
            'negative_styles': ['drums', 'full bass', 'bright sounds'],
            'transition_in': None,
            'transition_out': 'filter_open',
        },
        {
            'name': 'Verse 1',
            'bars': 16,
            'energy': {'intensity': 5, 'density': 5, 'rhythm': 5, 'harmonic': 5, 'brightness': 5},
            'emotion': {'valence': 5, 'arousal': 5, 'tension': 3, 'space': 5},
            'positive_styles': ['groove established', 'head-nodding beat', 'warm bass'],
            'negative_styles': ['full chorus energy', 'lead melody'],
            'transition_in': 'impact',
            'transition_out': 'riser',
        },
        # ... more sections
    ],
}
```

## Engagement Techniques by Context

### For Social Media / Short-Form (15-60s)

```python
SHORT_FORM_RULES = {
    'hook_window':    '0-3 seconds — must grab attention immediately',
    'no_long_intros': 'Skip or minimize intro, start near the action',
    'peak_early':     'First energy peak within 8-15 seconds',
    'constant_motion':'Something must change every 4-8 seconds',
    'loop_friendly':  'End should flow back into beginning',
    'bright_mix':     'Higher brightness for phone speakers (boost 2-5kHz)',
}
```

### For Full Tracks (2-4 minutes)

```python
FULL_TRACK_RULES = {
    'establish_mood':    '0-15s: Set the sonic world (ambient, texture, tone)',
    'introduce_groove':  '15-30s: Main rhythm and bass establish the pocket',
    'first_payoff':      '30-60s: First chorus/drop — reward the listeners patience',
    'develop':           '60-120s: Variations, new elements, build/release cycles',
    'climax':            '120-150s: Biggest moment — all elements, max energy, widest stereo',
    'resolve':           '150-180s: Wind down gracefully, callback to opening motif',
    'never_repeat_exact':'No 8-bar section should be an exact copy of another',
    'contrast_ratio':    'Energy difference between peaks and valleys should be 4+ points',
}
```

### For Background/Ambient (3-10 minutes)

```python
AMBIENT_RULES = {
    'slow_evolution':    'Changes happen over 16-32 bars, not 4-8',
    'no_sudden_shifts':  'All transitions use 4+ bar crossfades',
    'subtle_engagement': 'New micro-details emerge every 30-60 seconds',
    'drone_foundation':  'At least one element sustains throughout entire piece',
    'harmonic_drift':    'Chords change every 4-8 bars, not every bar',
    'spatial_movement':  'Pan automation and reverb shifts substitute for rhythmic energy',
}
```

## Energy Interpolation: Smooth Transitions

Never jump between energy levels — always interpolate. The script should compute per-sample energy curves:

```python
def build_energy_curve(sections, total_samples, bar_samples, sr=SR):
    """Build a smooth per-sample energy curve from section definitions.
    Returns dict of numpy arrays, one per dimension."""
    curves = {dim: np.zeros(total_samples) for dim in ENERGY_DIMENSIONS}
    transition_bars = 2  # Bars to crossfade between sections

    bar = 0
    for i, section in enumerate(sections):
        for b in range(section['bars']):
            start = int(bar * bar_samples)
            end = int((bar + 1) * bar_samples)
            if end > total_samples:
                end = total_samples

            for dim in ENERGY_DIMENSIONS:
                target = section['energy'][dim] / 10.0

                # Check if we're in a transition zone
                if b < transition_bars and i > 0:
                    prev = sections[i-1]['energy'][dim] / 10.0
                    t = b / transition_bars
                    value = prev + (target - prev) * smoothstep(t)
                elif b >= section['bars'] - transition_bars and i < len(sections) - 1:
                    next_val = sections[i+1]['energy'][dim] / 10.0
                    t = (b - (section['bars'] - transition_bars)) / transition_bars
                    value = target + (next_val - target) * smoothstep(t)
                else:
                    value = target

                curves[dim][start:end] = value

            bar += 1

    return curves

def smoothstep(t):
    """Hermite smoothstep interpolation (0→1)."""
    t = np.clip(t, 0, 1)
    return t * t * (3 - 2 * t)
```

## Per-Bar Energy Automation Effects

Apply energy curves to control these parameters per bar:

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
    'rhythm': {
        'hat_pattern':      lambda e: 'quarter' if e < 0.3 else '8th' if e < 0.6 else '16th',
        'fill_probability': lambda e: 0.05 + 0.25 * e,        # 5% to 30%
        'ghost_velocity':   lambda e: int(15 + 25 * e),       # 15 to 40
    },
    'harmonic': {
        'chord_type':       lambda e: 'triad' if e < 0.4 else '7th' if e < 0.7 else '9th',
        'voice_spread':     lambda e: 7 + int(10 * e),        # 7 to 17 semitones spread
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
    print("\n🎵 ENERGY MAP")
    print("=" * 70)

    max_bar_width = 40
    for section in sections:
        intensity = section['energy']['intensity']
        density = section['energy']['density']
        bar_fill = "█" * int(intensity * max_bar_width / 10)
        density_dots = "·" * int(density * max_bar_width / 10)

        name = f"{section['name']:.<20}"
        bars = f"({section['bars']}bars)"
        print(f"  {name} {bars:>8}  I:{bar_fill:<{max_bar_width}}")
        print(f"  {'':20} {'':8}  D:{density_dots:<{max_bar_width}}")

    print("=" * 70)
    print("  I=Intensity  D=Density")
    print()
```

## The Golden Rules of Engagement

1. **Front-load the hook** — First 8 bars must establish something memorable (a groove, a sound, a texture) that makes the listener want to hear what's next

2. **Contrast creates interest** — The drop only hits hard because of the breakdown. The chorus only soars because the verse was restrained. Always build contrast:
   - Energy contrast: peaks vs valleys (aim for 4+ points difference)
   - Timbral contrast: warm vs bright sections
   - Spatial contrast: intimate (dry/close) vs epic (reverb/wide)
   - Rhythmic contrast: groove vs breakdown

3. **Something must change every 4-8 bars** — Add a layer, remove a layer, change a pattern, shift a filter. Static = boring. But changes should be subtle enough to feel natural

4. **The 2/3 climax rule** — The biggest moment should come at approximately 2/3 through the track (bar 42 of 64, bar 56 of 80). This mirrors natural storytelling arcs and creates the most satisfying emotional payoff

5. **Strategic silence** — 1-2 beats of silence before a big moment is more powerful than any riser. The absence of sound creates maximum anticipation

6. **Never introduce everything at once** — Spread new elements across 8-16 bars. Each new layer gives the listener something to discover. Front-loading all elements leaves nowhere to go

7. **Call back to the beginning** — The outro should reference the intro (same pad, same filter position, same motif). This creates closure and makes the track feel intentional

8. **Ride the groove, don't fight it** — Once a groove is established, let it breathe for at least 8 bars before major changes. People need time to lock into a rhythm before you can meaningfully depart from it

9. **Ear candy every 16 bars** — A subtle detail that rewards attentive listening: a reversed note, a brief stereo widening, a one-time percussion hit, a vocal chop. These are not structural but they keep engaged listeners discovering new things

10. **End with intent** — Fading out is lazy. Either resolve decisively (final chord + reverb tail) or create a deliberate loop point. The last 4 bars should feel like a conclusion, not an abandonment
