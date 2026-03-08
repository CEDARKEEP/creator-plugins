# Mixing, EQ & Arrangement

Stereo mix pipeline, panning, sidechain, EQ carving, arrangement patterns, numpy master chain, and memory management.

## Stereo Mix Pipeline

### 1. Per-Track Processing

Each instrument track should be processed individually before mixing:

```python
# For each track:
track = instrument_builder(...)              # Generate audio
track = compressor(track, ...)               # Per-track compression
track = lowpass(track, cutoff)               # Filter harshness
track = freeverb(track, ...) if not bass     # Reverb (NOT on bass/kick)
left, right = pan_mono(track, pan_position)  # Stereo placement
```

### 2. Panning Positions

| Instrument | Pan (0=L, 0.5=C, 1=R) | Notes |
|-----------|------------------------|-------|
| Kick | 0.50 (center) | Always center |
| Bass | 0.50 (center) | Always center |
| Sub bass | 0.50 (center) | Always center |
| Snare | 0.48-0.52 | Nearly center |
| Lead/Melody | 0.45-0.55 | Near center |
| Hi-hat | 0.35-0.45 | Slightly left |
| Ride | 0.55-0.65 | Slightly right |
| Pad L | 0.25-0.35 | Left |
| Pad R | 0.65-0.75 | Right |
| Arp | 0.30-0.70 | Varies |
| Strings L | 0.20-0.30 | Left |
| Strings R | 0.70-0.80 | Right |
| FX/Perc | 0.15-0.85 | Wide spread |

**Rule: keep everything below 150Hz centered** (sub bass, kick fundamental). Wide stereo only for mid and high frequencies.

### 3. Multi-Dimensional Section Automation

Use 5-dimension energy maps (see [energy-and-engagement.md](energy-and-engagement.md)) for per-bar automation. Each dimension controls different mix parameters:

```python
def smoothstep(t):
    """Hermite smoothstep interpolation (0→1)."""
    t = np.clip(t, 0, 1)
    return t * t * (3 - 2 * t)

def build_bar_energy(sections):
    """Expand section energy into per-bar lookup for all 5 dimensions."""
    bar_energy = []
    for section in sections:
        e = section['energy']  # dict: intensity, density, rhythm, harmonic, brightness
        for _ in range(section['bars']):
            bar_energy.append(dict(e))  # copy per bar
    return bar_energy

def apply_section_automation(signal, bar, bar_energy, bar_samples, track_name, sr=SR):
    """Apply multi-dimensional energy automation to a track for one bar."""
    e = bar_energy[bar]
    n_bars = len(bar_energy)

    # INTENSITY → volume curve with smooth transitions
    current_vol = 0.3 + 0.7 * (e['intensity'] / 10.0)
    next_bar = min(bar + 1, n_bars - 1)
    next_vol = 0.3 + 0.7 * (bar_energy[next_bar]['intensity'] / 10.0)

    n = len(signal)
    # Apply volume with smooth transition in last 10% of bar
    fade_start = int(n * 0.9)
    vol_env = np.full(n, current_vol)
    for i in range(fade_start, n):
        t = (i - fade_start) / (n - fade_start)
        vol_env[i] = current_vol + (next_vol - current_vol) * smoothstep(t)
    signal = signal * vol_env

    # BRIGHTNESS → dynamic lowpass filter
    cutoff = 800 + (e['brightness'] / 10.0) * 12000  # 800Hz to 12.8kHz
    if cutoff < 12000:  # Only filter if not already bright
        sos = butter(2, min(cutoff / (sr / 2), 0.99), btype='low', output='sos')
        signal = sosfilt(sos, signal)

    return signal

# DENSITY → track gating (in the main mix loop)
TRACK_DENSITY_THRESHOLD = {
    'kick': 1, 'bass': 1, 'pad': 1,
    'hihat': 3, 'snare': 3, 'chord': 3,
    'melody': 5, 'arp': 5, 'ride': 5,
    'strings': 7, 'lead': 7, 'counter_mel': 7, 'perc': 7,
    'fx': 9, 'doubled': 9, 'harmony': 9,
}

def is_track_active(track_name, bar, bar_energy):
    """Check if track should sound in this bar based on density."""
    threshold = TRACK_DENSITY_THRESHOLD.get(track_name, 5)
    return bar_energy[bar]['density'] >= threshold

# RHYTHM → pattern selection (in drum builders)
def select_drum_pattern(bar, bar_energy):
    """Select pattern complexity based on rhythm energy."""
    r = bar_energy[bar]['rhythm']
    if r <= 3:   return 'basic'       # kick + hat only
    elif r <= 6: return 'standard'    # kick + snare + hat + ride
    elif r <= 8: return 'syncopated'  # + ghost notes + syncopation
    else:        return 'complex'     # + fills + rolls + polyrhythm

# HARMONIC → chord voicing depth (in chord builders)
def select_chord_voicing(bar, bar_energy):
    """Select chord richness based on harmonic energy."""
    h = bar_energy[bar]['harmonic']
    if h <= 3:   return 'triad'    # Root + 3rd + 5th
    elif h <= 6: return '7th'      # + 7th, inversions
    elif h <= 8: return '9th'      # + 9th, spread voicings
    else:        return 'extended'  # + 11th/13th, altered, chromatic passing
```

### Engagement Elements (Micro-Tension in the Mix)

Add these automatically based on bar position to keep listeners engaged:

```python
def add_engagement_elements(mix_left, mix_right, bar, bar_energy, bar_samples, sr=SR):
    """Add micro-tension elements at phrase boundaries."""
    start = int(bar * bar_samples)
    end = min(int((bar + 1) * bar_samples), len(mix_left))

    # Drum fill every 4-8 bars (probability increases with rhythm energy)
    if bar % 4 == 3:
        fill_prob = 0.05 + 0.25 * (bar_energy[bar]['rhythm'] / 10.0)
        if np.random.random() < fill_prob:
            pass  # Insert fill from fill builder

    # Reverse crash before section transitions
    # (check if next bar has different section energy)
    if bar + 1 < len(bar_energy):
        intensity_jump = abs(bar_energy[bar+1]['intensity'] - bar_energy[bar]['intensity'])
        if intensity_jump >= 3:
            pass  # Insert reverse crash in last beat

    # Ear candy every 16 bars (one-shot FX, stereo moment, etc.)
    if bar % 16 == 15 and bar_energy[bar]['density'] >= 5:
        pass  # Insert subtle ear candy (reversed note, ping-pong delay burst, etc.)

    return mix_left, mix_right
```

### 4. Sidechain Processing

Apply sidechain compression to make kick punch through:

```python
# Sidechain the pad/chord bus to the kick
pad_signal = sidechain(pad_signal, kick_signal,
                        threshold_db=-20, ratio=10.0,
                        attack_ms=1, release_ms=150)
```

Typical sidechain targets: pads, chords, bass (in EDM), ambient layers. NOT on lead melody or drums.

## Numpy Master Chain (Fallback)

Use this when pedalboard is not available. Apply in this exact order:

```python
# 1. High-pass filter at 30Hz (remove sub-rumble)
sos = butter(4, 30, btype='high', fs=SR, output='sos')
mix = sosfilt(sos, mix)

# 2. Bus compression (gentle glue)
mix = compressor(mix, threshold_db=-12, ratio=2.0,
                  attack_ms=20, release_ms=200, knee_db=6)

# 3. Tape saturation (warmth)
mix = np.tanh(mix * 1.2) / np.tanh(1.2)  # normalized soft clip

# 4. Stereo width (widen mids/highs, keep bass centered)
if stereo:
    left, right = stereo_width(left, right, width=1.3)

# 5. Limiter at -1 dBFS (brickwall)
mix = limiter(mix, threshold_db=-1.0, release_ms=50)

# 6. Normalize to -1 dBFS
peak = np.max(np.abs(mix))
if peak > 0:
    mix *= (10 ** (-1/20)) / peak

# 7. Fade in/out
fade_in = int(SR * 0.5)   # 500ms fade in
fade_out = int(SR * 1.0)  # 1s fade out
mix[:fade_in] *= 0.5 - 0.5 * np.cos(np.linspace(0, np.pi, fade_in))
mix[-fade_out:] *= 0.5 + 0.5 * np.cos(np.linspace(0, np.pi, fade_out))

# 8. Final safety: soft-clip + 2ms cosine fade edges
mix = np.tanh(mix)
edge = int(0.002 * SR)
mix[:edge] *= 0.5 - 0.5 * np.cos(np.linspace(0, np.pi, edge))
mix[-edge:] *= 0.5 + 0.5 * np.cos(np.linspace(0, np.pi, edge))
```

## EQ Carving Reference

### Frequency Ranges

| Range | Frequencies | Character | Action |
|-------|-----------|-----------|--------|
| Sub bass | 20-60 Hz | Feel, rumble | HPF everything except kick/bass |
| Bass | 60-250 Hz | Warmth, body | Keep for bass/kick, cut on other tracks |
| Low mids | 250-500 Hz | Muddiness zone | Cut 3-6dB on most tracks to clean up |
| Mids | 500Hz-2kHz | Body, presence | Main voice of instruments |
| Upper mids | 2-4 kHz | Presence, ear peak | Boost for presence, cut if harsh |
| Treble | 4-8 kHz | Brightness, air | HH/cymbals live here |
| Air | 8-20 kHz | Sparkle, space | Gentle shelf boost for "air" |

### Per-Instrument EQ Guidelines

| Instrument | Cut | Boost | Notes |
|-----------|-----|-------|-------|
| Kick | HPF 30Hz, cut 300-500Hz | Boost 60-80Hz (sub), 3-5kHz (click) | Scoop mids for clarity |
| Snare | HPF 80Hz | Boost 200Hz (body), 2-4kHz (crack) | |
| Hi-hat | HPF 300-500Hz | | Remove low-end bleed |
| Bass | HPF 30Hz, cut 200-300Hz | Boost 60-100Hz, 800Hz-1kHz (presence) | |
| Pad | HPF 200Hz, LPF 8kHz | | Stay out of bass/treble extremes |
| Piano/Keys | HPF 80Hz | Boost 2-4kHz (presence) | |
| Vocals/Lead | HPF 80-100Hz | Boost 3-5kHz (presence), 10kHz (air) | |
| Reverb return | HPF 200-400Hz, LPF 6-10kHz | | ALWAYS EQ reverb returns |

### Detailed Per-Instrument EQ Cheat Sheet

```python
EQ_GUIDE = {
    'kick': {
        'sub':    (30, 60,    'boost 3-6dB for sub weight'),
        'punch':  (80, 200,   'body and punch zone'),
        'mud':    (250, 500,  'cut 3-6dB — boxiness lives here'),
        'click':  (2000, 4000,'boost 2-4dB for beater attack'),
        'air':    (6000, 10000,'gentle cut — not needed on kick'),
        'hpf': 30,
    },
    'snare': {
        'body':   (150, 400,  'fundamental resonance, boost 2-3dB'),
        'honk':   (400, 800,  'cut 2-4dB if boxy or ringy'),
        'crack':  (2000, 4000,'boost 2-4dB for snap and attack'),
        'wire':   (6000, 10000,'boost 1-2dB for sizzle and wire'),
        'hpf': 80,
    },
    'hihat': {
        'hpf':    (200, 400,  'aggressive HPF — no low content in hats'),
        'body':   (400, 1000, 'not needed — cut if present'),
        'brightness': (6000, 15000, 'main energy zone'),
        'harshness':  (3000, 5000,  'cut 2-3dB if piercing'),
    },
    'bass_guitar': {
        'sub':    (40, 80,    'fundamental weight'),
        'body':   (80, 200,   'warmth and fullness'),
        'mud':    (250, 500,  'cut 2-4dB for clarity, especially 200-220Hz'),
        'attack': (1200, 1500,'finger/pick definition — boost 2dB for note readability'),
        'presence':(2000, 4000,'string noise, grind — boost sparingly'),
        'hpf': 30,
    },
    'guitar_electric': {
        'body':   (80, 250,   'low-end fullness'),
        'mud':    (300, 500,  'cut 2-4dB — boxiness'),
        'honk':   (500, 1000, 'nasal zone — often cut 2-3dB'),
        'presence':(2000, 5000,'pick attack, bite — boost for solos'),
        'air':    (6000, 12000,'shimmer, open sound'),
        'hpf': 80,
    },
    'guitar_acoustic': {
        'body':   (80, 200,   'warmth, fullness'),
        'boom':   (200, 400,  'cut if close-miked or boomy'),
        'clarity':(2000, 5000,'string detail and pick definition'),
        'air':    (8000, 14000,'sparkle and breathiness'),
        'hpf': 80,
    },
    'piano': {
        'fullness':(80, 120,  'low register weight'),
        'body':   (200, 500,  'warm midrange'),
        'clarity':(1000, 3000,'note definition'),
        'brightness':(3000, 5000,'presence and sparkle'),
        'attack': (5000, 6000,'hammer attack — boost 1-2dB for definition'),
        'hpf': 60,
    },
    'vocals': {
        'hpf':    (80, 120,   'ALWAYS HPF vocals'),
        'body':   (200, 400,  'chest resonance, warmth — boost 1-2dB for thin vocals'),
        'mud':    (250, 500,  'cut 2-3dB if muffled or muddy'),
        'intelligibility': (800, 1500, 'diction clarity zone — critical for vocals cutting through'),
        'presence':(3000, 5000,'cut-through, forward sound — boost 2-4dB'),
        'sibilance':(5000, 8000,'de-ess zone — cut if harsh with narrow Q'),
        'air':    (10000, 16000,'boost 1-3dB shelf for breathiness and openness'),
    },
    'lead_synth': {
        'body':   (200, 800,  'thickness and weight'),
        'presence':(1000, 4000,'main energy and character'),
        'harshness':(3000, 5000,'tame 2-3dB if fatiguing'),
        'air':    (8000, 14000,'brightness and shimmer'),
        'hpf': 100,
    },
    'pad_synth': {
        'body':   (300, 800,  'core warmth'),
        'mud':    (200, 400,  'cut 2-4dB to avoid masking bass'),
        'lpf':    (6000, 10000,'ALWAYS LPF pads for warmth — keeps them in the background'),
        'hpf': 200,
    },
    'strings': {
        'body':   (200, 500,  'richness and warmth'),
        'nasal':  (800, 1500, 'cut 2-3dB if thin or nasal'),
        'presence':(2000, 5000,'bow articulation and detail'),
        'air':    (8000, 14000,'high harmonic shimmer — shelf boost 1-2dB'),
        'hpf': 80,
    },
    'brass': {
        'body':   (200, 500,  'fundamental warmth'),
        'honk':   (500, 1000, 'characteristic brass tone — careful with boosts'),
        'blare':  (1000, 3000,'cut 2-3dB if too aggressive'),
        'presence':(3000, 6000,'brilliance and cut-through'),
        'hpf': 80,
    },
    'saxophone': {
        'body':   (120, 240,  'fullness and warmth'),
        'harshness':(1000, 2000,'cut if shrill'),
        'reed':   (5000, 7000,'reed noise — cut for smooth, boost for gritty'),
        'hpf': 100,
    },
}
```

### Problem Frequency Zones

| Zone | Frequency | Problem | Fix |
|------|-----------|---------|-----|
| Mud | 250-500 Hz | Muddiness, boomy, unclear | Cut 3-6dB on most tracks except bass/kick body |
| Honk | 800-1500 Hz | Nasal, boxy, cheap sounding | Cut on guitars/vocals if needed, narrow Q |
| Fatigue | 3-5 kHz | Ear is most sensitive here, harsh if over-boosted | Boost max 3dB, check after 10min listening |
| Sibilance | 6-8 kHz | Harsh 's' and 't' sounds on vocals | De-ess with dynamic EQ or narrow cut |
| Ice pick | 8-12 kHz | Piercing, thin, cold | Cut on cymbals and synths if fatiguing |

**Golden rule**: Cut narrow (high Q), boost wide (low Q). If you need more than 6dB of boost, the source sound is wrong.

## Arrangement Patterns

### Section Map (Multi-Dimensional)

Define 5-dimension energy levels per section (see [energy-and-engagement.md](energy-and-engagement.md) for full system):

```python
# Example: 64-bar pop arrangement with composition plan
SECTIONS = [
    {'name': 'Intro',      'bars': 8,
     'energy': {'intensity': 3, 'density': 2, 'rhythm': 2, 'harmonic': 4, 'brightness': 3},
     'positive_styles': ['atmospheric', 'spacious'],
     'negative_styles': ['full beat', 'lead melody'],
     'transition_out': 'filter_open'},
    {'name': 'Verse 1',    'bars': 16,
     'energy': {'intensity': 5, 'density': 5, 'rhythm': 5, 'harmonic': 5, 'brightness': 5},
     'transition_in': 'impact', 'transition_out': 'riser'},
    {'name': 'Chorus 1',   'bars': 16,
     'energy': {'intensity': 8, 'density': 8, 'rhythm': 7, 'harmonic': 8, 'brightness': 9},
     'tension': 'PUSH'},
    {'name': 'Breakdown',  'bars': 8,
     'energy': {'intensity': 4, 'density': 3, 'rhythm': 2, 'harmonic': 6, 'brightness': 4}},
    {'name': 'Final Chorus','bars': 12,
     'energy': {'intensity': 10, 'density': 10, 'rhythm': 8, 'harmonic': 9, 'brightness': 10},
     'tension': 'PUSH — climax at 2/3 point'},
    {'name': 'Outro',      'bars': 4,
     'energy': {'intensity': 3, 'density': 2, 'rhythm': 2, 'harmonic': 4, 'brightness': 2},
     'positive_styles': ['callback to intro']},
]

# Build per-bar lookup:
bar_energy = build_bar_energy(SECTIONS)
```

### Element Introduction by Density Level

| Density | Elements Present |
|---------|-----------------|
| 1-2 | Ambient pad or single instrument, light texture |
| 3-4 | + Simple drums (kick, hat), bass enters |
| 5-6 | + Full drums, chord instrument, melody, bass groove |
| 7-8 | + Lead, strings, counter-melody, percussion layers |
| 9-10 | Everything + doubled parts, FX, harmony, widest stereo |

### Variation Rules
- Every 4 bars: small variation (hi-hat pattern change, fill, velocity shift)
- Every 8 bars: medium variation (drum pattern B, new bass note, filter dip)
- Every 16 bars: significant change (new section, element add/remove, ear candy)
- NEVER copy-paste the same pattern for more than 8 bars without variation
- **Contrast ratio**: peaks and valleys should differ by 4+ intensity points

## WAV Export (scipy fallback)

```python
def export_wav(filename, left, right, sr=SR):
    """Export stereo 16-bit WAV with TPDF dithering."""
    # Ensure same length
    n = max(len(left), len(right))
    stereo = np.zeros((n, 2))
    stereo[:len(left), 0] = left
    stereo[:len(right), 1] = right
    # Normalize to -1 dBFS
    peak = np.max(np.abs(stereo))
    if peak > 0:
        stereo *= (10 ** (-1/20)) / peak
    # TPDF dither
    dither = (np.random.uniform(-1,1,stereo.shape) + np.random.uniform(-1,1,stereo.shape))
    quantized = np.clip(np.round(stereo * 32767 + dither), -32767, 32767).astype(np.int16)
    wavfile.write(filename, sr, quantized)
    print(f"Wrote {filename}: {n/sr:.1f}s stereo at {sr}Hz")
```

## Memory Management

For 2-3 minute tracks (~5-8M samples per channel):

```python
# Memory per channel: 44100 * 180 * 8 bytes (float64) = ~60MB
# Total stereo: ~120MB -- fits easily in RAM

# Tips:
# - Process Freeverb per-track on MONO, then pan to stereo after
# - Use float32 if memory is tight: audio = audio.astype(np.float32)
# - Pre-allocate arrays: buf = np.zeros(TOTAL_SAMPLES) instead of np.append()
# - NEVER use np.concatenate() in loops (reallocates each time)
# - Process in chunks for very long audio (>10 minutes)
# - Delete intermediate arrays: del intermediate; gc.collect()
```

## Normalization and Dithering

```python
def normalize(sig, target_dbfs=-1.0):
    """-1 dBFS leaves headroom, avoids intersample peak clipping."""
    peak = np.max(np.abs(sig))
    if peak == 0: return sig
    return sig * (10 ** (target_dbfs / 20)) / peak

def rms_normalize(sig, target_rms_dbfs=-18.0):
    """Normalize by RMS (perceptually consistent loudness)."""
    rms = np.sqrt(np.mean(sig**2))
    if rms == 0: return sig
    return sig * (10 ** (target_rms_dbfs / 20)) / rms
```

## Useful Constants

```python
# Frequency conversion
def midi_to_freq(n): return 440.0 * (2 ** ((n - 69) / 12))
def freq_to_midi(f): return 69 + 12 * np.log2(f / 440)
def db_to_linear(db): return 10 ** (db / 20)
def bpm_to_ms(bpm, div=1): return 60000 / bpm * div  # div: 1=quarter, 0.5=8th

# Memory: 1 min mono float64 = ~20MB, float32 = ~10MB
# 16-bit dynamic range = 96 dB (6 dB per bit)
# Max stable filter order (SOS) = ~12
```
