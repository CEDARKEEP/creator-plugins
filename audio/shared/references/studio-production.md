# Studio Production Techniques

Professional techniques that separate amateur from studio-quality mixes. All implementations in numpy/scipy.

## Contents

- [Professional EQ Techniques](#1-professional-eq-techniques)
- [Advanced Compression](#2-advanced-compression)
- [Stereo Imaging & Spatial Design](#3-stereo-imaging--spatial-design)
- [Saturation & Harmonic Excitement](#4-saturation--harmonic-excitement)
- [Advanced Sidechain Techniques](#5-advanced-sidechain-techniques)
- [Reverb Design](#6-reverb-design)
- [Dynamic EQ](#7-dynamic-eq)
- [Loudness & Mastering Targets](#8-loudness--mastering-targets)
- [Analog Emulation](#9-analog-emulation)
- [Mix Bus Processing Chains](#10-mix-bus-processing-chains)
- [Transient Design](#11-transient-design)
- [Dithering & Noise Shaping](#12-dithering--noise-shaping)
- [Summary: Top 10 Amateur vs Professional Differences](#summary-top-10-amateur-vs-professional-differences)

## 1. Professional EQ Techniques

### Automatic Resonance Detection & Removal

```python
def find_resonances(sig, sr=SR, threshold_db=6, min_q=4):
    """Detect spectral peaks that indicate resonances.
    Returns list of (freq, amplitude_db, suggested_cut_db)."""
    from scipy.signal import welch
    from scipy.ndimage import median_filter
    freqs, psd = welch(sig, fs=sr, nperseg=4096, noverlap=2048)
    psd_db = 10 * np.log10(psd + 1e-20)
    baseline = median_filter(psd_db, size=31)
    peaks = psd_db - baseline
    resonances = []
    for i in range(1, len(peaks) - 1):
        if peaks[i] > threshold_db and peaks[i] > peaks[i-1] and peaks[i] > peaks[i+1]:
            resonances.append((freqs[i], peaks[i], -peaks[i] * 0.7))
    return resonances

def auto_resonance_remove(sig, sr=SR, threshold_db=6, max_cuts=5):
    """Automatically detect and surgically remove resonances."""
    resonances = find_resonances(sig, sr, threshold_db)
    resonances.sort(key=lambda x: x[1], reverse=True)
    result = sig.copy()
    for freq, amp_db, cut_db in resonances[:max_cuts]:
        Q = max(6, amp_db)
        result = peak_eq(result, freq, cut_db, Q=Q, sr=sr)
    return result
```

### Mid/Side EQ

The professional secret weapon for width and clarity:
- Cut sides below 200Hz — mono bass for club/speaker compatibility
- Boost sides at 8-12kHz (+1-3dB) — adds air/width without affecting center
- Boost mid at 3-5kHz (+1-2dB) — vocal/lead presence
- Cut mid at 250-500Hz (-2dB) — removes mud where bass and kick compete

```python
def mid_side_eq(left, right, sr=SR):
    """Professional mid/side EQ. Returns processed (left, right)."""
    mid = (left + right) * 0.5
    side = (left - right) * 0.5

    # MID: presence boost, mud cut
    mid = peak_eq(mid, 3500, 1.5, Q=0.8, sr=sr)
    mid = peak_eq(mid, 350, -2.0, Q=1.0, sr=sr)
    mid = low_shelf(mid, 80, 1.0, sr=sr)

    # SIDE: no bass, air boost
    sos_hp = butter(4, 200, btype='high', fs=sr, output='sos')
    side = sosfilt(sos_hp, side)
    side = high_shelf(side, 8000, 2.5, sr=sr)
    side = peak_eq(side, 12000, 1.5, Q=0.7, sr=sr)

    return mid + side, mid - side
```

### EQ Matching (Reference Track)

```python
def eq_match(sig, reference, sr=SR, smoothing=31, strength=0.7):
    """Match spectral envelope of reference track. strength: 0-1."""
    from scipy.signal import welch
    from scipy.ndimage import uniform_filter1d
    _, psd_sig = welch(sig, fs=sr, nperseg=4096)
    _, psd_ref = welch(reference, fs=sr, nperseg=4096)
    psd_sig_s = uniform_filter1d(np.log10(psd_sig + 1e-20), smoothing)
    psd_ref_s = uniform_filter1d(np.log10(psd_ref + 1e-20), smoothing)
    correction = (psd_ref_s - psd_sig_s) * strength
    # Apply via STFT
    fft_size = 4096
    hop = fft_size // 4
    frames = stft(sig, fft_size, hop)
    n_bins = fft_size // 2 + 1
    correction_interp = np.interp(np.arange(n_bins), np.linspace(0, n_bins, len(correction)), correction)
    gain = 10 ** correction_interp
    for i in range(len(frames)):
        frames[i] *= gain
    return istft(frames, fft_size, hop)[:len(sig)]
```

### Harmonic Exciter

```python
def harmonic_exciter(sig, freq_range=(3000, 12000), drive=2.0, mix=0.15, sr=SR):
    """Add harmonics above freq_range[0] via saturation. Aural exciter effect."""
    sos_hp = butter(2, freq_range[0], btype='high', fs=sr, output='sos')
    sos_lp = butter(2, freq_range[1], btype='low', fs=sr, output='sos')
    highs = sosfilt(sos_lp, sosfilt(sos_hp, sig))
    harmonics = np.tanh(highs * drive) - highs  # new harmonics only
    harmonics = sosfilt(sos_hp, harmonics)
    return sig + harmonics * mix
```

### Frequency Real Estate — Instrument Slots

| Instrument | Primary (Hz) | Secondary | Cut Zone |
|------------|-------------|-----------|----------|
| Sub bass | 30-60 | — | Cut above 120 |
| Kick body | 60-100 | 3-5k (click) | 200-400 (box) |
| Bass guitar | 80-200 | 700-1k (growl) | 250-400 (mud) |
| Snare body | 150-300 | 2-4k (crack) | — |
| Guitar | 300-3k | 5-8k (presence) | 80-200 (rumble) |
| Piano | 200-1k | 3-5k (definition) | — |
| Vocals | 1-5k | 8-12k (air) | 200-400 (mud) |
| Hi-hats | 6-16k | — | Below 400 |
| Pads | 300-2k | — | Below 200, above 8k |

### The 3dB Rule

- Boosts: gentle, broad (low Q 0.5-1.0, max +3dB)
- Cuts: narrow, surgical (high Q 4-12, as deep as needed)
- Philosophy: turn up what you like by 1-2dB; find what's bad and surgically remove it

---

## 2. Advanced Compression

### Parallel (New York) Compression

The compressed signal should be severely crushed (20:1, threshold -30dB). Blend it under the dry signal for density without losing transients.

```python
def parallel_compress_pro(sig, sr=SR, dry_level=1.0, wet_level=0.4,
                           threshold_db=-30, ratio=20, attack_ms=0.5,
                           release_ms=80, hpf_detector=True):
    """Professional parallel compression with detector HPF."""
    detector = sig.copy()
    if hpf_detector:
        sos = butter(2, 80, btype='high', fs=sr, output='sos')
        detector = sosfilt(sos, detector)
    compressed = compressor_with_sidechain(sig, detector,
                                            threshold_db=threshold_db,
                                            ratio=ratio, attack_ms=attack_ms,
                                            release_ms=release_ms, sr=sr)
    rms_orig = np.sqrt(np.mean(sig**2))
    rms_comp = np.sqrt(np.mean(compressed**2))
    if rms_comp > 0:
        compressed *= rms_orig / rms_comp
    return sig * dry_level + compressed * wet_level
```

### Serial Compression

Two gentle stages = more transparent than one heavy stage:

```python
def serial_compress(sig, sr=SR):
    """Two-stage serial compression."""
    stage1 = compressor(sig, threshold_db=-18, ratio=2.5,
                        attack_ms=1, release_ms=60, knee_db=4, sr=sr)
    stage2 = compressor(stage1, threshold_db=-14, ratio=2.0,
                        attack_ms=20, release_ms=200, knee_db=6, sr=sr)
    return stage2
```

### Multiband Compression Presets

Standard 4-band crossovers: 100Hz | 1kHz | 8kHz

```python
MULTIBAND_PRESETS = {
    'mastering': {
        'crossovers': [100, 1000, 8000],
        'bands': [
            {'threshold': -18, 'ratio': 2.5, 'attack': 20, 'release': 200, 'gain': 0},
            {'threshold': -16, 'ratio': 2.0, 'attack': 10, 'release': 150, 'gain': 0},
            {'threshold': -20, 'ratio': 2.0, 'attack': 5, 'release': 100, 'gain': 0},
            {'threshold': -22, 'ratio': 2.5, 'attack': 3, 'release': 80, 'gain': 1},
        ]
    },
    'edm_master': {
        'crossovers': [80, 2500, 8000],
        'bands': [
            {'threshold': -12, 'ratio': 4.0, 'attack': 30, 'release': 250, 'gain': 2},
            {'threshold': -14, 'ratio': 2.5, 'attack': 8, 'release': 120, 'gain': 0},
            {'threshold': -16, 'ratio': 2.0, 'attack': 3, 'release': 80, 'gain': 1},
            {'threshold': -20, 'ratio': 3.0, 'attack': 1, 'release': 60, 'gain': 2},
        ]
    },
    'vocal_multiband': {
        'crossovers': [200, 2000, 6000],
        'bands': [
            {'threshold': -20, 'ratio': 3.0, 'attack': 10, 'release': 150, 'gain': -1},
            {'threshold': -16, 'ratio': 2.0, 'attack': 8, 'release': 100, 'gain': 1},
            {'threshold': -14, 'ratio': 2.5, 'attack': 3, 'release': 80, 'gain': 2},
            {'threshold': -18, 'ratio': 3.0, 'attack': 1, 'release': 50, 'gain': 0},
        ]
    }
}
```

### Per-Instrument Compression Settings

| Instrument | Threshold | Ratio | Attack | Release | Notes |
|------------|-----------|-------|--------|---------|-------|
| Kick | -12dB | 4:1 | 0.5-2ms | 50-80ms | Fast attack catches transient |
| Snare | -15dB | 3:1-4:1 | 1-5ms | 80-120ms | Let initial crack through |
| Bass | -16dB | 3:1-4:1 | 5-15ms | 100-200ms | Even out note dynamics |
| Acoustic guitar | -18dB | 2:1-3:1 | 10-20ms | 100-150ms | Gentle, preserve dynamics |
| Vocals | -18dB | 2.5:1-4:1 | 2-10ms | 80-150ms | Fast enough for consonants |
| Drum bus | -12dB | 2:1-4:1 | 10-30ms | 100-200ms | Let transient through |
| Mix bus | -12dB | 2:1 | 20-30ms | 150-300ms | SSL-style glue |
| Mastering | -6 to -3dB | 1.5:1-2:1 | 30ms+ | auto/300ms | Barely touching |

### Sidechain Compressor with Filtered Detector

```python
def compressor_with_sidechain(sig, sidechain_sig, threshold_db=-20, ratio=4.0,
                                attack_ms=10, release_ms=100, knee_db=6, sr=SR):
    """Compressor where gain reduction is determined by sidechain_sig."""
    n = len(sig)
    ac = np.exp(-1.0 / (attack_ms * sr / 1000))
    rc = np.exp(-1.0 / (release_ms * sr / 1000))
    window = int(sr * 0.01)
    level = np.sqrt(np.convolve(sidechain_sig**2, np.ones(window)/window, mode='same'))
    env = 0.0
    gr = np.ones(n)
    for i in range(n):
        if level[i] > env:
            env = ac * env + (1 - ac) * level[i]
        else:
            env = rc * env + (1 - rc) * level[i]
        if env <= 0: continue
        env_db = 20 * np.log10(max(env, 1e-10))
        over_db = env_db - threshold_db
        if knee_db > 0 and -knee_db/2 < over_db < knee_db/2:
            over_db = (over_db + knee_db/2)**2 / (2 * knee_db)
        if over_db > 0:
            gr[i] = 10 ** (-(over_db * (1 - 1/ratio)) / 20)
    return sig * gr
```

---

## 3. Stereo Imaging & Spatial Design

### Frequency-Dependent Stereo Width

The single most important stereo technique: mono bass, moderate mids, wide highs.

```python
def freq_dependent_width(left, right, sr=SR):
    """3-band stereo width: mono bass, moderate mids, wide highs."""
    sos_lo = butter(4, 200, btype='low', fs=sr, output='sos')
    sos_mi_lo = butter(4, 200, btype='high', fs=sr, output='sos')
    sos_mi_hi = butter(4, 4000, btype='low', fs=sr, output='sos')
    sos_hi = butter(4, 4000, btype='high', fs=sr, output='sos')

    # Low: force mono
    lo_l, lo_r = sosfilt(sos_lo, left), sosfilt(sos_lo, right)
    lo_mono = (lo_l + lo_r) * 0.5

    # Mid: slight widening (1.2x)
    mi_l = sosfilt(sos_mi_hi, sosfilt(sos_mi_lo, left))
    mi_r = sosfilt(sos_mi_hi, sosfilt(sos_mi_lo, right))
    mid_m, mid_s = (mi_l + mi_r) * 0.5, (mi_l - mi_r) * 0.5 * 1.2

    # High: wider (1.5x)
    hi_l, hi_r = sosfilt(sos_hi, left), sosfilt(sos_hi, right)
    hi_m, hi_s = (hi_l + hi_r) * 0.5, (hi_l - hi_r) * 0.5 * 1.5

    return (lo_mono + mid_m + mid_s + hi_m + hi_s,
            lo_mono + mid_m - mid_s + hi_m - hi_s)
```

### Mono-Compatible Haas Widening

Basic Haas causes comb filtering in mono. Fix: short delay + HPF the delayed channel:

```python
def haas_safe(mono, delay_ms=5, sr=SR):
    """Mono-compatible Haas widening. HPF delayed side to avoid LF comb filtering."""
    delay_samples = int(delay_ms * sr / 1000)
    delayed = np.zeros_like(mono)
    delayed[delay_samples:] = mono[:-delay_samples]
    sos = butter(2, 800, btype='high', fs=sr, output='sos')
    delayed = sosfilt(sos, delayed)
    return mono, delayed * 0.8
```

### Stereo Correlation Monitoring

| Target | Correlation | Notes |
|--------|------------|-------|
| Overall mix | +0.3 to +0.7 | Predominantly positive |
| Wide sections | +0.1 to +0.4 | Pads, ambient |
| Bass/kick | +0.9 to +1.0 | Nearly mono |
| DANGER | < 0.0 sustained | Will cancel in mono |

```python
def check_mono_compatibility(left, right, sr=SR):
    """Returns average correlation and warns if poor mono compatibility."""
    window = int(0.05 * sr)
    n = len(left)
    corr_vals = []
    for i in range(window, n, window):
        l_win = left[i-window:i] - np.mean(left[i-window:i])
        r_win = right[i-window:i] - np.mean(right[i-window:i])
        denom = np.sqrt(np.sum(l_win**2) * np.sum(r_win**2))
        if denom > 1e-10:
            corr_vals.append(np.sum(l_win * r_win) / denom)
    avg = np.mean(corr_vals) if corr_vals else 1.0
    if avg < 0.2:
        print(f"WARNING: Low stereo correlation ({avg:.2f}). May sound thin in mono.")
    return avg
```

### LCR Panning Philosophy

Many top mixers use only Left, Center, Right — no in-between. Creates a more powerful stereo image:

```python
LCR_PANNING = {
    'kick': 0.5, 'bass': 0.5, 'snare': 0.5, 'lead_vocal': 0.5,
    'guitar_L': 0.0, 'guitar_R': 1.0,
    'keys_L': 0.0, 'keys_R': 1.0,
    'backing_vox_L': 0.0, 'backing_vox_R': 1.0,
}
```

---

## 4. Saturation & Harmonic Excitement

### Three Types of Analog Saturation

| Type | Harmonics | Character | Transfer Function |
|------|-----------|-----------|-------------------|
| Tube | Even (2nd, 4th) | Warm, musical | Asymmetric: `1-exp(-x)` / `-(1-exp(x))` |
| Tape | Even + odd + compression | Smooth, glue | `(2/pi) * arctan(x)` + head bump + HF rolloff |
| Transistor | Odd (3rd, 5th) | Gritty, aggressive | Symmetric diode clip |

### Tube Saturation

```python
def tube_saturation(sig, drive=2.0, mix=0.5, bias=0.1):
    """Tube-style: asymmetric clipping generates even harmonics.
    bias: DC offset for asymmetry (0.05-0.2). drive: 1=clean, 2=warm, 4=overdriven."""
    x = sig * drive + bias
    positive = 1 - np.exp(-x)
    negative = -(1 - np.exp(x))
    saturated = np.where(x >= 0, positive, negative)
    saturated -= np.mean(saturated)
    rms_in = np.sqrt(np.mean(sig**2))
    rms_out = np.sqrt(np.mean(saturated**2))
    if rms_out > 0:
        saturated *= rms_in / rms_out
    return sig * (1 - mix) + saturated * mix
```

### Professional Tape Saturation

```python
def tape_saturation_pro(sig, drive=1.5, bias=0.5, speed='15ips', sr=SR):
    """Tape model: arctan + head bump + HF rolloff + self-compression.
    speed: '7.5ips' (more sat, less HF) or '15ips' (cleaner, more HF)."""
    x = sig * drive
    saturated = (2 / np.pi) * np.arctan(x * np.pi / 2 * (1 + bias))
    bump_freq = 60 if speed == '15ips' else 40
    saturated = peak_eq(saturated, bump_freq, 2.0, Q=1.5, sr=sr)
    rolloff = 16000 if speed == '15ips' else 10000
    sos = butter(1, rolloff, btype='low', fs=sr, output='sos')
    saturated = sosfilt(sos, saturated)
    saturated = compressor(saturated, threshold_db=-12, ratio=1.5,
                           attack_ms=5, release_ms=80, knee_db=10, sr=sr)
    return saturated
```

### Transistor Clipping

```python
def transistor_clip(sig, drive=3.0, asymmetry=0.0):
    """Transistor-style symmetric clipping. asymmetry>0 adds even harmonics."""
    x = sig * drive
    threshold = 0.7
    pos_clip = threshold * np.tanh(x / threshold)
    neg_clip = -(threshold * (1 + asymmetry)) * np.tanh(-x / (threshold * (1 + asymmetry)))
    out = np.where(x >= 0, pos_clip, neg_clip)
    return out / (np.max(np.abs(out)) + 1e-10)
```

### Waveshaping Transfer Functions

| Name | Formula | Character |
|------|---------|-----------|
| Soft clip (tanh) | `tanh(x * drive)` | Warm, round |
| Arctangent | `(2/pi) * arctan(x * drive)` | Tape-like |
| Cubic | `x - x^3/3` | Gentle, 3rd harmonic only |
| Chebyshev T2 | `2x^2 - 1` | Pure 2nd harmonic |
| Asymmetric exp | See tube_saturation | Tube, even harmonics |
| Foldback | `4*abs(mod(x+1,2)-1)-2` | Aggressive, dense |

---

## 5. Advanced Sidechain Techniques

### Frequency-Specific Sidechain (Duck Only Conflicting Bands)

```python
def multiband_sidechain(sig, trigger, bands=[(200, 500)], threshold_db=-20,
                         ratio=4.0, attack_ms=5, release_ms=100, sr=SR):
    """Sidechain only specific frequency bands. E.g., duck pad at 200-500Hz when vocal plays."""
    result = sig.copy()
    for low, high in bands:
        sos_bp = butter(2, [low, high], btype='band', fs=sr, output='sos')
        band = sosfilt(sos_bp, sig)
        trigger_band = sosfilt(sos_bp, trigger)
        compressed_band = compressor_with_sidechain(
            band, trigger_band, threshold_db=threshold_db,
            ratio=ratio, attack_ms=attack_ms, release_ms=release_ms, sr=sr)
        result = result - band + compressed_band
    return result
```

### Modern Volume Shaping (LFOtool-style)

```python
def volume_shaper(sig, bpm, shape='smooth_pump', depth=0.7, sr=SR):
    """Precise rhythmic volume shaping. More control than sidechain compression."""
    beat_samples = int(60 * sr / bpm)
    n = len(sig)
    SHAPES = {
        'smooth_pump': lambda t: (1 - depth) + depth * (1 - np.exp(-t * 6)),
        'hard_gate': lambda t: np.where(t < 0.1, 1 - depth, 1.0),
        'trance_gate': lambda t: np.where(t % 0.25 < 0.15, 1.0, 1 - depth),
        'half_time': lambda t: (1 - depth) + depth * (1 - np.exp(-t * 3)),
    }
    shape_func = SHAPES[shape]
    step = beat_samples * (2 if shape == 'half_time' else 1)
    env = np.ones(n)
    for pos in range(0, n, step):
        length = min(step, n - pos)
        env[pos:pos+length] = shape_func(np.linspace(0, 1, length))
    return sig * env
```

---

## 6. Reverb Design

### Two-Stage Professional Reverb

```python
def professional_reverb(sig, room_type='hall', pre_delay_ms=40, wet=0.25, sr=SR):
    """Early reflections + late Freeverb diffusion. Always EQ the output."""
    ROOM_TYPES = {
        'plate':     {'er_delays': [7,13,19,23,29,37], 'er_gains': [0.8,0.7,0.5,0.4,0.3,0.2],
                      'decay': 1.5, 'damping': 0.5, 'hf_damp': 8000},
        'hall':      {'er_delays': [11,23,37,41,53,67], 'er_gains': [0.6,0.5,0.4,0.35,0.25,0.15],
                      'decay': 2.5, 'damping': 0.3, 'hf_damp': 6000},
        'room':      {'er_delays': [5,11,17,23,29,31], 'er_gains': [0.8,0.6,0.4,0.3,0.2,0.15],
                      'decay': 0.6, 'damping': 0.6, 'hf_damp': 10000},
        'spring':    {'er_delays': [30,33,36,39,42,45], 'er_gains': [0.7,0.65,0.5,0.4,0.3,0.2],
                      'decay': 1.8, 'damping': 0.4, 'hf_damp': 5000},
        'chamber':   {'er_delays': [9,17,23,31,41,47], 'er_gains': [0.7,0.6,0.5,0.4,0.3,0.2],
                      'decay': 1.2, 'damping': 0.45, 'hf_damp': 7000},
        'cathedral': {'er_delays': [19,37,53,71,89,97], 'er_gains': [0.5,0.45,0.4,0.35,0.3,0.25],
                      'decay': 4.0, 'damping': 0.15, 'hf_damp': 4000},
    }
    params = ROOM_TYPES[room_type]
    n = len(sig)
    pre_delay = int(pre_delay_ms * sr / 1000)
    delayed_sig = np.zeros(n)
    if pre_delay < n:
        delayed_sig[pre_delay:] = sig[:n - pre_delay]
    # Early reflections
    early = np.zeros(n)
    for delay_ms, gain in zip(params['er_delays'], params['er_gains']):
        d = int(delay_ms * sr / 1000)
        if d < n:
            early[d:] += delayed_sig[:n-d] * gain
    # Late diffusion
    late = freeverb(early, room=params['decay']/4.0, damp=params['damping'], wet=1.0, sr=sr)
    sos = butter(1, params['hf_damp'], btype='low', fs=sr, output='sos')
    late = sosfilt(sos, late)
    # EQ the reverb return (CRITICAL)
    sos_hp = butter(2, 250, btype='high', fs=sr, output='sos')
    sos_lp = butter(2, min(params['hf_damp'], sr/2 - 100), btype='low', fs=sr, output='sos')
    reverb_out = sosfilt(sos_lp, sosfilt(sos_hp, early * 0.5 + late * 0.5))
    return sig * (1 - wet) + reverb_out * wet
```

### Pre-Delay Guidelines

| Element | Pre-delay | Notes |
|---------|-----------|-------|
| Vocals | 50-100ms | Keeps vocal in front |
| Drums | 0-20ms | Tight, part of drum sound |
| Strings/Pads | 20-40ms | Blended |
| Guitar | 30-60ms | Moderate separation |

### Decay Time by Genre

| Genre | Decay (s) | Damping | Pre-delay | Notes |
|-------|-----------|---------|-----------|-------|
| Pop | 1.0-1.5 | 0.5 | 40-80ms | Clean, present |
| Rock | 0.8-1.2 | 0.5 | 20-40ms | Not too washy |
| Jazz | 1.5-2.5 | 0.3 | 30-60ms | Natural room |
| Classical | 2.0-4.0 | 0.2 | 0-20ms | Concert hall |
| Ambient | 3.0-8.0+ | 0.1-0.2 | 50-100ms | Huge, ethereal |
| EDM | 0.5-1.0 | 0.6-0.8 | 0-20ms | Tight, controlled |
| Lo-fi | 1.0-2.0 | 0.6 | 20-40ms | Degraded, filtered |
| Trap | 0.8-1.5 | 0.5 | 20-40ms | Plate on snare |

---

## 7. Dynamic EQ

### Frequency-Targeted Dynamic Processing

```python
def dynamic_eq(sig, center_freq, threshold_db=-20, max_cut_db=-8,
               Q=2.0, attack_ms=5, release_ms=50, sr=SR):
    """Only cuts when energy at center_freq exceeds threshold.
    Use for de-essing (5-8kHz), controlling boominess (200-400Hz)."""
    bw = center_freq / Q
    lo, hi = max(20, center_freq - bw/2), min(sr/2 - 100, center_freq + bw/2)
    sos_bp = butter(2, [lo, hi], btype='band', fs=sr, output='sos')
    band_level = np.abs(sosfilt(sos_bp, sig))
    ac = np.exp(-1 / (attack_ms * sr / 1000))
    rc = np.exp(-1 / (release_ms * sr / 1000))
    threshold = 10 ** (threshold_db / 20)
    env = np.zeros(len(sig))
    e = 0.0
    for i in range(len(sig)):
        if band_level[i] > e:
            e = ac * e + (1 - ac) * band_level[i]
        else:
            e = rc * e + (1 - rc) * band_level[i]
        env[i] = e
    over = np.maximum(env - threshold, 0) / (np.max(env) - threshold + 1e-10)
    eq_gain_db = max_cut_db * over
    # Apply as time-varying band reduction
    band = sosfilt(sos_bp, sig)
    gain = 10 ** (eq_gain_db / 20)
    return sig - band + band * gain
```

### De-Esser

```python
def de_esser(sig, frequency=6500, threshold_db=-25, max_reduction_db=-10,
             bandwidth_hz=3000, sr=SR):
    """Dynamic EQ targeting sibilance at 5-8kHz."""
    lo = max(20, frequency - bandwidth_hz/2)
    hi = min(sr/2 - 100, frequency + bandwidth_hz/2)
    sos = butter(2, [lo, hi], btype='band', fs=sr, output='sos')
    sibilance = sosfilt(sos, sig)
    sib_env = np.abs(sibilance)
    smooth = int(0.005 * sr)
    sib_env = np.convolve(sib_env, np.ones(smooth)/smooth, mode='same')
    threshold = 10 ** (threshold_db / 20)
    max_reduction = 10 ** (max_reduction_db / 20)
    gain = np.ones(len(sig))
    for i in range(len(sig)):
        if sib_env[i] > threshold:
            gain[i] = max(max_reduction, threshold / sib_env[i])
    return sig - sibilance + sibilance * gain
```

---

## 8. Loudness & Mastering Targets

### LUFS Targets by Platform

| Platform | Integrated LUFS | True Peak | Normalization |
|----------|----------------|-----------|---------------|
| Spotify | -14 LUFS | -1 dBTP | Up and down |
| Apple Music | -16 LUFS | -1 dBTP | Sound Check (on by default) |
| YouTube | -14 LUFS | -1 dBTP | Down only |
| Tidal | -14 LUFS | -1 dBTP | Down only |
| Amazon Music | -14 LUFS | -2 dBTP | Down only |
| Broadcast (EBU R128) | -23 LUFS | -1 dBTP | Strict |
| Podcast | -16 LUFS | -1 dBTP | Varies |

### True Peak Limiter with Oversampling

```python
def true_peak_limiter(sig, threshold_db=-1.0, release_ms=50, oversample=4, sr=SR):
    """True peak limiter — catches intersample peaks via oversampling."""
    from scipy.signal import resample_poly
    upsampled = resample_poly(sig, oversample, 1)
    up_sr = sr * oversample
    threshold_lin = 10 ** (threshold_db / 20)
    abs_up = np.abs(upsampled)
    lookahead = int(0.0005 * up_sr)
    gr = np.ones(len(upsampled))
    for i in range(len(upsampled)):
        end = min(i + lookahead, len(upsampled))
        peak = np.max(abs_up[i:end]) if end > i else abs_up[i]
        if peak > threshold_lin:
            gr[i] = threshold_lin / peak
    release_c = np.exp(-1 / (release_ms * up_sr / 1000))
    attack_c = np.exp(-1 / (0.1 * up_sr / 1000))
    smoothed = np.ones(len(gr))
    s = 1.0
    for i in range(len(gr)):
        if gr[i] < s:
            s = attack_c * s + (1 - attack_c) * gr[i]
        else:
            s = release_c * s + (1 - release_c) * gr[i]
        smoothed[i] = s
    limited_up = upsampled * smoothed
    return resample_poly(limited_up, 1, oversample)[:len(sig)]
```

### Dynamics Measurement

```python
def measure_dynamics(sig, sr=SR):
    """Measure crest factor and dynamic range metrics."""
    peak = np.max(np.abs(sig))
    rms = np.sqrt(np.mean(sig**2))
    crest_factor_db = 20 * np.log10(peak / (rms + 1e-10))
    window = int(3 * sr)
    rms_windows = []
    for i in range(0, len(sig) - window, window // 2):
        w_rms = np.sqrt(np.mean(sig[i:i+window]**2))
        if w_rms > 1e-6:
            rms_windows.append(20 * np.log10(w_rms))
    dynamic_range = max(rms_windows) - min(rms_windows) if rms_windows else 0
    return {'crest_factor_db': crest_factor_db, 'peak_dbfs': 20*np.log10(peak+1e-10),
            'rms_dbfs': 20*np.log10(rms+1e-10), 'dynamic_range_db': dynamic_range}
```

### Crest Factor Targets by Genre

| Genre | Crest Factor | Notes |
|-------|-------------|-------|
| Unmastered | 14-20 dB | Full dynamics |
| Classical | 14-18 dB | Preserve dynamics |
| Jazz/Acoustic | 10-14 dB | Some control |
| Pop/Rock | 8-12 dB | Modern standard |
| EDM/Hip-hop | 6-8 dB | Louder, less dynamic |
| Hypercompressed | 4-6 dB | Loudness war (avoid) |

---

## 9. Analog Emulation

### Transformer Saturation

```python
def transformer_saturation(sig, drive=1.2, sr=SR):
    """Subtle 3rd harmonic + bass bump + HF rolloff."""
    x = sig * drive
    saturated = x - (x**3) * 0.05
    saturated = peak_eq(saturated, 60, 0.8, Q=1.5, sr=sr)
    sos = butter(1, 18000, btype='low', fs=sr, output='sos')
    saturated = sosfilt(sos, saturated)
    return saturated / drive
```

### Capacitor Coupling (AC Coupling)

```python
def capacitor_coupling(sig, cutoff=25, resonance=0.5, sr=SR):
    """HPF at 20-30Hz with slight resonance — the analog low-end character."""
    w0 = 2 * np.pi * cutoff / sr
    Q = 0.707 + resonance
    alpha = np.sin(w0) / (2 * Q)
    b0 = (1 + np.cos(w0)) / 2
    b1 = -(1 + np.cos(w0))
    b2 = (1 + np.cos(w0)) / 2
    a0 = 1 + alpha
    a1 = -2 * np.cos(w0)
    a2 = 1 - alpha
    sos = np.array([[b0/a0, b1/a0, b2/a0, 1.0, a1/a0, a2/a0]])
    return sosfilt(sos, sig)
```

### Analog Channel Strip

```python
def analog_channel(sig, sr=SR, character='warm'):
    """Complete analog console channel: 'warm' (Neve), 'clean' (SSL), 'colored' (API)."""
    CHARACTERS = {
        'warm':    {'transformer_drive': 1.2, 'hf_rolloff': 16000, 'cap_cutoff': 20, 'cap_res': 0.3, 'noise_db': -72},
        'clean':   {'transformer_drive': 1.05, 'hf_rolloff': 20000, 'cap_cutoff': 15, 'cap_res': 0.1, 'noise_db': -80},
        'colored': {'transformer_drive': 1.4, 'hf_rolloff': 14000, 'cap_cutoff': 25, 'cap_res': 0.5, 'noise_db': -66},
    }
    c = CHARACTERS[character]
    sig = transformer_saturation(sig, drive=c['transformer_drive'], sr=sr)
    sig = capacitor_coupling(sig, cutoff=c['cap_cutoff'], resonance=c['cap_res'], sr=sr)
    sos = butter(1, min(c['hf_rolloff'], sr/2-100), btype='low', fs=sr, output='sos')
    sig = sosfilt(sos, sig)
    return sig
```

### Wow & Flutter

```python
def analog_pitch_drift(sig, wow_rate=0.5, wow_depth_cents=3, flutter_rate=6,
                        flutter_depth_cents=1, sr=SR):
    """Wow (slow <2Hz) and flutter (4-12Hz) pitch modulation."""
    n = len(sig)
    t = np.arange(n) / sr
    wow = wow_depth_cents / 1200.0 * np.sin(2*np.pi*wow_rate*t + np.random.uniform(0, 2*np.pi))
    wow += (wow_depth_cents * 0.3) / 1200.0 * np.sin(2*np.pi*wow_rate*0.7*t)
    flutter = flutter_depth_cents / 1200.0 * np.sin(2*np.pi*flutter_rate*t)
    mod = wow + flutter
    read_positions = np.arange(n) + np.cumsum(mod) * sr
    read_positions = np.clip(read_positions, 0, n - 2)
    idx = read_positions.astype(int)
    frac = read_positions - idx
    return sig[idx] * (1 - frac) + sig[np.minimum(idx + 1, n - 1)] * frac
```

---

## 10. Mix Bus Processing Chains

### SSL Bus Compressor

```python
def ssl_bus_compressor(sig, threshold_db=-12, ratio=2.0, attack_ms=30,
                        release='auto', makeup_db=0, sr=SR):
    """SSL-style bus compressor. On more hit records than any other single piece of gear.
    attack: 10ms (punchy) or 30ms (breathe — preferred). release: 'auto' or ms."""
    n = len(sig)
    ac = np.exp(-1 / (attack_ms * sr / 1000))
    if release == 'auto':
        rc_fast = np.exp(-1 / (100 * sr / 1000))
        rc_slow = np.exp(-1 / (600 * sr / 1000))
    else:
        rc_fast = rc_slow = np.exp(-1 / (release * sr / 1000))
    threshold = 10 ** (threshold_db / 20)
    makeup = 10 ** (makeup_db / 20)
    window = int(sr * 0.01)
    level = np.sqrt(np.convolve(sig**2, np.ones(window)/window, mode='same'))
    env = 0.0
    gr = np.ones(n)
    for i in range(n):
        target = level[i]
        if target > env:
            env = ac * env + (1 - ac) * target
        else:
            if release == 'auto':
                gr_amount = max(0, 20*np.log10(max(env,1e-10)) - threshold_db)
                blend = min(gr_amount / 12, 1.0)
                rc = rc_fast * (1 - blend) + rc_slow * blend
            else:
                rc = rc_fast
            env = rc * env + (1 - rc) * target
        if env > 0:
            env_db = 20 * np.log10(max(env, 1e-10))
            over_db = env_db - threshold_db
            if over_db > 0:
                gr[i] = 10 ** (-(over_db * (1 - 1/ratio)) / 20)
    return sig * gr * makeup
```

### Pultec-Style EQ

```python
def pultec_bass(sig, freq=60, boost_db=3, atten_db=2, sr=SR):
    """Pultec bass trick: wide shelf boost + narrower cut slightly above.
    Result: resonant peak with dip above — tight, punchy bass."""
    result = low_shelf(sig, freq, boost_db, sr=sr)
    result = peak_eq(result, freq * 1.5, -atten_db, Q=1.0, sr=sr)
    return result

def pultec_treble(sig, freq=10000, boost_db=2, atten_db=1, sr=SR):
    """Pultec treble: shelf boost + cut below. Airy, open top end."""
    result = high_shelf(sig, freq, boost_db, sr=sr)
    result = peak_eq(result, freq * 0.7, -atten_db, Q=0.7, sr=sr)
    return result
```

### Professional Mix Bus Chain Presets

```python
PRO_MIX_BUS_CHAINS = {
    'modern_pop': [
        ('pultec_bass', {'freq': 60, 'boost_db': 2, 'atten_db': 1.5}),
        ('pultec_treble', {'freq': 12000, 'boost_db': 1.5, 'atten_db': 0.5}),
        ('ssl_bus', {'threshold_db': -14, 'ratio': 2, 'attack_ms': 30}),
        ('tape_sat', {'drive': 1.3}),
        ('mid_side_eq', {}),
        ('limiter', {'threshold_db': -1}),
    ],
    'aggressive_edm': [
        ('ssl_bus', {'threshold_db': -10, 'ratio': 4, 'attack_ms': 10}),
        ('multiband_comp', {'crossovers': [80, 2500, 8000]}),
        ('tape_sat', {'drive': 1.8}),
        ('limiter', {'threshold_db': -0.3}),
    ],
    'gentle_acoustic': [
        ('pultec_bass', {'freq': 80, 'boost_db': 1.5, 'atten_db': 1}),
        ('ssl_bus', {'threshold_db': -18, 'ratio': 1.5, 'attack_ms': 30}),
        ('mid_side_eq', {}),
        ('limiter', {'threshold_db': -1}),
    ],
}
```

---

## 11. Transient Design

### Professional Transient Shaper

```python
def transient_designer(sig, attack_db=0, sustain_db=0, sr=SR):
    """Dual-envelope transient designer. attack_db/sustain_db: -12 to +12."""
    n = len(sig)
    rect = np.abs(sig)
    fast_att = np.exp(-1 / (0.0003 * sr))
    fast_rel = np.exp(-1 / (0.005 * sr))
    slow_att = np.exp(-1 / (0.020 * sr))
    slow_rel = np.exp(-1 / (0.200 * sr))
    fast_env = np.zeros(n)
    slow_env = np.zeros(n)
    fe = se = 0.0
    for i in range(n):
        if rect[i] > fe: fe = fast_att * fe + (1 - fast_att) * rect[i]
        else: fe = fast_rel * fe + (1 - fast_rel) * rect[i]
        fast_env[i] = fe
        if rect[i] > se: se = slow_att * se + (1 - slow_att) * rect[i]
        else: se = slow_rel * se + (1 - slow_rel) * rect[i]
        slow_env[i] = se
    transient = np.maximum(fast_env - slow_env, 0)
    transient_norm = transient / (np.max(transient) + 1e-10)
    sustain_norm = slow_env / (np.max(slow_env) + 1e-10)
    attack_gain = 10 ** (attack_db / 20) - 1
    sustain_gain = 10 ** (sustain_db / 20) - 1
    gain = np.clip(1.0 + attack_gain * transient_norm + sustain_gain * sustain_norm, 0.05, 10.0)
    return sig * gain
```

### Transient Presets

```python
TRANSIENT_PRESETS = {
    'kick_punch':      {'attack_db': 6, 'sustain_db': -3},
    'kick_boom':       {'attack_db': -2, 'sustain_db': 6},
    'snare_snap':      {'attack_db': 8, 'sustain_db': -4},
    'snare_fat':       {'attack_db': 2, 'sustain_db': 4},
    'hihat_crisp':     {'attack_db': 4, 'sustain_db': -6},
    'room_mics':       {'attack_db': -6, 'sustain_db': 8},
    'bass_tighten':    {'attack_db': 2, 'sustain_db': -4},
    'guitar_sustain':  {'attack_db': -3, 'sustain_db': 6},
    'piano_soft':      {'attack_db': -6, 'sustain_db': 2},
}
```

---

## 12. Dithering & Noise Shaping

### TPDF Dither

```python
def tpdf_dither(sig_float, target_bits=16):
    """TPDF dither for bit-depth reduction. The standard professional method."""
    max_val = 2 ** (target_bits - 1) - 1
    scaled = sig_float * max_val
    dither = np.random.uniform(-1, 1, sig_float.shape) + np.random.uniform(-1, 1, sig_float.shape)
    quantized = np.round(scaled + dither)
    return np.clip(quantized, -max_val, max_val).astype(np.int16 if target_bits == 16 else np.int32)
```

### Noise Shaping

```python
def noise_shaped_dither(sig_float, target_bits=16, shape='modified_e', sr=SR):
    """Dither with noise shaping — pushes quantization noise to less audible frequencies.
    Shapes: 'flat', 'f_weighted', 'e_weighted', 'modified_e' (POW-R style)."""
    max_val = 2 ** (target_bits - 1) - 1
    scaled = sig_float * max_val
    SHAPES = {
        'flat': [0],
        'f_weighted': [1.0],
        'e_weighted': [1.623, -0.982, 0.109],
        'modified_e': [2.033, -2.165, 1.959, -1.590, 0.6149],
    }
    coeffs = np.array(SHAPES[shape])
    def process_channel(signal):
        out = np.zeros(len(signal))
        error_buf = np.zeros(len(coeffs) + 1)
        for i in range(len(signal)):
            shaped_error = sum(c * error_buf[j] for j, c in enumerate(coeffs) if i-j-1 >= 0)
            dither = np.random.uniform(-1, 1) + np.random.uniform(-1, 1)
            val = signal[i] - shaped_error + dither
            quantized = np.clip(np.round(val), -max_val, max_val)
            error_buf = np.roll(error_buf, 1)
            error_buf[0] = quantized - signal[i]
            out[i] = quantized
        return out
    if sig_float.ndim == 1:
        return process_channel(scaled).astype(np.int16)
    result = np.zeros_like(scaled)
    for ch in range(sig_float.shape[0]):
        result[ch] = process_channel(scaled[ch])
    return result.astype(np.int16)
```

### Professional WAV Export

```python
def export_wav_pro(filename, left, right, sr=SR, bit_depth=16, dither='modified_e'):
    """Professional WAV export with dithering and noise shaping.
    16-bit: apply dither. 24-bit: no dither needed. Only dither once at final export."""
    n = max(len(left), len(right))
    stereo = np.zeros((2, n))
    stereo[0, :len(left)] = left
    stereo[1, :len(right)] = right
    peak = np.max(np.abs(stereo))
    if peak > 0:
        stereo *= (10 ** (-1/20)) / peak  # normalize to -1 dBTP
    if bit_depth == 16 and dither:
        quantized = noise_shaped_dither(stereo, target_bits=16, shape=dither, sr=sr)
        wavfile.write(filename, sr, quantized.T.astype(np.int16))
    elif bit_depth == 24:
        import soundfile as sf
        sf.write(filename, stereo.T, sr, subtype='PCM_24')
    elif bit_depth == 32:
        import soundfile as sf
        sf.write(filename, stereo.T.astype(np.float32), sr, subtype='FLOAT')
```

### Dithering Rules

- Always dither when going float/24-bit → 16-bit
- Never dither at 24-bit or float (enough resolution)
- Only once at the very final export step
- Never re-dither an already-dithered file

---

## Summary: Top 10 Amateur vs Professional Differences

1. Frequency management — carve space per instrument with surgical cuts + gentle boosts
2. Dynamic control — serial, parallel, and multiband compression (not one heavy comp)
3. Reverb discipline — EQ returns (HPF 200-400Hz, LPF 6-10kHz), use pre-delay
4. Low-end management — mono bass below 150-200Hz, sidechain kick/bass
5. Stereo imaging — frequency-dependent width, mid/side processing
6. Saturation — subtle analog-style coloring for warmth and glue
7. Transient control — shape attack/sustain per element
8. Mastering targets — platform-specific LUFS with true peak limiting
9. Automation — EQ/compression/FX change across sections (chorus brighter, verse intimate)
10. Dithering — proper TPDF with noise shaping for 16-bit export
