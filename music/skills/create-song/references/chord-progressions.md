# Chord Progressions

Advanced chord progressions for neo-soul, gospel, film score, video game music, and advanced harmonic techniques. Plus bass patterns and arpeggio patterns.

## Neo-Soul

```python
NEOSOUL_PROGRESSIONS = {
    'erykah_badu': {
        'chords': ['Fmaj9', 'Em7', 'Dm9', 'Cmaj7'],
        'roman': ['IVmaj9', 'iii7', 'ii9', 'Imaj7'],
        'key': 'C', 'bars_per_chord': 2,
        'description': 'Descending diatonic 7ths. Classic neo-soul cadence.',
    },
    'dangelo': {
        'chords': ['Dm9', 'G13', 'Cmaj9', 'Fmaj7#11'],
        'roman': ['ii9', 'V13', 'Imaj9', 'IVmaj7#11'],
        'key': 'C', 'bars_per_chord': 2,
        'description': 'ii-V-I with extended voicings. The #11 on IV = Lydian color.',
    },
    'hiatus_kaiyote': {
        'chords': ['AbMaj7', 'Gm7', 'Cm9', 'Fm7', 'Bb13'],
        'roman': ['bVImaj7', 'v7', 'i9', 'iv7', 'bVII13'],
        'key': 'Cm',
        'description': 'Modal interchange. bVI major gives unexpected brightness.',
    },
    'classic_turnaround': {
        'chords': ['Cmaj9', 'Am7', 'Dm7', 'G7#9'],
        'roman': ['Imaj9', 'vi7', 'ii7', 'V7#9'],
        'key': 'C',
        'description': 'I-vi-ii-V with Hendrix chord (#9) on V.',
    },
    'gospel_neosoul': {
        'chords': ['Db/Eb', 'AbMaj7', 'Fm9', 'Bbm7', 'Eb13sus4', 'Eb13'],
        'key': 'Ab',
        'description': 'Gospel-influenced. Sus4 resolving to 13 = gospel cadence.',
    },
}
```

## Gospel

```python
GOSPEL_PROGRESSIONS = {
    'basic_shout': {
        'chords': ['C', 'C/E', 'F', 'G', 'Am', 'G/B', 'C'],
        'roman': ['I', 'I/3', 'IV', 'V', 'vi', 'V/3', 'I'],
        'key': 'C',
        'description': 'Ascending bass line (C-E-F-G-A-B-C). Builds energy.',
    },
    'turnaround': {
        'chords': ['Ab', 'Ab/C', 'Db', 'Dbm', 'Ab/Eb', 'Eb7', 'Ab'],
        'roman': ['I', 'I/3', 'IV', 'iv', 'I/5', 'V7', 'I'],
        'key': 'Ab',
        'description': 'Major IV to minor iv is the gospel money chord.',
    },
    'chain_of_251s': {
        'chords': ['Am7','D7','Gmaj7', 'Gm7','C7','Fmaj7', 'Fm7','Bb7','Ebmaj7'],
        'key': 'C (descending)',
        'description': 'ii-V-I chains descending by whole steps. Jazz-gospel crossover.',
    },
    'clark_sisters': {
        'chords': ['Cmaj7','C7','Fmaj7','Fm6','Cmaj7/G','A7#9','Dm9','G13'],
        'key': 'C',
        'description': 'I becomes V7/IV. Minor iv6 is the dark pivot.',
    },
    'praise_break_vamp': {
        'chords': ['Bb/C', 'C', 'Bb/C', 'C'],
        'key': 'C',
        'technique': 'Raise key by half step every 4 bars for building intensity.',
    },
}
```

## Film Score

```python
FILM_SCORE_PROGRESSIONS = {
    'zimmer_inception': {
        'chords': ['Am', 'Am/G', 'Fmaj7', 'Dm'],
        'roman': ['i', 'i/b7', 'bVImaj7', 'iv'],
        'key': 'Am',
        'description': 'Pedal-tone descent in bass. Massive with brass + strings.',
    },
    'zimmer_interstellar': {
        'chords': ['Am', 'C', 'Dm', 'F'],
        'roman': ['i', 'bIII', 'iv', 'bVI'],
        'key': 'Am',
        'description': 'All minor-side chords. Organ + strings. Hopeful yet melancholic.',
    },
    'zimmer_dark_knight': {
        'chords': ['Dm', 'Bb', 'Dm', 'A'],
        'roman': ['i', 'bVI', 'i', 'V'],
        'key': 'Dm',
        'description': 'Two-chord oscillation with dramatic V resolution.',
    },
    'williams_superman': {
        'chords': ['C', 'G/B', 'Am', 'C/G', 'F', 'G', 'C'],
        'roman': ['I', 'V/3', 'vi', 'I/5', 'IV', 'V', 'I'],
        'key': 'C',
        'description': 'Heroic major key, descending bass C-B-A-G-F then V-I.',
    },
    'williams_et': {
        'chords': ['C', 'Ab', 'C', 'Ab', 'Bb', 'C'],
        'roman': ['I', 'bVI', 'I', 'bVI', 'bVII', 'I'],
        'key': 'C',
        'description': 'Chromatic mediants (I-bVI). The "wonder" sound. Magical, awe.',
    },
    'williams_jaws': {
        'chords': ['Em', 'F', 'Em', 'F'],
        'roman': ['i', 'bII', 'i', 'bII'],
        'key': 'Em',
        'description': 'Semitone ostinato. Dread and approaching danger.',
    },
    'epic_trailer': {
        'chords': ['Cm', 'Ab', 'Eb', 'Bb'],
        'roman': ['i', 'bVI', 'bIII', 'bVII'],
        'key': 'Cm',
        'description': 'Power chords with epic percussion. Every trailer ever.',
    },
    'mystery_tension': {
        'chords': ['Am', 'F#dim', 'Fmaj7', 'E7'],
        'roman': ['i', '#vi_dim', 'bVImaj7', 'V7'],
        'key': 'Am',
        'description': 'Diminished chord creates unease, resolves through bVI.',
    },
    'romantic_theme': {
        'chords': ['Db', 'Ab/C', 'Bbm', 'Gb', 'Ab', 'Db'],
        'roman': ['I', 'V/3', 'vi', 'IV', 'V', 'I'],
        'key': 'Db',
        'description': 'Warm, sweeping romance. Db major = rich, full key for strings.',
    },
}
```

## Video Game Music

```python
GAME_MUSIC_PROGRESSIONS = {
    'jrpg_town': {
        'chords': ['C', 'Am', 'Dm', 'G'],
        'roman': ['I', 'vi', 'ii', 'V'],
        'key': 'C',
        'description': 'Safe, warm, inviting. Classic town/village theme.',
    },
    'jrpg_battle': {
        'chords': ['Dm', 'Bb', 'C', 'Dm', 'Gm', 'A7', 'Dm'],
        'roman': ['i', 'bVI', 'bVII', 'i', 'iv', 'V7', 'i'],
        'key': 'Dm',
        'description': 'Driving minor key. Fast tempo. bVI-bVII-i = power.',
    },
    'zelda_overworld': {
        'chords': ['Bb', 'F', 'C', 'F'],
        'roman': ['IV', 'I', 'V', 'I'],
        'key': 'F',
        'description': 'Adventure, open world, discovery. Major key brightness.',
    },
    'boss_fight': {
        'chords': ['Em', 'C', 'D', 'B7'],
        'roman': ['i', 'bVI', 'bVII', 'V7'],
        'key': 'Em',
        'description': 'Intense. B7 is the dramatic V7 resolution back to Em.',
    },
    'menu_ambient': {
        'chords': ['Cmaj7', 'Fmaj7'],
        'roman': ['Imaj7', 'IVmaj7'],
        'key': 'C', 'bars_per_chord': 4,
        'description': 'Two chords alternating slowly. Calm, contemplative.',
    },
    'victory_fanfare': {
        'chords': ['Bb', 'C', 'F', 'Dm', 'Bb', 'C', 'F'],
        'roman': ['IV', 'V', 'I', 'vi', 'IV', 'V', 'I'],
        'key': 'F',
        'description': 'Triumphant! Short, punchy. Brass fanfare.',
    },
    'underwater_level': {
        'chords': ['Fmaj7', 'Em7', 'Am7', 'Dm7'],
        'key': 'C',
        'description': 'Diatonic 7ths, descending. Dreamy, submerged feel.',
    },
    'retro_chiptune': {
        'chords': ['C', 'F', 'G', 'Am'],
        'roman': ['I', 'IV', 'V', 'vi'],
        'key': 'C',
        'description': 'Simple triads for 8-bit. Square wave arps.',
    },
}
```

## Advanced Progressions

```python
ADVANCED_PROGRESSIONS = {
    'modulation_up_half': {
        'chords': ['C', 'G', 'Am', 'F', '|', 'Db', 'Ab', 'Bbm', 'Gb'],
        'technique': 'Direct modulation up a half step. Use at chorus 2 or final chorus.',
    },
    'modulation_up_whole': {
        'chords': ['C', 'G', 'Am', 'F', '|', 'D', 'A', 'Bm', 'G'],
        'technique': 'Truck driver modulation. Common in pop/country.',
    },
    'modulation_relative': {
        'chords': ['C', 'Am', 'F', 'G', '|', 'Am', 'F', 'Dm', 'E7'],
        'technique': 'Pivot to relative minor. Am is shared between C major and A minor.',
    },
    'coltrane_changes': {
        'chords': ['Bmaj7','D7','Gmaj7','Bb7','Ebmaj7','Gb7','Bmaj7'],
        'key': 'B (modulating through 3 keys M3 apart)',
        'description': 'Giant Steps. Tonal centers a major third apart: B-G-Eb.',
    },
    'deceptive_cadences': {
        'V_to_vi':       ['G7', 'Am'],       # Most common deceptive
        'V_to_bVI':      ['G7', 'Ab'],       # Dramatic surprise (Picardy neighbor)
        'V_to_IV':       ['G7', 'F'],        # Backdoor resolution
        'V_to_iv':       ['G7', 'Fm'],       # Dark surprise
        'V_to_bVII':     ['G7', 'Bb'],       # Modal surprise
        'V_to_N6':       ['G7', 'Ab/C'],     # Neapolitan resolution (very dramatic)
    },
    'chromatic_mediants': {
        'I_to_bVI':  ['C', 'Ab'],   # Down major 3rd (share C note). Dramatic, cinematic.
        'I_to_bIII': ['C', 'Eb'],   # Down minor 3rd (share G note). Mysterious.
        'I_to_III':  ['C', 'E'],    # Up major 3rd (share E note). Bright, magical.
        'I_to_VI':   ['C', 'A'],    # Up minor 3rd (share none). Bold shift.
    },
    'tonic_pedal': {
        'chords': ['C', 'Dm/C', 'Em/C', 'F/C', 'G/C', 'Am/C', 'G/C', 'C'],
        'description': 'Bass holds tonic while chords move above. Stability + color.',
    },
    'dominant_pedal': {
        'chords': ['G', 'C/G', 'Dm/G', 'C/G', 'F/G', 'C/G', 'G', 'C'],
        'description': 'Bass holds dominant. Builds massive tension before final I.',
    },
}
```

## Bass Patterns

Format: `(step, midi_offset_from_root, velocity, duration_steps)` on 16-step grid.

```python
BASS_PATTERNS = {
    'walking_jazz': {
        'notes': [(0,0,100,4), (4,4,90,4), (8,7,95,4), (12,5,85,4)],
        'bpm': (100, 200), 'swing': 0.62,
        'description': 'Quarter-note walk. Root-3rd-5th-4th. Voice-lead to next chord root.',
    },
    'funk_slap': {
        'notes': [(0,0,110,2), (2,-12,60,1), (4,0,100,2), (7,12,90,1),
                  (8,0,110,2), (10,-12,55,1), (12,0,100,1), (13,7,80,1), (14,5,85,2)],
        'bpm': (95, 120),
        'description': 'Slap on root, pop on octave. Dead notes (-12 = muted thump).',
    },
    'funk_fingerstyle': {
        'notes': [(0,0,100,2), (3,0,80,1), (4,7,90,2), (6,5,85,1),
                  (8,0,100,2), (10,3,75,2), (12,5,90,1), (13,7,85,1), (14,0,95,2)],
        'bpm': (100, 125),
        'description': 'Syncopated 16th-note funk. Chromatic approach tones.',
    },
    'reggae_one_drop': {
        'notes': [(0,0,110,6), (8,0,100,4), (14,7,80,2)],
        'bpm': (65, 85),
        'description': 'Long held root, short fill notes. Bass provides weight on beat 1.',
    },
    'edm_wobble': {
        'notes': [(0,0,120,16)],  # Sustained note, rhythm from filter LFO
        'filter_lfo_rate': '1/8',  # LFO synced to tempo
        'bpm': (140, 150),
        'description': 'Single held note. The "wobble" comes from LFO on filter cutoff.',
    },
    'reese_bass': {
        'notes': [(0,0,110,8), (8,5,100,8)],
        'synthesis': 'detuned_saws',  # 2-3 saws, 10-20 cents detune
        'bpm': (170, 180),
        'description': 'DnB staple. Detuned saws, slow filter movement. Half-note rhythm.',
    },
    'acid_303': {
        'notes': [(0,0,100,2), (2,0,70,1), (3,12,90,1), (4,7,100,2),
                  (6,5,80,1), (7,3,85,1), (8,0,110,2), (10,12,100,1),
                  (11,7,70,1), (12,0,90,2), (14,5,100,2)],
        'slides': [3, 7, 10],  # Step indices where slide (portamento) occurs
        'accents': [0, 4, 8],  # Step indices with accent (higher filter cutoff)
        'bpm': (125, 140),
        'description': 'TB-303 acid. Slides and accents are essential.',
    },
    'motown': {
        'notes': [(0,0,100,2), (2,7,80,2), (4,0,90,2), (6,4,75,1), (7,5,80,1),
                  (8,7,95,2), (10,5,80,2), (12,0,100,2), (14,4,75,2)],
        'bpm': (95, 125),
        'description': 'James Jamerson style. Melodic, chromatic, counterpoint to vocal.',
    },
}
```

## Arpeggio Patterns

```python
ARPEGGIO_TYPES = {
    'up':       lambda notes: notes,
    'down':     lambda notes: notes[::-1],
    'up_down':  lambda notes: notes + notes[-2:0:-1],
    'down_up':  lambda notes: notes[::-1] + notes[1:-1],
    'random':   lambda notes: [notes[i] for i in np.random.permutation(len(notes))],
    'converge': lambda notes: [notes[i//2] if i%2==0 else notes[-(i//2+1)] for i in range(len(notes))],
    'diverge':  lambda notes: [notes[len(notes)//2 + (i//2 if i%2==0 else -(i//2+1))] for i in range(len(notes))],
    'pedal_top': lambda notes: [x for n in notes for x in (n, notes[-1])],
    'pedal_bottom': lambda notes: [x for n in notes for x in (notes[0], n)],
    'skip_up':  lambda notes: [notes[i] for i in range(0, len(notes), 2)] + [notes[i] for i in range(1, len(notes), 2)],
}

ARPEGGIO_RHYTHMS = {
    'straight_8th': [1,0,1,0,1,0,1,0,1,0,1,0,1,0,1,0],
    'straight_16th': [1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1],
    'dotted_8th': [1,0,0,1,0,0,1,0,0,1,0,0,1,0,0,1],
    'trance_gate': {
        'pattern': [1,1,0,1,1,0,1,0,1,1,0,1,1,0,1,0],
        'velocity': [100,80,0,90,70,0,100,0,80,90,0,100,70,0,90,0],
    },
    'italo_disco': [1,0,0,1,0,0,1,0,1,0,0,1,0,0,1,0],
}

def generate_arp(chord_notes, pattern_type='up', octaves=2, rhythm='straight_8th'):
    """Generate an arpeggio pattern from chord notes.
    chord_notes: list of MIDI note numbers (e.g., [60, 64, 67])
    Returns: list of (step, midi_note, velocity) tuples
    """
    extended = []
    for oct in range(octaves):
        extended.extend([n + oct * 12 for n in chord_notes])
    ordered = ARPEGGIO_TYPES[pattern_type](extended)
    if isinstance(rhythm, str):
        rhythm_pattern = ARPEGGIO_RHYTHMS[rhythm]
    else:
        rhythm_pattern = rhythm
    if isinstance(rhythm_pattern, dict):
        steps = rhythm_pattern['pattern']
        vels = rhythm_pattern['velocity']
    else:
        steps = rhythm_pattern
        vels = [100] * len(steps)
    result = []
    note_idx = 0
    for step, (hit, vel) in enumerate(zip(steps, vels)):
        if hit:
            result.append((step, ordered[note_idx % len(ordered)], vel))
            note_idx += 1
    return result
```
