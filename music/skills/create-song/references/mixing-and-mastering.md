# Mixing, Mastering, and Arrangement

Final mix pipeline, master chain, EQ reference, arrangement patterns, and memory management.

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

### 3. Section Automation

Use `smoothstep` for volume automation between sections:

```python
def section_volume(bar, section_map, total_samples, bar_dur_samples):
    """Get volume multiplier for current bar based on section energy."""
    energy = section_map[bar]  # 0-10 energy level
    return energy / 10.0

# Smooth transitions between sections (avoid clicks):
for bar in range(total_bars):
    start = int(bar * bar_samples)
    end = int((bar + 1) * bar_samples)
    current_energy = section_map[bar] / 10.0
    next_energy = section_map[min(bar + 1, total_bars - 1)] / 10.0
    # Smoothstep crossfade over last 10% of bar
    fade_start = int(end - 0.1 * bar_samples)
    for i in range(fade_start, end):
        t = (i - fade_start) / (end - fade_start)
        mix_buf[i] *= current_energy + (next_energy - current_energy) * smoothstep(t)
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

## Master Chain

Apply in this exact order:

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

## Pedalboard Master Chain (Preferred)

When pedalboard is available, use it for the master chain — it runs C++ JUCE audio processors at ~300x real-time, far higher quality than numpy equivalents. **Always prefer pedalboard over numpy for mastering when available.**

```python
from pedalboard import Pedalboard, Gain, Compressor, Limiter, HighpassFilter, LowpassFilter, PeakFilter, Clipping
from pedalboard.io import AudioFile

# Master chain — apply to final stereo mix (float32, shape [channels, samples])
master = Pedalboard([
    HighpassFilter(cutoff_frequency_hz=30),                          # Remove sub-rumble
    PeakFilter(cutoff_frequency_hz=200, gain_db=-2, q=1.0),         # Tame muddiness
    PeakFilter(cutoff_frequency_hz=3000, gain_db=1.5, q=0.8),       # Presence lift
    Compressor(threshold_db=-18, ratio=3, attack_ms=20, release_ms=150),  # Bus glue
    Gain(gain_db=3),                                                 # Makeup gain
    Limiter(threshold_db=-0.5, release_ms=100),                      # Brickwall limiter
])

# Apply — pedalboard expects float32 [channels, samples]
stereo = np.stack([left, right]).astype(np.float32)
mastered = master(stereo, SR)

# Export with soundfile (supports 24-bit WAV, FLAC, OGG)
import soundfile as sf
sf.write('output.wav', mastered.T, SR, subtype='PCM_24')     # 24-bit WAV
sf.write('output.flac', mastered.T, SR, subtype='PCM_24')    # FLAC
```

### Multiband Compression (Pedalboard)

Split into bands, compress independently, recombine:

```python
from pedalboard import Pedalboard, Compressor, HighpassFilter, LowpassFilter

def multiband_compress(audio, sr):
    """3-band multiband compression using pedalboard."""
    # Band split frequencies
    low_cut, high_cut = 200, 4000

    # Split into 3 bands
    low_band = Pedalboard([LowpassFilter(cutoff_frequency_hz=low_cut)])(audio, sr)
    mid_band = Pedalboard([
        HighpassFilter(cutoff_frequency_hz=low_cut),
        LowpassFilter(cutoff_frequency_hz=high_cut),
    ])(audio, sr)
    high_band = Pedalboard([HighpassFilter(cutoff_frequency_hz=high_cut)])(audio, sr)

    # Compress each band differently
    low_comp = Pedalboard([
        Compressor(threshold_db=-20, ratio=4, attack_ms=30, release_ms=200),
        Gain(gain_db=2),
    ])(low_band, sr)
    mid_comp = Pedalboard([
        Compressor(threshold_db=-16, ratio=2.5, attack_ms=15, release_ms=100),
        Gain(gain_db=1),
    ])(mid_band, sr)
    high_comp = Pedalboard([
        Compressor(threshold_db=-22, ratio=3, attack_ms=5, release_ms=80),
        Gain(gain_db=1.5),
    ])(high_band, sr)

    return low_comp + mid_comp + high_comp
```

### Mid/Side Mastering

Process the center (mid) and sides independently for better stereo control:

```python
def mid_side_master(left, right, sr):
    """Mid/side processing with pedalboard."""
    mid = (left + right) * 0.5
    side = (left - right) * 0.5

    # Stack to [1, samples] for pedalboard (mono)
    mid_2d = mid[np.newaxis, :].astype(np.float32)
    side_2d = side[np.newaxis, :].astype(np.float32)

    # Process mid: tighten bass, add presence
    mid_chain = Pedalboard([
        HighpassFilter(cutoff_frequency_hz=30),
        Compressor(threshold_db=-16, ratio=2.5, attack_ms=20, release_ms=150),
    ])
    mid_2d = mid_chain(mid_2d, sr)

    # Process side: remove bass (keep mono-compatible), add air
    side_chain = Pedalboard([
        HighpassFilter(cutoff_frequency_hz=200),   # No bass in sides
        PeakFilter(cutoff_frequency_hz=8000, gain_db=2, q=0.7),  # Air
        Compressor(threshold_db=-20, ratio=2, attack_ms=10, release_ms=100),
        Gain(gain_db=1.5),  # Widen stereo image
    ])
    side_2d = side_chain(side_2d, sr)

    # Decode back to L/R
    mid_out = mid_2d[0]
    side_out = side_2d[0]
    return mid_out + side_out, mid_out - side_out  # left, right
```

### Transient Shaping

Enhance or soften transients for punch control:

```python
def transient_shaper(audio, attack_gain=1.5, sustain_gain=0.8, sr=SR):
    """Transient shaper: boost/cut attack vs sustain independently."""
    # Envelope follower — fast for attack, slow for sustain
    fast_env = np.zeros_like(audio)
    slow_env = np.zeros_like(audio)
    fast_coeff = np.exp(-1 / (0.001 * sr))   # 1ms attack
    slow_coeff = np.exp(-1 / (0.050 * sr))   # 50ms sustain

    rectified = np.abs(audio)
    for i in range(1, len(audio)):
        fast_env[i] = max(rectified[i], fast_coeff * fast_env[i-1])
        slow_env[i] = max(rectified[i], slow_coeff * slow_env[i-1])

    # Transient = fast - slow; sustain = slow
    transient = np.clip(fast_env - slow_env, 0, None)
    sustain = slow_env

    # Normalize envelopes
    t_max = np.max(transient) or 1
    s_max = np.max(sustain) or 1
    transient /= t_max
    sustain /= s_max

    # Apply shaping
    gain = 1.0 + transient * (attack_gain - 1.0) + sustain * (sustain_gain - 1.0)
    return audio * gain
```

### Parallel Compression (New York Style)

Mix dry signal with heavily compressed version for density without losing dynamics:

```python
def parallel_compress(audio, sr, dry_mix=0.7, wet_mix=0.5):
    """Parallel compression — blend dry with crushed signal."""
    crushed_2d = audio[np.newaxis, :].astype(np.float32) if audio.ndim == 1 else audio.astype(np.float32)

    heavy_comp = Pedalboard([
        Compressor(threshold_db=-30, ratio=10, attack_ms=5, release_ms=50),
        Gain(gain_db=8),
    ])
    wet = heavy_comp(crushed_2d, sr)
    if audio.ndim == 1:
        wet = wet[0]

    return audio * dry_mix + wet * wet_mix
```

### Export Formats with soundfile

```python
import soundfile as sf

def export_audio(filename, left, right, sr=SR, fmt='wav', bit_depth=24):
    """Export stereo audio in multiple formats via soundfile."""
    stereo = np.stack([left, right], axis=-1).astype(np.float32)

    # Normalize to -1 dBFS
    peak = np.max(np.abs(stereo))
    if peak > 0:
        stereo *= (10 ** (-1/20)) / peak

    subtypes = {'wav': f'PCM_{bit_depth}', 'flac': f'PCM_{bit_depth}', 'ogg': 'VORBIS'}
    sf.write(filename, stereo, sr, subtype=subtypes.get(fmt, f'PCM_{bit_depth}'))
    print(f"Wrote {filename}: {len(left)/sr:.1f}s stereo at {sr}Hz ({fmt.upper()}, {bit_depth}-bit)")
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

## Arrangement Patterns

### Section Map

Define energy levels (0-10) per bar:

```python
# Example: 64-bar arrangement
section_map = {
    # Intro (bars 0-7): sparse, building
    **{i: 3 for i in range(8)},
    # Verse 1 (bars 8-23): groove established
    **{i: 5 for i in range(8, 24)},
    # Chorus 1 (bars 24-39): full energy
    **{i: 8 for i in range(24, 40)},
    # Bridge/Breakdown (bars 40-47): strip back
    **{i: 4 for i in range(40, 48)},
    # Final Chorus (bars 48-59): peak energy
    **{i: 10 for i in range(48, 60)},
    # Outro (bars 60-63): fade
    **{i: 3 for i in range(60, 64)},
}
```

### Element Introduction by Energy Level

| Energy | Elements Present |
|--------|-----------------|
| 1-2 | Ambient pad or single instrument, light texture |
| 3-4 | + Simple drums (kick, hat), bass enters |
| 5-6 | + Full drums, chord instrument, bass groove |
| 7-8 | + Melody/lead, all drums with fills, effects |
| 9-10 | Everything + doubled parts, extra layers, wider stereo |

### Variation Rules
- Every 4 bars: small variation (hi-hat pattern change, fill)
- Every 8 bars: medium variation (drum pattern B, new bass note)
- Every 16 bars: significant change (new section, element add/remove)
- NEVER copy-paste the same pattern for more than 8 bars without variation

## WAV Export

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



