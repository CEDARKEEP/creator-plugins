# Mastering & Export (Pedalboard)

Pedalboard-based master chain (preferred), multiband compression, mid/side mastering, transient shaping, parallel compression, and multi-format export via soundfile.

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

## Multiband Compression (Pedalboard)

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

## Mid/Side Mastering

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

## Transient Shaping

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

## Parallel Compression (New York Style)

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

## Export Formats with soundfile

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
