# Psychoacoustics & Human Hearing

How the human auditory system perceives sound, and how to exploit that knowledge for better music synthesis. All implementations in numpy/scipy.

## Contents

- [Equal-Loudness Contours (ISO 226:2003)](#1-equal-loudness-contours-iso-2262003)
- [Critical Bands & the Bark Scale](#2-critical-bands--the-bark-scale)
- [Auditory Masking](#3-auditory-masking)
- [Consonance & Dissonance](#4-consonance--dissonance)
- [The Missing Fundamental (Virtual Pitch)](#5-the-missing-fundamental-virtual-pitch)
- [Frequency-Emotion Mapping](#6-frequency-emotion-mapping)
- [Temporal Perception](#7-temporal-perception)
- [LUFS Metering (ITU-R BS.1770-4)](#8-lufs-metering-itu-r-bs1770-4)
- [Harmonic Series & Timbre Perception](#9-harmonic-series--timbre-perception)
- [Psychoacoustic Bass Enhancement](#10-psychoacoustic-bass-enhancement)
- [Quick Reference: Key Psychoacoustic Numbers](#quick-reference-key-psychoacoustic-numbers)

---

## 1. Equal-Loudness Contours (ISO 226:2003)

Human hearing is not flat. We're most sensitive at 2-5kHz (ear canal resonance) and least sensitive at very low (<100Hz) and very high (>10kHz) frequencies. At low volumes, bass and treble "disappear" first.

### Key Frequencies and Sensitivity

```python
# Approximate equal-loudness contour offsets relative to 1kHz (in dB)
# At moderate listening levels (~70 phon)
EQUAL_LOUDNESS_OFFSETS = {
    # freq_hz: dB_boost_needed_to_sound_as_loud_as_1kHz
    20:    +30,
    31.5:  +22,
    50:    +16,
    63:    +14,
    100:   +8,
    125:   +6,
    200:   +3,
    250:   +1,
    500:   0,
    1000:  0,     # Reference
    2000:  -2,    # MORE sensitive here
    3150:  -4,    # Peak sensitivity (ear canal resonance)
    4000:  -3,
    5000:  -1,
    6300:  +1,
    8000:  +3,
    10000: +6,
    12500: +12,
    16000: +20,
}
```

### Loudness Compensation Filter

At low listening levels, boost bass and treble to maintain perceived balance:

```python
def loudness_compensation(sig, listening_level_db=70, target_level_db=85, sr=SR):
    """Apply equal-loudness compensation. At lower volumes, boost bass and treble.
    listening_level_db: expected playback level (60=quiet room, 70=moderate, 85=studio).
    target_level_db: the level the mix was designed for."""
    if listening_level_db >= target_level_db:
        return sig  # No compensation needed at loud levels

    diff = target_level_db - listening_level_db
    # More compensation needed at greater level differences
    bass_boost = diff * 0.15   # ~1.5dB bass boost per 10dB quieter
    treble_boost = diff * 0.08  # ~0.8dB treble boost per 10dB quieter

    sig = low_shelf(sig, 100, bass_boost, sr=sr)
    sig = high_shelf(sig, 8000, treble_boost, sr=sr)
    return sig
```

### Production Implications

- Mix at moderate volumes (~75-85 dB SPL) — Fletcher-Munson curves are flattest here
- Check mixes at low volume — if bass and vocals disappear, the mix balance is wrong
- The 3-5kHz range is where human hearing is most sensitive — harsh synths/vocals live here. Be careful with boosts in this range
- Sub bass (20-60Hz) needs to be much louder in amplitude to be perceived as equal loudness to mid-range. This is why 808s are so loud in the mix
- "Air" frequencies (10-16kHz) are perceived as quieter — you can boost more here without harshness

---

## 2. Critical Bands & the Bark Scale

The cochlea divides sound into ~24 overlapping frequency bands called critical bands. When two tones fall within the same critical band, they interact (beat/roughness). When they're in separate bands, they're perceived independently.

### Critical Bandwidth by Frequency

```python
def critical_bandwidth(freq_hz):
    """ERB (Equivalent Rectangular Bandwidth) in Hz at a given frequency.
    Based on Glasberg & Moore 1990."""
    return 24.7 * (4.37 * freq_hz / 1000 + 1)

def freq_to_bark(freq_hz):
    """Convert frequency to Bark scale (perceptual frequency scale)."""
    return 13 * np.arctan(0.00076 * freq_hz) + 3.5 * np.arctan((freq_hz / 7500) ** 2)

# Critical bandwidth at key frequencies:
CRITICAL_BANDWIDTHS = {
    # freq_hz: bandwidth_hz
    100:   90,    # Very wide relative to frequency
    200:   100,
    500:   110,
    1000:  130,
    2000:  200,   # Wider absolute bandwidth at higher freqs
    4000:  350,
    8000:  650,
    16000: 1200,
}
```

### Voicing and Mixing Implications

```python
def min_voicing_interval(base_freq_hz):
    """Minimum interval (in semitones) between chord notes to avoid roughness.
    Notes closer than ~1 critical bandwidth apart create dissonant beating."""
    cbw = critical_bandwidth(base_freq_hz)
    # Convert bandwidth to semitones: semitones = 12 * log2(1 + cbw/freq)
    min_semitones = 12 * np.log2(1 + cbw / base_freq_hz)
    return round(min_semitones, 1)

# Results — minimum clean voicing intervals:
# At 100Hz: ~12 semitones (octave!) — this is why low bass chords sound muddy
# At 200Hz: ~7 semitones (5th) — keep bass voicings wide
# At 500Hz: ~3 semitones (minor 3rd) — standard chord voicing works
# At 1kHz: ~2 semitones — close voicings OK
# At 4kHz: ~1.5 semitones — very close intervals are fine

def check_chord_voicing(notes_hz):
    """Check if chord voicing will sound clean or muddy based on critical bands."""
    issues = []
    for i in range(len(notes_hz)):
        for j in range(i+1, len(notes_hz)):
            lo, hi = sorted([notes_hz[i], notes_hz[j]])
            cbw = critical_bandwidth(lo)
            interval = hi - lo
            if interval < cbw * 0.5:
                issues.append(f"{lo:.0f}Hz and {hi:.0f}Hz too close "
                             f"(interval {interval:.0f}Hz < critical band {cbw:.0f}Hz) — will sound rough")
            elif interval < cbw:
                issues.append(f"{lo:.0f}Hz and {hi:.0f}Hz borderline "
                             f"(interval {interval:.0f}Hz ≈ critical band {cbw:.0f}Hz)")
    return issues
```

### Key Rules

- Below 200Hz: only root + octave (or root + 5th at most). No 3rds in the bass!
- 200-500Hz: open voicings (spread notes across 1+ octave)
- Above 500Hz: close voicings are fine (3rds, clusters)
- Pad voicings: spread across 1.5-2 octaves for richness without mud
- Mixing: two instruments in the same critical band will mask each other — EQ one of them out of that band

---

## 3. Auditory Masking

When a loud sound and a quiet sound share frequency content, the loud one "masks" (hides) the quiet one. This happens in both frequency and time.

### Simultaneous Masking (Frequency Domain)

```python
def masking_threshold(masker_freq_hz, masker_level_db, target_freq_hz):
    """Estimate masked threshold at target_freq given a masker.
    Based on Zwicker spreading function.
    Returns: minimum level (dB) target needs to be heard above masker."""
    bark_diff = freq_to_bark(target_freq_hz) - freq_to_bark(masker_freq_hz)

    # Spreading function (asymmetric — masking spreads more upward in frequency)
    if bark_diff >= 0:
        # Upward masking (higher freqs masked by lower): -25 dB/Bark
        spread = -25 * bark_diff
    else:
        # Downward masking (lower freqs masked by higher): -10 dB/Bark (less)
        spread = 10 * bark_diff  # Note: bark_diff is negative, so this is -10 * |bark_diff|

    masked_threshold = masker_level_db + spread
    return max(masked_threshold, -100)  # Floor at -100 dB

# Key insight: LOW frequencies mask HIGH frequencies much more than vice versa
# This is why muddy bass ruins the entire mix — it masks everything above it
```

### Temporal Masking

```python
TEMPORAL_MASKING = {
    'forward_masking': {
        # After a loud sound stops, quiet sounds are masked for 100-200ms
        'duration_ms': 200,
        'decay': 'exponential',
        'initial_threshold_db': -20,  # 20dB below masker level
        'formula': 'threshold = masker_level - 20 * (1 - exp(-t / 0.050))',
    },
    'backward_masking': {
        # A loud sound masks quiet sounds that occurred 5-20ms BEFORE it
        'duration_ms': 20,
        'practical_use': 'Pre-transient sounds are hidden by the transient itself',
    },
    'production_implications': [
        'Reverb tails are masked by the next transient — short reverb OK even at high density',
        'Sidechain ducking exploits forward masking — pad returns during masked window',
        'Pre-delay on reverb exploits the gap between masking windows',
        'Compression attack time: let transient through (5-30ms) while body is masked by transient',
    ],
}
```

### Mix Unmasking Strategies

```python
UNMASKING_STRATEGIES = {
    'frequency_separation': {
        'technique': 'EQ cut conflicting band in one instrument, boost in other',
        'example': 'Cut bass at 200-400Hz, boost vocal at 200-400Hz',
        'effectiveness': 'High — reduces simultaneous masking directly',
    },
    'temporal_separation': {
        'technique': 'Sidechain, gating, or rhythmic offset',
        'example': 'Duck pad when vocal plays, or offset rhythms so they alternate',
        'effectiveness': 'High — eliminates simultaneous masking entirely',
    },
    'spatial_separation': {
        'technique': 'Pan conflicting elements to different positions',
        'example': 'Guitar left, piano right',
        'effectiveness': 'Moderate — binaural unmasking gives ~3-6dB release',
    },
    'spectral_contrast': {
        'technique': 'Use different timbres (saw vs sine, bright vs dark)',
        'example': 'Bright pluck melody over dark pad — different spectral envelopes',
        'effectiveness': 'Moderate — helps even at same frequency',
    },
}
```

---

## 4. Consonance & Dissonance

### Plomp & Levelt Dissonance Curve (1965)

Two pure tones sound maximally dissonant when they're about 25% of a critical bandwidth apart (quarter-tone interval). As they move further apart, dissonance drops rapidly.

```python
def plomp_levelt_dissonance(freq1, freq2):
    """Dissonance between two pure tones (Plomp & Levelt 1965).
    Returns 0 (consonant) to 1 (maximally dissonant)."""
    if freq1 > freq2:
        freq1, freq2 = freq2, freq1
    s = 0.24 / (0.021 * freq1 + 19)  # Critical bandwidth scaling
    diff = freq2 - freq1
    # Dissonance peaks at ~25% of critical bandwidth, decays exponentially
    d = np.exp(-3.5 * s * diff) - np.exp(-5.75 * s * diff)
    return max(0, d)

def roughness_complex(freqs, amps):
    """Compute roughness (Sethares model) for a set of partials.
    freqs: array of partial frequencies. amps: array of amplitudes.
    Lower = more consonant. Use for comparing voicings or intervals."""
    roughness = 0.0
    for i in range(len(freqs)):
        for j in range(i+1, len(freqs)):
            roughness += amps[i] * amps[j] * plomp_levelt_dissonance(freqs[i], freqs[j])
    return roughness
```

### Consonance Ranking of Musical Intervals

```python
# Intervals ranked by consonance (complex tones with harmonics, not pure tones)
INTERVAL_CONSONANCE = {
    # semitones: (consonance_1_to_10, name, frequency_ratio)
    0:  (10, 'unison',        '1:1'),
    12: (9,  'octave',        '2:1'),
    7:  (8,  'perfect 5th',   '3:2'),
    5:  (7,  'perfect 4th',   '4:3'),
    4:  (6,  'major 3rd',     '5:4'),
    3:  (5,  'minor 3rd',     '6:5'),
    9:  (5,  'major 6th',     '5:3'),
    8:  (5,  'minor 6th',     '8:5'),
    2:  (3,  'major 2nd',     '9:8'),
    10: (3,  'minor 7th',     '9:5'),
    11: (2,  'major 7th',     '15:8'),
    1:  (1,  'minor 2nd',     '16:15'),
    6:  (1,  'tritone',       '45:32'),
}
# Production use: prefer consonant intervals for sustained sounds (pads, drones)
# Dissonant intervals are fine for short durations (passing tones, transients)
```

### Practical Chord Dissonance Checker

```python
def chord_roughness(midi_notes, n_harmonics=6):
    """Compute perceptual roughness of a chord, including harmonics.
    Use to compare voicings — lower roughness = cleaner sound."""
    freqs = []
    amps = []
    for note in midi_notes:
        f0 = 440 * 2 ** ((note - 69) / 12)
        for h in range(1, n_harmonics + 1):
            freqs.append(f0 * h)
            amps.append(1.0 / h)  # Natural harmonic rolloff
    return roughness_complex(np.array(freqs), np.array(amps))

# Usage: compare two voicings of Cmaj7
# Close voicing: [60, 64, 67, 71] (C4 E4 G4 B4) → roughness ~0.15
# Spread voicing: [48, 64, 67, 71] (C3 E4 G4 B4) → roughness ~0.08 (less muddy)
```

---

## 5. The Missing Fundamental (Virtual Pitch)

The brain can reconstruct a pitch from its harmonics alone, even if the fundamental frequency is absent. This is why you "hear" bass on small phone speakers.

### How It Works

If harmonics 2f, 3f, 4f, 5f are present, the brain perceives pitch f even though f itself is missing. The brain extracts the common fundamental from the harmonic series.

```python
def psychoacoustic_bass_layer(sig, fundamental_hz, sr=SR):
    """Add harmonics of a bass note to create perceived low-end on small speakers.
    The harmonics (2f, 3f, 4f) are in a higher frequency range that small speakers
    can reproduce, but the brain perceives the fundamental."""
    n = len(sig)
    t = np.arange(n) / sr

    # Generate harmonics 2-5 of the fundamental
    harmonics = np.zeros(n)
    for h in range(2, 6):
        freq = fundamental_hz * h
        if freq < sr / 2:  # Stay below Nyquist
            amp = 0.3 / h  # Decreasing amplitude for higher harmonics
            harmonics += amp * np.sin(2 * np.pi * freq * t)

    return sig + harmonics

def bass_enhancer(sig, crossover=120, harmonics_mix=0.3, sr=SR):
    """MaxxBass-style psychoacoustic bass enhancement.
    Extracts bass below crossover, generates harmonics, blends above crossover.
    Result: perceived bass on systems that can't reproduce sub-frequencies."""
    # Isolate bass
    sos_lo = butter(4, crossover, btype='low', fs=sr, output='sos')
    sos_hi = butter(4, crossover, btype='high', fs=sr, output='sos')
    bass = sosfilt(sos_lo, sig)
    highs = sosfilt(sos_hi, sig)

    # Generate harmonics of the bass via gentle saturation
    # Saturation creates harmonics at 2x, 3x, 4x the input frequency
    bass_harmonics = np.tanh(bass * 3.0) - bass  # Extract only the new harmonics
    # HPF the harmonics to remove the original fundamental
    sos_hp = butter(2, crossover, btype='high', fs=sr, output='sos')
    bass_harmonics = sosfilt(sos_hp, bass_harmonics)

    # Blend: original bass + harmonics (harmonics carry to small speakers)
    return bass + highs + bass_harmonics * harmonics_mix
```

### Optimal Harmonic Range for Virtual Pitch

```python
VIRTUAL_PITCH_RANGES = {
    # fundamental_hz: best harmonics to add for perceived bass
    30:  {'harmonics': [2,3,4,5,6], 'sweet_spot': '60-180Hz'},   # Sub bass
    40:  {'harmonics': [2,3,4,5],   'sweet_spot': '80-200Hz'},   # Deep bass
    60:  {'harmonics': [2,3,4],     'sweet_spot': '120-240Hz'},  # Bass guitar
    80:  {'harmonics': [2,3],       'sweet_spot': '160-240Hz'},  # Low E guitar
    100: {'harmonics': [2,3],       'sweet_spot': '200-300Hz'},  # Upper bass
}
# The sweet spot is the frequency range where the harmonics create the strongest
# virtual pitch perception. These frequencies are reproducible on most speakers.
```

### Production Rules

- Always layer bass: sub sine (30-60Hz) + harmonics (100-300Hz) + click (2-5kHz)
- On small speakers: the sub disappears but harmonics carry the perceived pitch
- 808 processing: add subtle saturation to generate harmonics above the fundamental
- Multiband saturation on bass: saturate only 80-300Hz band to add harmonics without distorting the sub

---

## 6. Frequency-Emotion Mapping

Different frequency ranges trigger different perceptual and emotional responses:

```python
FREQUENCY_EMOTION_MAP = {
    'sub_bass':   {'range': (20, 60),    'feeling': 'Physical power, rumble, pressure',
                   'emotional': 'Awe, dread, weight, impact'},
    'bass':       {'range': (60, 200),   'feeling': 'Warmth, fullness, body',
                   'emotional': 'Comfort, grounding, intimacy'},
    'low_mids':   {'range': (200, 500),  'feeling': 'Body, thickness',
                   'emotional': 'Warmth if controlled, mud/boxy if excess'},
    'mids':       {'range': (500, 2000), 'feeling': 'Presence, clarity',
                   'emotional': 'Human voice range — natural, honest, direct'},
    'upper_mids': {'range': (2000, 5000),'feeling': 'Presence, edge, bite',
                   'emotional': 'Aggression, urgency, clarity (ear most sensitive here)'},
    'presence':   {'range': (5000, 8000),'feeling': 'Definition, sibilance',
                   'emotional': 'Crispness, excitement, can fatigue if overcooked'},
    'air':        {'range': (8000,16000),'feeling': 'Shimmer, sparkle, breathiness',
                   'emotional': 'Openness, space, ethereal quality'},
}

# Mood EQ profiles — frequency adjustments for different emotional targets
MOOD_EQ_CURVES = {
    'warm': {
        'low_shelf':  (100, +2),
        'peak_cut':   (3000, -2, 1.0),
        'high_shelf': (10000, -1),
        'description': 'Boosted lows, reduced presence — cozy, intimate',
    },
    'cold': {
        'low_shelf':  (100, -2),
        'peak_boost': (5000, +2, 0.8),
        'high_shelf': (12000, +2),
        'description': 'Reduced warmth, boosted highs — clinical, distant',
    },
    'dark': {
        'low_shelf':  (80, +3),
        'peak_cut':   (4000, -4, 0.7),
        'high_shelf': (8000, -4),
        'description': 'Heavy lows, suppressed highs — mysterious, brooding',
    },
    'bright': {
        'low_shelf':  (100, -1),
        'peak_boost': (3000, +2, 0.8),
        'high_shelf': (10000, +3),
        'description': 'Reduced mud, boosted presence and air — energetic, uplifting',
    },
    'aggressive': {
        'peak_boost': (200, +2, 1.5),
        'peak_boost2':(3500, +3, 1.0),
        'peak_cut':   (500, -2, 1.5),
        'description': 'Boosted body and bite, scooped mids — powerful, intense',
    },
    'intimate': {
        'low_shelf':  (100, +1),
        'peak_boost': (800, +1, 0.5),
        'high_shelf': (12000, -2),
        'description': 'Subtle warmth, proximity, rolled-off air — whispered, close',
    },
    'ethereal': {
        'low_shelf':  (80, -3),
        'peak_cut':   (500, -2, 1.0),
        'high_shelf': (8000, +4),
        'description': 'Minimal bass, boosted air — floating, otherworldly',
    },
    'vintage': {
        'high_shelf': (6000, -3),
        'low_shelf':  (60, -2),
        'peak_boost': (800, +2, 0.5),
        'description': 'Rolled-off extremes, mid-focused — nostalgic, analog',
    },
    'lo_fi': {
        'high_shelf': (4000, -6),
        'low_shelf':  (60, -3),
        'peak_boost': (600, +3, 0.5),
        'description': 'Heavily rolled-off, mid-heavy — degraded, nostalgic',
    },
}

def apply_mood_eq(sig, mood, sr=SR):
    """Apply frequency profile for a target emotional mood."""
    curve = MOOD_EQ_CURVES[mood]
    for key, params in curve.items():
        if key == 'description':
            continue
        if key.startswith('low_shelf'):
            sig = low_shelf(sig, params[0], params[1], sr=sr)
        elif key.startswith('high_shelf'):
            sig = high_shelf(sig, params[0], params[1], sr=sr)
        elif 'peak' in key:
            sig = peak_eq(sig, params[0], params[1], Q=params[2], sr=sr)
    return sig
```

### Ear Canal Resonance

The ear canal resonates at 2-5kHz (primarily ~3.5kHz), amplifying those frequencies by 10-15dB. This is why:
- Vocals are most intelligible in this range
- Sibilance (s, t, sh) is so prominent
- Harsh synths at 3-5kHz cause fatigue
- Presence boosts are so effective (you're amplifying what the ear already amplifies)

---

## 7. Temporal Perception

### Key Thresholds

```python
TEMPORAL_THRESHOLDS = {
    'gap_detection':        2,      # ms — minimum silence we can detect
    'click_threshold':      0.5,    # ms — shorter transient = heard as click, not tone
    'pitch_detection_min':  10,     # ms — minimum duration to perceive pitch
    'timbre_detection_min': 50,     # ms — minimum for full timbral recognition
    'echo_threshold':       50,     # ms — delays > 50ms heard as distinct echo
    'haas_window':          35,     # ms — delays < 35ms fuse into single image
    'precedence_effect':    (1, 35),# ms — first-arriving sound dominates localization
    'rhythmic_grouping':    (100, 1800),  # ms IOI — outside this range, no beat
    'tempo_preference':     500,    # ms IOI = 120 BPM — "natural" tempo
    'flutter_echo':         (50, 100),    # ms — repeated echoes perceived as flutter
}
```

### Haas Effect for Stereo Width

```python
def haas_widener(mono_sig, delay_ms=12, level_db=-3, sr=SR):
    """Create stereo width from mono using the Haas (precedence) effect.
    delay_ms: 1-35ms (>35ms = distinct echo). 8-15ms = natural room width.
    level_db: delayed channel level relative to direct (-3 to 0 dB)."""
    delay_samples = int(delay_ms * sr / 1000)
    level = 10 ** (level_db / 20)

    left = mono_sig.copy()
    right = np.zeros_like(mono_sig)
    if delay_samples < len(mono_sig):
        right[delay_samples:] = mono_sig[:-delay_samples] * level

    return left, right

# For stereo sources, apply different delays to L and R:
def stereo_haas_enhance(left, right, delay_ms=8, sr=SR):
    """Widen an existing stereo signal using cross-delayed Haas."""
    delay_samples = int(delay_ms * sr / 1000)
    # Add delayed opposite channel (very quiet) to each side
    left_out = left.copy()
    right_out = right.copy()
    if delay_samples < len(left):
        left_out[delay_samples:] += right[:-delay_samples] * 0.15
        right_out[delay_samples:] += left[:-delay_samples] * 0.15
    return left_out, right_out
```

### ITD/ILD for Spatial Positioning

```python
def position_in_space(mono_sig, azimuth_deg, distance=1.0, sr=SR):
    """Position a mono sound in stereo space using ITD and ILD.
    azimuth_deg: -90 (full left) to +90 (full right), 0 = center.
    distance: relative distance (affects reverb amount, not implemented here)."""
    # Interaural Time Difference (max ~0.6ms at 90 degrees)
    max_itd_ms = 0.6
    itd_ms = max_itd_ms * np.sin(np.radians(azimuth_deg))
    itd_samples = int(abs(itd_ms) * sr / 1000)

    # Interaural Level Difference (up to ~10dB at high frequencies)
    # Head shadow mainly affects frequencies > 1500Hz
    ild_db = 10 * np.sin(np.radians(azimuth_deg))  # Simplified

    # Apply
    if azimuth_deg >= 0:  # Sound to the right
        left = np.zeros_like(mono_sig)
        right = mono_sig.copy()
        if itd_samples < len(mono_sig):
            left[itd_samples:] = mono_sig[:-itd_samples] * 10 ** (-abs(ild_db) / 20)
        right *= 10 ** (abs(ild_db) / 40)  # Slight boost to near ear
    else:  # Sound to the left
        right = np.zeros_like(mono_sig)
        left = mono_sig.copy()
        if itd_samples < len(mono_sig):
            right[itd_samples:] = mono_sig[:-itd_samples] * 10 ** (-abs(ild_db) / 20)
        left *= 10 ** (abs(ild_db) / 40)

    return left, right
```

---

## 8. LUFS Metering (ITU-R BS.1770-4)

The industry standard for perceived loudness measurement.

### K-Weighting Filter

LUFS uses a two-stage filter: shelf boost at high frequencies (head/torso model) + highpass at low frequencies.

```python
def k_weight_filter(sr=SR):
    """ITU-R BS.1770-4 K-weighting filter (two cascaded biquads).
    Stage 1: High shelf (+4dB at ~1.5kHz) — models head/ear response.
    Stage 2: High-pass at ~38Hz — removes sub-bass from loudness calculation."""
    # Stage 1: Shelving filter (pre-filter)
    f0 = 1681.974
    Q = 0.7071752
    K = np.tan(np.pi * f0 / sr)
    Vh = 10 ** (4 / 20)  # +4dB
    Vb = Vh ** 0.4996667
    a0 = 1 + K/Q + K*K
    b_shelf = np.array([
        (Vh + Vb*K/Q + K*K) / a0,
        2 * (K*K - Vh) / a0,
        (Vh - Vb*K/Q + K*K) / a0
    ])
    a_shelf = np.array([1.0, 2*(K*K - 1)/a0, (1 - K/Q + K*K)/a0])

    # Stage 2: High-pass at 38Hz
    f0_hp = 38.13547
    Q_hp = 0.5003270
    K_hp = np.tan(np.pi * f0_hp / sr)
    a0_hp = 1 + K_hp/Q_hp + K_hp*K_hp
    b_hp = np.array([1/a0_hp, -2/a0_hp, 1/a0_hp])
    a_hp = np.array([1.0, 2*(K_hp*K_hp - 1)/a0_hp, (1 - K_hp/Q_hp + K_hp*K_hp)/a0_hp])

    # Cascade into SOS format
    sos = np.array([
        [b_shelf[0], b_shelf[1], b_shelf[2], 1.0, a_shelf[1], a_shelf[2]],
        [b_hp[0], b_hp[1], b_hp[2], 1.0, a_hp[1], a_hp[2]],
    ])
    return sos

def measure_lufs(left, right, sr=SR):
    """Measure integrated LUFS (ITU-R BS.1770-4) with gating."""
    sos = k_weight_filter(sr)
    # Apply K-weighting
    l_weighted = sosfilt(sos, left)
    r_weighted = sosfilt(sos, right)

    # Mean square per channel (sum with channel weights: L=R=1.0)
    block_size = int(0.4 * sr)   # 400ms blocks
    hop = int(0.1 * sr)          # 75% overlap (100ms hop)

    block_loudness = []
    for i in range(0, len(l_weighted) - block_size, hop):
        l_block = l_weighted[i:i+block_size]
        r_block = r_weighted[i:i+block_size]
        ms = np.mean(l_block2) + np.mean(r_block2)
        if ms > 0:
            block_loudness.append(-0.691 + 10 * np.log10(ms))

    if not block_loudness:
        return -70.0

    # Gating: absolute threshold at -70 LUFS
    above_abs = [b for b in block_loudness if b > -70]
    if not above_abs:
        return -70.0

    # Relative threshold: -10 dB below ungated mean
    ungated_mean = -0.691 + 10 * np.log10(np.mean([10**((b+0.691)/10) for b in above_abs]))
    relative_threshold = ungated_mean - 10

    # Final gated measurement
    gated = [b for b in above_abs if b > relative_threshold]
    if not gated:
        return ungated_mean

    integrated = -0.691 + 10 * np.log10(np.mean([10**((b+0.691)/10) for b in gated]))
    return integrated

def normalize_to_lufs(left, right, target_lufs=-14, sr=SR):
    """Normalize stereo signal to target LUFS (e.g., -14 for Spotify)."""
    current = measure_lufs(left, right, sr)
    gain_db = target_lufs - current
    gain_lin = 10 ** (gain_db / 20)
    return left * gain_lin, right * gain_lin
```

### Platform Targets

```python
LUFS_TARGETS = {
    'spotify':      -14,
    'apple_music':  -16,
    'youtube':      -14,
    'tidal':        -14,
    'broadcast':    -23,  # EBU R128
    'podcast':      -16,
    'club_music':   -8,   # For DJ playback (not streaming)
}
```

### Genre-Specific Dynamic Range

```python
GENRE_DYNAMICS = {
    # genre: (target_lufs, crest_factor_db, short_term_range_LU)
    'classical':  (-18, 16, 12),   # Wide dynamics
    'jazz':       (-16, 12, 8),
    'folk':       (-15, 10, 6),
    'pop':        (-14, 8, 5),
    'rock':       (-12, 8, 4),
    'edm':        (-10, 6, 3),
    'hip_hop':    (-10, 7, 4),
    'lo_fi':      (-14, 10, 6),
    'ambient':    (-18, 14, 10),
}
```

---

## 9. Harmonic Series & Timbre Perception

### Why Different Waveforms Sound Different

All pitched sounds are sums of sine waves at integer multiples of the fundamental (harmonics). The relative amplitudes of these harmonics determine timbre.

```python
HARMONIC_CHARACTERS = {
    'pure_sine':    {'harmonics': [1.0], 'character': 'Hollow, empty, whistle-like'},
    'triangle':     {'harmonics': [1.0, 0, 1/9, 0, 1/25, 0, 1/49],
                     'character': 'Soft, mellow, recorder-like (odd harmonics, rapid falloff)'},
    'square':       {'harmonics': [1.0, 0, 1/3, 0, 1/5, 0, 1/7, 0, 1/9],
                     'character': 'Hollow, woody, clarinet-like (odd harmonics only)'},
    'sawtooth':     {'harmonics': [1.0, 1/2, 1/3, 1/4, 1/5, 1/6, 1/7, 1/8],
                     'character': 'Bright, buzzy, brassy (all harmonics, 1/n falloff)'},
    'warm_pad':     {'harmonics': [1.0, 0.5, 0.15, 0.05],
                     'character': 'Warm, rounded (fast harmonic decay)'},
    'bright_lead':  {'harmonics': [1.0, 0.8, 0.6, 0.5, 0.4, 0.3],
                     'character': 'Bright, present (slow harmonic decay)'},
}
```

### Even vs Odd Harmonics

```python
HARMONIC_TYPES = {
    'even_harmonics': {
        'partials': [2, 4, 6, 8],
        'intervals': ['octave', '2 octaves', 'octave+5th', '3 octaves'],
        'character': 'Warm, rich, musical, "tube-like"',
        'source': 'Tube/valve amps, tape saturation, asymmetric clipping',
        'perception': 'Consonant (octave relations) — adds "warmth" and "body"',
    },
    'odd_harmonics': {
        'partials': [3, 5, 7, 9],
        'intervals': ['octave+5th', '2 oct+M3', '2 oct+m7', '3 oct+M2'],
        'character': 'Edgy, hollow, aggressive, "transistor-like"',
        'source': 'Transistor clipping, square wave, symmetric distortion',
        'perception': 'More dissonant (7th and 9th especially) — adds "grit" and "edge"',
    },
}
# Production rule: use even-harmonic saturation for warmth (tube, tape),
# odd-harmonic distortion for aggression (transistor, fuzz)
```

### Inharmonicity

Real instruments don't have perfectly integer harmonics. Slight detuning (inharmonicity) affects timbre perception:

```python
INHARMONICITY = {
    # B = inharmonicity coefficient; partial_n_freq = n * f0 * sqrt(1 + B * n^2)
    'tuning_fork':  0.0,      # Perfect harmonics
    'nylon_guitar': 0.00005,  # Barely inharmonic
    'steel_guitar': 0.0002,   # Slightly bright
    'piano_bass':   0.001,    # Noticeably inharmonic (low strings)
    'piano_treble': 0.003,    # More inharmonic (short, stiff strings)
    'bell':         0.02,     # Strongly inharmonic (metallic)
    'bar_metal':    0.05,     # Very inharmonic (gong, cymbal)

    # Perception thresholds:
    'just_noticeable': 0.0001,  # Below this, sounds "pure"
    'warm_zone':      (0.0001, 0.001),  # Slight inharmonicity = warmth
    'metallic_zone':  (0.001, 0.01),    # Clearly metallic character
    'bell_zone':      (0.01, 0.05),     # Distinctly bell/chime-like
}

def inharmonic_partials(f0, n_partials, B):
    """Generate inharmonic partial frequencies.
    B: inharmonicity coefficient (see table above)."""
    partials = []
    for n in range(1, n_partials + 1):
        fn = n * f0 * np.sqrt(1 + B * n * n)
        partials.append(fn)
    return partials
```

---

## 10. Psychoacoustic Bass Enhancement

### Subharmonic Synthesis

Generate a subharmonic (f/2) from the bass signal using frequency division:

```python
def subharmonic_synth(sig, crossover=100, sub_mix=0.3, sr=SR):
    """Generate subharmonic (one octave below) from bass content.
    Useful for adding sub-bass weight to thin bass sounds."""
    # Isolate bass
    sos = butter(4, crossover, btype='low', fs=sr, output='sos')
    bass = sosfilt(sos, sig)

    # Rectify to double frequency, then filter to get fundamental back
    # Actually: to get f/2, use zero-crossing detection
    # Simpler approach: ring modulate bass with sub-octave
    # Track envelope and generate sub tone
    env = np.abs(bass)
    smooth = int(0.01 * sr)
    env = np.convolve(env, np.ones(smooth)/smooth, mode='same')

    # Generate sub-oscillator tracking the bass pitch
    # Simplified: use the rectified signal filtered to sub range
    sub = np.sign(bass)  # Square wave at bass frequency
    sub_sos = butter(4, crossover / 2, btype='low', fs=sr, output='sos')
    sub = sosfilt(sub_sos, sub)  # Filter to sub-bass range
    sub *= env  # Apply original envelope

    return sig + sub * sub_mix

def harmonic_bass_exciter(sig, fundamental_range=(30, 80), n_harmonics=4,
                           harmonics_mix=0.25, sr=SR):
    """Generate upper harmonics from bass to enhance perceived low-end.
    Works on any speaker — harmonics are in the 100-400Hz range."""
    # Isolate bass fundamentals
    sos = butter(4, fundamental_range[1], btype='low', fs=sr, output='sos')
    bass = sosfilt(sos, sig)

    # Generate harmonics via polynomial saturation
    # Each power of x generates a specific harmonic
    harmonics = np.zeros_like(bass)
    for n in range(2, n_harmonics + 2):
        # nth power generates nth harmonic (Chebyshev relationship)
        h = bass ** n
        # Bandpass to isolate just this harmonic range
        lo = fundamental_range[0] * n
        hi = min(fundamental_range[1] * n, sr / 2 - 100)
        if lo < hi:
            sos_bp = butter(2, [lo, hi], btype='band', fs=sr, output='sos')
            harmonics += sosfilt(sos_bp, h) / n  # Decreasing amplitude

    # Normalize harmonics
    peak = np.max(np.abs(harmonics))
    if peak > 0:
        harmonics *= np.max(np.abs(bass)) / peak

    return sig + harmonics * harmonics_mix
```

### Multiband Bass Enhancement

```python
def multiband_bass_enhance(sig, sr=SR):
    """Professional bass enhancement: sub + harmonics + top-end click.
    Three complementary techniques for full bass perception on any system."""
    # Band 1: Sub-bass (30-60Hz) — subharmonic synthesis for physical weight
    sos_sub = butter(4, 60, btype='low', fs=sr, output='sos')
    sub = sosfilt(sos_sub, sig)
    sub_enhanced = np.tanh(sub * 1.5)  # Gentle saturation for density

    # Band 2: Upper bass (60-200Hz) — harmonic excitement for warmth
    sos_lo = butter(4, [60, 200], btype='band', fs=sr, output='sos')
    upper_bass = sosfilt(sos_lo, sig)
    # Add 2nd and 3rd harmonics
    harmonics = np.tanh(upper_bass * 2.0) - upper_bass
    sos_harm = butter(2, [120, 600], btype='band', fs=sr, output='sos')
    harmonics = sosfilt(sos_harm, harmonics)

    # Band 3: Click/presence (2-5kHz) — transient excitement for definition
    sos_click = butter(2, [2000, 5000], btype='band', fs=sr, output='sos')
    click = sosfilt(sos_click, sig)
    # Enhance transients in this range
    click_env = np.abs(click)
    click_smooth = np.convolve(click_env, np.ones(int(0.002*sr))/int(0.002*sr), mode='same')
    click_enhanced = click * (1 + click_smooth * 2)

    # Recombine
    sos_mid = butter(4, [200, 2000], btype='band', fs=sr, output='sos')
    mids = sosfilt(sos_mid, sig)
    sos_hi = butter(4, 5000, btype='high', fs=sr, output='sos')
    highs = sosfilt(sos_hi, sig)

    return sub_enhanced + upper_bass + harmonics * 0.2 + mids + click_enhanced + highs
```

### Playback System Decision Matrix

```python
BASS_STRATEGY = {
    # target_system: processing approach
    'club_pa':       {'sub': True,  'harmonics': False, 'enhancer': False,
                      'note': 'Full sub-bass reproduction, no enhancement needed'},
    'studio_monitors':{'sub': True, 'harmonics': False, 'enhancer': False,
                      'note': 'Accurate reproduction'},
    'headphones':    {'sub': True,  'harmonics': True,  'enhancer': False,
                      'note': 'Sub perceived but add harmonics for impact'},
    'car_speakers':  {'sub': False, 'harmonics': True,  'enhancer': True,
                      'note': 'Limited sub, rely on harmonics + exciter'},
    'laptop':        {'sub': False, 'harmonics': True,  'enhancer': True,
                      'note': 'No sub capability, harmonics essential'},
    'phone':         {'sub': False, 'harmonics': True,  'enhancer': True,
                      'note': 'Worst case — psychoacoustic bass only'},
}
# Best practice: always include both real sub AND harmonics in the mix,
# so the music translates well to any playback system
```

---

## Quick Reference: Key Psychoacoustic Numbers

| Parameter | Value | Application |
|-----------|-------|-------------|
| Peak hearing sensitivity | 2-5kHz | Be careful with boosts here |
| Ear canal resonance | ~3.5kHz (+12dB) | Why presence range is so powerful |
| Critical bandwidth at 100Hz | ~90Hz | Keep bass voicings WIDE |
| Critical bandwidth at 1kHz | ~130Hz | Close chord voicings OK here |
| Roughness maximum | 25% of critical bandwidth | Maximum beating/dissonance |
| Gap detection threshold | 2ms | Minimum for audible silence |
| Echo threshold | 50ms | Delays >50ms heard as echo |
| Haas fusion window | 1-35ms | Delays <35ms fuse to one image |
| Forward masking duration | ~200ms | Quiet sounds hidden after loud ones |
| Minimum pitch perception | 10ms | Shorter = click, not tone |
| Streaming LUFS target | -14 LUFS | Spotify, YouTube, Tidal |
| True peak limit | -1 dBTP | Prevents intersample clipping |
| Even harmonics source | Tube/tape saturation | Adds warmth |
| Odd harmonics source | Transistor/symmetric clip | Adds grit/edge |
| Virtual pitch minimum | 2nd harmonic needed | Brain infers f from 2f,3f,4f |
| Inharmonicity threshold | B > 0.0001 | Below = pure, above = metallic |
| Walking pace | 100-130 BPM | Why 120 BPM feels "natural" |
