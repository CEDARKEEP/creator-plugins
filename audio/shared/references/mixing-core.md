# Core Mixing & Audio Pipeline

Generic stereo mixing, master chain, EQ fundamentals, and export utilities. Shared across all audio generation skills.

## Stereo Mix Pipeline

### 1. Per-Track Processing

Each source track should be processed individually before mixing:

```python
# For each track:
track = source_builder(...)                 # Generate audio
track = compressor(track, ...)              # Per-track compression
track = lowpass(track, cutoff)              # Filter harshness
track = freeverb(track, ...) if applicable  # Reverb (skip on low-frequency sources)
left, right = pan_mono(track, pan_position) # Stereo placement
```

### 2. Multi-Dimensional Section Automation

Use multi-dimension energy maps for per-bar automation. Each dimension controls different mix parameters:

```python
def smoothstep(t):
    """Hermite smoothstep interpolation (0→1)."""
    t = np.clip(t, 0, 1)
    return t * t * (3 - 2 * t)

def build_bar_energy(sections):
    """Expand section energy into per-bar lookup for all dimensions."""
    bar_energy = []
    for section in sections:
        e = section['energy']  # dict of energy dimensions (e.g. intensity, density, brightness)
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
```

### 3. Sidechain Processing

Apply sidechain compression to let a dominant source punch through:

```python
# Sidechain a background layer to the dominant source
bg_signal = sidechain(bg_signal, dominant_signal,
                       threshold_db=-20, ratio=10.0,
                       attack_ms=1, release_ms=150)
```

Typical sidechain targets: sustained layers, ambient textures, background elements. NOT on foreground sources or transient-heavy tracks.

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
| Sub bass | 20-60 Hz | Feel, rumble | HPF everything except low-frequency sources |
| Bass | 60-250 Hz | Warmth, body | Keep for bass sources, cut on other tracks |
| Low mids | 250-500 Hz | Muddiness zone | Cut 3-6dB on most tracks to clean up |
| Mids | 500Hz-2kHz | Body, presence | Main voice of most sources |
| Upper mids | 2-4 kHz | Presence, ear peak | Boost for presence, cut if harsh |
| Treble | 4-8 kHz | Brightness, air | High-frequency transient sources live here |
| Air | 8-20 kHz | Sparkle, space | Gentle shelf boost for "air" |

### Problem Frequency Zones

| Zone | Frequency | Problem | Fix |
|------|-----------|---------|-----|
| Mud | 250-500 Hz | Muddiness, boomy, unclear | Cut 3-6dB on most tracks except low-frequency body |
| Honk | 800-1500 Hz | Nasal, boxy, cheap sounding | Cut if needed, narrow Q |
| Fatigue | 3-5 kHz | Ear is most sensitive here, harsh if over-boosted | Boost max 3dB, check after 10min listening |
| Sibilance | 6-8 kHz | Harsh transient artifacts | De-ess with dynamic EQ or narrow cut |
| Ice pick | 8-12 kHz | Piercing, thin, cold | Cut on bright sources if fatiguing |

**Golden rule**: Cut narrow (high Q), boost wide (low Q). If you need more than 6dB of boost, the source sound is wrong.

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

For long-form audio (~5-8M samples per channel):

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
