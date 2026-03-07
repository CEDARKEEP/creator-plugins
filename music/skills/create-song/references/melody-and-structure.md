# Melody Data & Song Structure Templates

Melody generation data, contour archetypes, riff patterns, song structure templates, transition techniques, and JSON schemas.

## Interval Weights (for probabilistic generation)

```python
MELODY_INTERVAL_WEIGHTS = {
    # interval_semitones: (probability, direction_bias)
    # direction_bias: 0.5=equal, >0.5=favor up, <0.5=favor down
    0:  (0.10, 0.50),   # unison
    1:  (0.13, 0.52),   # minor 2nd
    2:  (0.17, 0.53),   # major 2nd (most common melodic motion)
    3:  (0.07, 0.50),   # minor 3rd
    4:  (0.05, 0.50),   # major 3rd
    5:  (0.04, 0.50),   # perfect 4th
    7:  (0.03, 0.50),   # perfect 5th
    8:  (0.01, 0.45),   # minor 6th (usually down)
    9:  (0.01, 0.45),   # major 6th (usually down)
    12: (0.01, 0.55),   # octave
}

POST_SKIP_RULES = {
    'ascending_skip_gt4':  {'descend_stepwise': 0.85, 'continue': 0.15},
    'descending_skip_gt4': {'ascend_stepwise': 0.80, 'continue': 0.20},
    'skip_skip':           {'change_direction': 0.70, 'continue': 0.30},
}
```

## Melody Contour Archetypes

```python
MELODY_CONTOURS = {
    # Values 0.0-1.0 representing pitch height over 8 time positions
    'arch':           [0.3, 0.5, 0.7, 0.9, 1.0, 0.8, 0.5, 0.3],  # Rise then fall
    'inverted_arch':  [0.8, 0.5, 0.3, 0.2, 0.1, 0.3, 0.6, 0.8],  # Dip then recover
    'ascending':      [0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9],  # Steady climb
    'descending':     [0.9, 0.8, 0.7, 0.6, 0.5, 0.4, 0.3, 0.2],  # Steady fall
    'plateau':        [0.3, 0.5, 0.8, 0.8, 0.8, 0.8, 0.5, 0.3],  # Hold at peak
    'zigzag':         [0.3, 0.7, 0.4, 0.8, 0.3, 0.9, 0.5, 0.3],  # Oscillating
    'delayed_climax': [0.3, 0.3, 0.4, 0.5, 0.5, 0.8, 1.0, 0.4],  # Peak at 2/3
    'question':       [0.3, 0.5, 0.8, 0.6],  # Half phrase, ends up (tension)
    'answer':         [0.7, 0.5, 0.3, 0.2],  # Half phrase, ends down (resolution)
}
```

## Riff Patterns (semitone offsets from root)

```python
RIFF_PATTERNS = {
    'blues':       [0, 0, 3, 5, 6, 5, 3, 0],      # Blue note (b5) approach
    'rock_power':  [0, 0, 7, 7, 5, 5, 0, 0],       # Power chord roots
    'funk':        [0, 0, 10, 0, 7, 0, 5, 3],       # Syncopated, b7 emphasis
    'metal_gallop':[0, 0, 0, 0, 0, 0, 7, 5],        # Gallop rhythm, power intervals
    'edm_lead':    [0, 3, 5, 7, 5, 3, 0, -2],       # Minor scale up-down
    'surf':        [0, 4, 7, 12, 7, 4, 0, -5],      # Arpeggiated, reverb-drenched
    'middle_east': [0, 1, 4, 5, 7, 8, 11, 12],      # Phrygian dominant ascending
    'jazz_bebop':  [0, 2, 4, 5, 7, 9, 10, 11],      # Bebop dominant scale run
}
```

## Cross-Cultural Pentatonic Patterns

```python
PENTATONIC_SCALES = {
    'blues_box':       [0, 3, 5, 7, 10],        # Minor pentatonic (universal blues)
    'major_pent':      [0, 2, 4, 7, 9],          # Major pentatonic (folk, country)
    'chinese_gong':    [0, 2, 4, 7, 9],          # Same intervals, different context
    'japanese_miyako': [0, 1, 5, 7, 8],          # In-sen scale variant, dark
    'japanese_yo':     [0, 2, 5, 7, 9],          # Bright Japanese folk
    'west_african':    [0, 2, 4, 7, 9],          # Widespread in West African music
    'celtic':          [0, 2, 4, 7, 9],          # Irish/Scottish folk
    'andean':          [0, 3, 5, 7, 10],         # South American folk
    'indian_bhupali':  [0, 2, 4, 7, 9],          # Raga Bhupali (Carnatic: Mohanam)
}
```

## Call and Response

```python
CALL_RESPONSE_TYPES = {
    'exact_repeat': {
        'description': 'Response = exact copy of call',
        'transform': lambda notes: notes,
    },
    'transposed': {
        'description': 'Response = call transposed up a 4th (5 semitones)',
        'transform': lambda notes: [(n + 5, d, v) for n, d, v in notes],
    },
    'inverted': {
        'description': 'Response = call with intervals mirrored around axis',
        'transform': lambda notes: [(notes[0][0] - (n - notes[0][0]), d, v) for n, d, v in notes],
    },
    'complementary': {
        'description': 'Call ends on tension (non-tonic), response ends on resolution (tonic/5th)',
        'call_end': 'scale_degree_2_or_7',
        'response_end': 'scale_degree_1_or_5',
    },
}
```

## Motif Development Techniques

```python
MOTIF_DEVELOPMENT = {
    'augmentation':     'Double all note durations',
    'diminution':       'Halve all note durations',
    'inversion':        'Mirror intervals: up becomes down, down becomes up',
    'retrograde':       'Play notes in reverse order',
    'retrograde_inv':   'Retrograde + inversion combined',
    'sequence_real':    'Repeat motif at different pitch, preserving exact intervals',
    'sequence_tonal':   'Repeat motif at different pitch within the key (intervals adjust)',
    'fragmentation':    'Use only the first 2-3 notes of the motif',
    'extension':        'Add 1-2 notes to the end of the motif',
    'rhythmic_displace':'Shift motif start by 1 beat (or half beat)',
    'ornamentation':    'Add passing tones, neighbor tones, or turns between motif notes',
}
```

## Song Structure Templates

```python
SONG_TEMPLATES = {
    'pop_standard': {
        'total_bars': 64,
        'sections': [
            {'name': 'Intro',      'bars': 4,  'energy': 3, 'elements': ['pad', 'light_drums']},
            {'name': 'Verse 1',    'bars': 8,  'energy': 5, 'elements': ['drums', 'bass', 'pad', 'melody']},
            {'name': 'Pre-Chorus', 'bars': 4,  'energy': 6, 'elements': ['drums', 'bass', 'chords', 'melody', 'riser']},
            {'name': 'Chorus 1',   'bars': 8,  'energy': 8, 'elements': ['full_drums', 'bass', 'chords', 'lead', 'arp']},
            {'name': 'Verse 2',    'bars': 8,  'energy': 5, 'elements': ['drums', 'bass', 'pad', 'melody']},
            {'name': 'Pre-Chorus', 'bars': 4,  'energy': 7, 'elements': ['drums', 'bass', 'chords', 'melody', 'riser']},
            {'name': 'Chorus 2',   'bars': 8,  'energy': 9, 'elements': ['full_drums', 'bass', 'chords', 'lead', 'arp', 'strings']},
            {'name': 'Bridge',     'bars': 8,  'energy': 6, 'elements': ['light_drums', 'bass', 'pad', 'melody']},
            {'name': 'Chorus 3',   'bars': 8,  'energy': 10,'elements': ['full_drums', 'bass', 'chords', 'lead', 'arp', 'strings', 'fx']},
            {'name': 'Outro',      'bars': 4,  'energy': 4, 'elements': ['pad', 'light_drums', 'melody']},
        ],
    },
    'edm_drop': {
        'total_bars': 88,
        'sections': [
            {'name': 'Intro',       'bars': 8,  'energy': 3, 'elements': ['pad', 'atmosphere']},
            {'name': 'Build 1',     'bars': 8,  'energy': 5, 'elements': ['drums', 'bass', 'arp', 'riser']},
            {'name': 'Breakdown 1', 'bars': 8,  'energy': 4, 'elements': ['pad', 'melody', 'filter_sweep']},
            {'name': 'Build 2',     'bars': 8,  'energy': 7, 'elements': ['drums', 'bass', 'arp', 'riser', 'snare_roll']},
            {'name': 'Drop 1',      'bars': 16, 'energy': 10,'elements': ['full_drums', 'bass', 'lead', 'chords', 'sidechain']},
            {'name': 'Breakdown 2', 'bars': 8,  'energy': 3, 'elements': ['pad', 'atmosphere', 'vocal_chop']},
            {'name': 'Build 3',     'bars': 8,  'energy': 8, 'elements': ['drums', 'bass', 'riser', 'snare_roll', 'pitch_riser']},
            {'name': 'Drop 2',      'bars': 16, 'energy': 10,'elements': ['full_drums', 'bass', 'lead', 'chords', 'sidechain', 'fx']},
            {'name': 'Outro',       'bars': 8,  'energy': 4, 'elements': ['pad', 'atmosphere', 'filter_sweep']},
        ],
    },
    'hip_hop': {
        'total_bars': 72,
        'sections': [
            {'name': 'Intro',    'bars': 4,  'energy': 3, 'elements': ['sample', 'atmosphere']},
            {'name': 'Verse 1',  'bars': 16, 'energy': 6, 'elements': ['drums', 'bass', 'sample', 'pad']},
            {'name': 'Hook',     'bars': 8,  'energy': 8, 'elements': ['full_drums', 'bass', 'melody', 'chords']},
            {'name': 'Verse 2',  'bars': 16, 'energy': 6, 'elements': ['drums', 'bass', 'sample', 'pad']},
            {'name': 'Hook',     'bars': 8,  'energy': 8, 'elements': ['full_drums', 'bass', 'melody', 'chords']},
            {'name': 'Bridge',   'bars': 8,  'energy': 5, 'elements': ['light_drums', 'bass', 'pad']},
            {'name': 'Hook',     'bars': 8,  'energy': 9, 'elements': ['full_drums', 'bass', 'melody', 'chords', 'fx']},
            {'name': 'Outro',    'bars': 4,  'energy': 3, 'elements': ['pad', 'atmosphere']},
        ],
    },
    'jazz_aaba': {
        'total_bars': 128,
        'sections': [
            {'name': 'Head A1',   'bars': 8,  'energy': 5, 'elements': ['piano', 'bass', 'drums', 'melody']},
            {'name': 'Head A2',   'bars': 8,  'energy': 5, 'elements': ['piano', 'bass', 'drums', 'melody']},
            {'name': 'Head B',    'bars': 8,  'energy': 6, 'elements': ['piano', 'bass', 'drums', 'melody']},
            {'name': 'Head A3',   'bars': 8,  'energy': 5, 'elements': ['piano', 'bass', 'drums', 'melody']},
            {'name': 'Solo 1',    'bars': 32, 'energy': 7, 'elements': ['piano', 'bass', 'drums', 'solo']},
            {'name': 'Solo 2',    'bars': 32, 'energy': 8, 'elements': ['piano', 'bass', 'drums', 'solo']},
            {'name': 'Head Out',  'bars': 32, 'energy': 6, 'elements': ['piano', 'bass', 'drums', 'melody']},
        ],
    },
    'ambient': {
        'total_bars': 64,
        'sections': [
            {'name': 'Emergence',    'bars': 16, 'energy': 2, 'elements': ['drone', 'texture', 'atmosphere']},
            {'name': 'Development',  'bars': 16, 'energy': 4, 'elements': ['drone', 'pad', 'melody', 'texture']},
            {'name': 'Plateau',      'bars': 16, 'energy': 5, 'elements': ['drone', 'pad', 'melody', 'arp', 'texture']},
            {'name': 'Dissolution',  'bars': 16, 'energy': 2, 'elements': ['drone', 'texture']},
        ],
    },
    'metal': {
        'total_bars': 68,
        'sections': [
            {'name': 'Intro Riff',  'bars': 4,  'energy': 7, 'elements': ['guitar', 'bass', 'drums']},
            {'name': 'Verse',       'bars': 8,  'energy': 6, 'elements': ['guitar', 'bass', 'drums', 'vocal']},
            {'name': 'Pre-Chorus',  'bars': 4,  'energy': 7, 'elements': ['guitar', 'bass', 'drums', 'vocal']},
            {'name': 'Chorus',      'bars': 8,  'energy': 9, 'elements': ['guitar', 'bass', 'full_drums', 'vocal', 'harmony']},
            {'name': 'Verse 2',     'bars': 8,  'energy': 6, 'elements': ['guitar', 'bass', 'drums', 'vocal']},
            {'name': 'Chorus 2',    'bars': 8,  'energy': 9, 'elements': ['guitar', 'bass', 'full_drums', 'vocal', 'harmony']},
            {'name': 'Breakdown',   'bars': 8,  'energy': 8, 'elements': ['guitar', 'bass', 'half_time_drums']},
            {'name': 'Solo',        'bars': 8,  'energy': 10,'elements': ['guitar_solo', 'bass', 'full_drums', 'harmony']},
            {'name': 'Chorus 3',    'bars': 8,  'energy': 10,'elements': ['guitar', 'bass', 'full_drums', 'vocal', 'harmony', 'fx']},
            {'name': 'Outro',       'bars': 4,  'energy': 5, 'elements': ['guitar', 'bass', 'drums']},
        ],
    },
}
```

## Transition Techniques

```python
TRANSITIONS = {
    'fill':           {'description': 'Drum fill (1-4 beats)', 'use': 'Every 4-8 bars'},
    'riser':          {'description': 'Noise sweep + snare roll over 4-16 bars', 'use': 'Before drops/choruses'},
    'downlifter':     {'description': 'Reverse of riser, HPF 16k->200Hz', 'use': 'After drops'},
    'impact':         {'description': 'Kick + crash + sub boom on beat 1', 'use': 'Section starts'},
    'silence':        {'description': '1-2 beats of silence before impact', 'use': 'Before big moments'},
    'filter_sweep':   {'description': 'LPF opening 200->20kHz over 4-8 bars', 'use': 'Intro, builds'},
    'reverse_reverb': {'description': 'Reversed heavy reverb tail', 'use': 'Before vocal/lead entry'},
    'pitch_drop':     {'description': 'Pitch bend down 12-24 semitones', 'use': 'End of sections'},
    'stutter':        {'description': 'Buffer repeat getting faster', 'use': 'Before drops, builds'},
}
```

## JSON Schemas

### Drum Pattern Schema

```json
{
    "name": "string",
    "genre": "string",
    "bpm": {"min": "number", "max": "number"},
    "time_signature": "string (default '4/4')",
    "steps": "integer (default 16)",
    "swing": "number (0.5-0.75, default 0.5)",
    "tracks": {
        "track_name": {
            "pattern": [0, 1, 0, 1, "..."],
            "velocity": [0, 100, 0, 90, "..."],
            "midi_note": "integer",
            "probability": "number (0.0-1.0, default 1.0)"
        }
    }
}
```

### Chord Progression Schema

```json
{
    "name": "string",
    "key": "string",
    "mode": "string",
    "bpm": {"min": "number", "max": "number"},
    "chords": [
        {
            "symbol": "string (e.g. 'Cmaj7')",
            "roman": "string (e.g. 'Imaj7')",
            "root_midi": "integer",
            "intervals": [0, 4, 7, 11],
            "voicing_midi": [48, 55, 64, 67, 71],
            "duration_bars": "number (default 1)"
        }
    ]
}
```

### Song Template Schema

```json
{
    "genre": "string",
    "bpm": "number",
    "key": "string",
    "scale": "string",
    "swing": "number",
    "total_bars": "integer",
    "sections": [
        {
            "name": "string",
            "start_bar": "integer",
            "length_bars": "integer",
            "energy": "integer (0-10)",
            "chord_progression": "string (reference key)",
            "drum_pattern": "string (reference key)",
            "elements": ["string"],
            "transition_in": "string (reference key)",
            "transition_out": "string (reference key)"
        }
    ]
}
```
