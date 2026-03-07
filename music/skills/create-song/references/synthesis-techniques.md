# Advanced Synthesis Techniques

Granular, physical modeling, modal, vector, formant, vocoder, sample playback, waveshaping, and wavetable synthesis. All numpy/scipy.

## Granular Synthesis

Break audio into tiny "grains" (10-100ms), window them, scatter and overlap for textures.

```python
def granular_synthesis(source, grain_size_ms=50, density=20, scatter_ms=10,
                       pitch_shift=1.0, duration=None, sr=SR):
    """Granular texture generator.
    density: grains per second. scatter_ms: random position jitter.
    pitch_shift: 1.0=original, 2.0=octave up, 0.5=octave down."""
    grain_samples = int(grain_size_ms * sr / 1000)
    out_len = int((duration or len(source) / sr) * sr)
    output = np.zeros(out_len)
    window = np.hanning(grain_samples)
    n_grains = int((duration or len(source) / sr) * density)
    for _ in range(n_grains):
        src_pos = np.random.randint(0, max(1, len(source) - grain_samples))
        grain = source[src_pos:src_pos + grain_samples].copy()
        if pitch_shift != 1.0:
            indices = np.linspace(0, len(grain) - 1, int(len(grain) / pitch_shift))
            indices = np.clip(indices, 0, len(grain) - 1)
            idx0 = indices.astype(int)
            frac = indices - idx0
            idx1 = np.minimum(idx0 + 1, len(grain) - 1)
            grain = grain[idx0] * (1 - frac) + grain[idx1] * frac
        win_len = min(len(grain), grain_samples)
        grain[:win_len] *= window[:win_len]
        scatter = int(np.random.uniform(-scatter_ms, scatter_ms) * sr / 1000)
        out_pos = np.random.randint(0, max(1, out_len - len(grain)))
        out_pos = np.clip(out_pos + scatter, 0, out_len - len(grain))
        end = min(out_pos + len(grain), out_len)
        output[out_pos:end] += grain[:end - out_pos]
    return output / (np.max(np.abs(output)) + 1e-10)
```

### Granular Parameter Guide

| Parameter | Low | Medium | High | Effect |
|-----------|-----|--------|------|--------|
| Grain size | 10-20ms | 30-60ms | 80-200ms | Small=glitchy, Large=smooth cloud |
| Density | 5-10/s | 15-30/s | 40-100/s | Sparse=droplets, Dense=continuous |
| Scatter | 0ms | 10-30ms | 50-200ms | None=rhythmic, High=chaotic |
| Pitch rand | 0 | +/-50 cents | +/-1200 cents | None=clean, High=shimmering |

### Granular Freeze

Lock grains to one position with small jitter for sustained ambient pad from a single moment:

```python
def granular_freeze(source, freeze_pos, duration, grain_size_ms=60,
                    density=30, jitter_ms=20, sr=SR):
    freeze_sample = int(freeze_pos * sr)
    grain_samples = int(grain_size_ms * sr / 1000)
    jitter_samples = int(jitter_ms * sr / 1000)
    out_len = int(duration * sr)
    output = np.zeros(out_len)
    window = np.hanning(grain_samples)
    n_grains = int(duration * density)
    for _ in range(n_grains):
        jitter = np.random.randint(-jitter_samples, jitter_samples + 1)
        src_pos = np.clip(freeze_sample + jitter, 0, len(source) - grain_samples)
        grain = source[src_pos:src_pos + grain_samples] * window
        pitch_var = 2 ** (np.random.uniform(-0.1, 0.1))  # +/- ~1 semitone
        if abs(pitch_var - 1.0) > 0.01:
            indices = np.linspace(0, len(grain) - 1, int(len(grain) / pitch_var))
            indices = np.clip(indices, 0, len(grain) - 1)
            idx0 = indices.astype(int); frac = indices - idx0
            grain = grain[idx0] * (1 - frac) + grain[np.minimum(idx0 + 1, len(grain) - 1)] * frac
        out_pos = np.random.randint(0, max(1, out_len - len(grain)))
        end = min(out_pos + len(grain), out_len)
        output[out_pos:end] += grain[:end - out_pos]
    return output / (np.max(np.abs(output)) + 1e-10)
```

## Physical Modeling

### Extended Karplus-Strong (Drums)

Stretch factor randomly inverts samples, breaking periodicity for drum-like tones:

```python
def ks_drum(duration=0.3, freq=80, damping=0.5, stretch_factor=0.5, sr=SR):
    delay_len = max(2, int(sr / freq))
    n = int(sr * duration)
    out = np.zeros(n)
    burst_len = min(delay_len, int(sr * 0.005))
    out[:burst_len] = np.random.uniform(-1, 1, burst_len)
    for i in range(delay_len, n):
        avg = 0.5 * (out[i - delay_len] + out[i - delay_len - 1])
        out[i] = damping * avg if np.random.random() < stretch_factor else damping * (-avg)
    return out / (np.max(np.abs(out)) + 1e-10)
```

### Digital Waveguide (Plucked String)

```python
def waveguide_string(freq, duration=2.0, pluck_pos=0.25, damping=0.998,
                     pickup_pos=0.15, sr=SR):
    """Two counter-propagating delay lines. pluck_pos and pickup_pos shape harmonics.
    pluck_pos near bridge (0.1) = brighter. Near center (0.5) = mellower."""
    delay_len = int(sr / freq)
    n = int(sr * duration)
    forward = np.zeros(n)
    backward = np.zeros(n)
    pluck_idx = max(1, int(pluck_pos * delay_len))
    excitation = np.zeros(delay_len)
    for i in range(delay_len):
        excitation[i] = i / pluck_idx if i <= pluck_idx else (delay_len - i) / (delay_len - pluck_idx)
    forward[:delay_len] = excitation * 0.5
    backward[:delay_len] = excitation * 0.5
    lp_state = 0.0
    for i in range(delay_len, n):
        lp_state = damping * (0.5 * forward[i - delay_len] + 0.5 * lp_state)
        backward[i] = -lp_state
        backward_delayed = backward[i - delay_len] if i >= delay_len else 0
        forward[i] = -backward_delayed * damping
    pickup_delay = int(pickup_pos * delay_len)
    output = np.zeros(n)
    for i in range(n):
        f = forward[i - pickup_delay] if i >= pickup_delay else 0
        b = backward[i - (delay_len - pickup_delay)] if i >= (delay_len - pickup_delay) else 0
        output[i] = f + b
    return output / (np.max(np.abs(output)) + 1e-10)
```

### Bowed String Model

```python
def bowed_string(freq, duration=2.0, bow_velocity=0.2, bow_pressure=0.5, sr=SR):
    """Stick-slip friction model. bow_velocity = dynamics, bow_pressure = brightness."""
    delay_len = max(2, int(sr / freq))
    n = int(sr * duration)
    output = np.zeros(n)
    delay_line = np.zeros(delay_len + 1)
    lp_state = 0.0
    for i in range(n):
        idx = i % delay_len
        string_vel = delay_line[idx]
        v_diff = bow_velocity - string_vel
        friction = bow_pressure * 50 * v_diff * np.exp(-bow_pressure * 50 * v_diff**2 + 0.5)
        new_val = string_vel + friction * 0.1
        lp_state = 0.5 * new_val + 0.5 * lp_state
        delay_line[idx] = lp_state * 0.999
        output[i] = lp_state
    env = np.ones(n)
    att = int(0.1 * sr)
    env[:att] = np.linspace(0, 1, att) ** 2
    env[-int(0.05 * sr):] = np.linspace(1, 0, int(0.05 * sr))
    output *= env
    return output / (np.max(np.abs(output)) + 1e-10)
```

### Single Reed Model (Clarinet/Sax)

```python
def reed_instrument(freq, duration=2.0, breath_pressure=0.5, reed_stiffness=0.5, sr=SR):
    """Cylindrical bore = odd harmonics (clarinet). Conical bore = all harmonics (sax)."""
    delay_len = max(2, int(sr / (2 * freq)))
    n = int(sr * duration)
    output = np.zeros(n)
    bore = np.zeros(delay_len + 1)
    breath = np.ones(n) * breath_pressure
    attack = int(0.05 * sr)
    breath[:attack] = np.linspace(0, breath_pressure, attack)
    breath += np.random.randn(n) * 0.01
    y_prev = 0.0
    for i in range(n):
        idx = i % delay_len
        reflected = -bore[idx] * 0.95
        pressure_diff = breath[i] - reflected
        reed_opening = max(0, 1.0 - reed_stiffness * pressure_diff)
        flow = pressure_diff * reed_opening ** 2
        bore[idx] = flow
        y_prev = 0.7 * flow + 0.3 * y_prev
        output[i] = y_prev
    return output / (np.max(np.abs(output)) + 1e-10)
```

## Vector Synthesis

4 waveforms at corners of a 2D plane, blended with LFO-driven position:

```python
def vector_synth(freq, duration, wave_funcs, x_lfo_rate=0.2, y_lfo_rate=0.13, sr=SR):
    """wave_funcs: [top_left, top_right, bottom_left, bottom_right] -- each is (freq, dur, sr) -> array.
    Irrational LFO ratio = non-repeating pattern."""
    n = int(sr * duration)
    t = np.arange(n) / sr
    waves = [f(freq, duration, sr)[:n] for f in wave_funcs]
    x = 0.5 + 0.5 * np.sin(2 * np.pi * x_lfo_rate * t)
    y = 0.5 + 0.5 * np.sin(2 * np.pi * y_lfo_rate * t)
    top = waves[0] * (1 - x) + waves[1] * x
    bottom = waves[2] * (1 - x) + waves[3] * x
    return top * (1 - y) + bottom * y
```

## Modal Synthesis (Resonator Banks)

Model objects as damped resonant modes. Each mode = bandpass filter at a specific frequency.

```python
MODAL_PRESETS = {
    'bell':       {'ratios': [1,2,2.37,2.97,3.156,4.23,5.404,6.27],
                   'amps':   [1,.6,.5,.4,.25,.15,.1,.07],
                   'decays': [3,2.5,2,1.5,1.2,.8,.6,.4]},
    'wine_glass': {'ratios': [1,2.32,4.15,6.45], 'amps': [1,.4,.15,.05], 'decays': [5,4,3,2]},
    'marimba':    {'ratios': [1,3.98,9], 'amps': [1,.3,.05], 'decays': [.8,.3,.1]},
    'gong':       {'ratios': [1,1.48,1.55,2.06,2.67,3.15,3.98,4.67,5.31],
                   'amps':   [1,.8,.7,.6,.4,.3,.2,.15,.1],
                   'decays': [6,5,4.5,4,3,2.5,2,1.5,1]},
    'steel_drum': {'ratios': [1,2,3,4.01,4.53], 'amps': [1,.5,.3,.2,.1], 'decays': [1.5,1,.7,.5,.3]},
    'wood_block': {'ratios': [1,2.57,4.53,6.6], 'amps': [1,.4,.15,.05], 'decays': [.15,.08,.05,.03]},
    'tubular_bell':{'ratios': [1,2.76,4.07,5.16,5.93,6.35,6.91,8.54],
                    'amps':   [1,.5,.4,.3,.2,.15,.1,.05],
                    'decays': [5,4,3.5,3,2.5,2,1.5,1]},
}

def modal_synth(freq, preset, duration=3.0, sr=SR):
    """Excite resonator bank with noise burst."""
    p = MODAL_PRESETS[preset]
    n = int(sr * duration)
    excitation = np.zeros(n)
    burst = int(0.003 * sr)
    excitation[:burst] = np.random.randn(burst) * np.exp(-np.arange(burst) / (burst * 0.3))
    output = np.zeros(n)
    for ratio, amp, decay in zip(p['ratios'], p['amps'], p['decays']):
        f = freq * ratio
        if f >= SR / 2: continue
        Q = max(f * decay * np.pi, 0.5)
        w0 = 2 * np.pi * f / sr
        alpha = np.sin(w0) / (2 * Q)
        sos = np.array([[alpha/(1+alpha), 0, -alpha/(1+alpha), 1, -2*np.cos(w0)/(1+alpha), (1-alpha)/(1+alpha)]])
        output += amp * sosfilt(sos, excitation)
    return output / (np.max(np.abs(output)) + 1e-10)
```

## Vocoder

```python
def vocoder(carrier, modulator, n_bands=16, freq_range=(80, 8000), sr=SR):
    """Classic vocoder. carrier=synth, modulator=voice. 8 bands=robotic, 16+=intelligible."""
    n = min(len(carrier), len(modulator))
    output = np.zeros(n)
    band_edges = np.exp(np.linspace(np.log(freq_range[0]), np.log(freq_range[1]), n_bands + 1))
    for i in range(n_bands):
        low, high = band_edges[i], min(band_edges[i + 1], sr / 2 - 100)
        if low >= high: continue
        sos = butter(2, [low, high], btype='band', fs=sr, output='sos')
        mod_band = sosfilt(sos, modulator[:n])
        car_band = sosfilt(sos, carrier[:n])
        envelope = np.abs(mod_band)
        env_sos = butter(1, 50, btype='low', fs=sr, output='sos')
        envelope = sosfilt(env_sos, envelope)
        output += car_band * envelope
    return output / (np.max(np.abs(output)) + 1e-10)
```

## Multimode State Variable Filter

```python
def svf_multimode(sig, cutoff, Q=1.0, mode='low', sr=SR):
    """Modes: 'low', 'high', 'band', 'notch'. At Q > ~10, self-oscillates."""
    n = len(sig)
    out = np.zeros(n)
    low = band = high = 0.0
    q_inv = 1.0 / max(Q, 0.1)
    for i in range(n):
        fc = cutoff[i] if hasattr(cutoff, '__len__') else cutoff
        f = 2 * np.sin(np.pi * min(fc, sr * 0.45) / sr)
        high = sig[i] - low - q_inv * band
        band += f * high
        low += f * band
        if mode == 'low': out[i] = low
        elif mode == 'high': out[i] = high
        elif mode == 'band': out[i] = band
        elif mode == 'notch': out[i] = high + low
    return out
```

## Sample Playback

```python
def load_sample(filepath, sr=SR):
    """Load audio file, convert to float64 mono, resample if needed."""
    import soundfile as sf
    data, file_sr = sf.read(filepath)
    if data.ndim > 1:
        data = data.mean(axis=1)
    if file_sr != sr:
        ratio = sr / file_sr
        n_new = int(len(data) * ratio)
        indices = np.linspace(0, len(data) - 1, n_new)
        idx0 = indices.astype(int); frac = indices - idx0
        data = data[idx0] * (1 - frac) + data[np.minimum(idx0 + 1, len(data) - 1)] * frac
    return data

def play_sample(sample, original_freq, target_freq, duration=None, sr=SR):
    """Pitch-shift sample by resampling. Works well within +/- 7 semitones."""
    ratio = original_freq / target_freq
    n_out = int(len(sample) * ratio) if duration is None else int(sr * duration)
    n_out = min(n_out, int(len(sample) * ratio))
    indices = np.arange(n_out) / ratio
    indices = np.clip(indices, 0, len(sample) - 1.001)
    idx0 = indices.astype(int); frac = indices - idx0
    return sample[idx0] * (1 - frac) + sample[np.minimum(idx0 + 1, len(sample) - 1)] * frac
```

## Advanced Formant Synthesis

5-formant model with vowel data for male, female, and morphing:

```python
VOWEL_FORMANTS = {
    'a':  {'F': [800,1150,2800,3500,4950], 'BW': [80,70,100,120,120], 'G': [1,.5,.3,.15,.1]},
    'e':  {'F': [400,1600,2700,3300,4950], 'BW': [70,80,100,120,120], 'G': [1,.5,.25,.15,.05]},
    'i':  {'F': [270,2300,3000,3700,4950], 'BW': [60,90,100,120,120], 'G': [1,.4,.2,.15,.05]},
    'o':  {'F': [500,700,2800,3500,4950],  'BW': [70,80,100,120,120], 'G': [1,.6,.2,.15,.05]},
    'u':  {'F': [300,640,2700,3500,4950],  'BW': [60,70,100,120,120], 'G': [1,.5,.15,.1,.05]},
}
# Female formants: multiply all F values by 1.2
# Child formants: multiply all F values by 1.4

def formant_synth(freq, duration, vowel='a', sr=SR):
    """Source-filter vocal synthesis. Glottal saw + formant resonators."""
    t = np.arange(int(sr * duration)) / sr
    source = 2.0 * ((freq * t) % 1.0) - 1.0
    sos = butter(1, min(freq * 6, sr/2 - 100), btype='low', fs=sr, output='sos')
    source = sosfilt(sos, source) + np.random.randn(len(t)) * 0.03
    v = VOWEL_FORMANTS[vowel]
    output = np.zeros(len(t))
    for fc, bw, gain in zip(v['F'], v['BW'], v['G']):
        if fc >= sr / 2: continue
        Q = fc / max(bw, 1)
        w0 = 2 * np.pi * fc / sr
        alpha = np.sin(w0) / (2 * Q)
        sos_f = np.array([[alpha/(1+alpha), 0, -alpha/(1+alpha), 1, -2*np.cos(w0)/(1+alpha), (1-alpha)/(1+alpha)]])
        output += gain * sosfilt(sos_f, source)
    return output
```

## Noise Types

```python
def colored_noise(duration, color='pink', sr=SR):
    """Generate colored noise. pink=-3dB/oct, brown=-6dB/oct, blue=+3dB/oct, violet=+6dB/oct."""
    n = int(sr * duration)
    white = np.random.randn(n)
    spectrum = np.fft.rfft(white)
    freqs = np.fft.rfftfreq(n, 1.0 / sr)
    freqs[0] = 1
    if color == 'pink':     spectrum *= 1.0 / np.sqrt(freqs)
    elif color == 'brown':  spectrum *= 1.0 / freqs
    elif color == 'blue':   spectrum *= np.sqrt(freqs)
    elif color == 'violet': spectrum *= freqs
    sig = np.fft.irfft(spectrum, n=n)
    return sig / (np.max(np.abs(sig)) + 1e-10)

def perlin_noise_1d(n_samples, scale=100, octaves=4, persistence=0.5):
    """Smooth organic noise for modulation. Better than sine LFOs for natural feel."""
    output = np.zeros(n_samples)
    for octave in range(octaves):
        freq = 2 ** octave
        amp = persistence ** octave
        n_points = max(2, n_samples // (scale // max(freq, 1)) + 2)
        grid = np.random.randn(n_points)
        x = np.linspace(0, n_points - 1, n_samples)
        idx = np.clip(x.astype(int), 0, len(grid) - 2)
        frac = x - idx
        t = frac * frac * (3 - 2 * frac)  # smoothstep
        output += (grid[idx] * (1 - t) + grid[idx + 1] * t) * amp
    output -= np.mean(output)
    return output / (np.max(np.abs(output)) + 1e-10)
```

## Convolution Reverb

```python
from scipy.signal import fftconvolve

def convolution_reverb(sig, ir, wet=0.3, sr=SR):
    """Apply convolution reverb using an impulse response.
    Use fftconvolve (O(N log N)) -- NEVER np.convolve (O(N*M)).
    Always EQ the reverb return: HPF 200Hz, LPF 8kHz."""
    ir = ir / (np.max(np.abs(ir)) + 1e-10)
    reverb = fftconvolve(sig, ir, mode='full')[:len(sig)]
    sos_hp = butter(2, 200, btype='high', fs=sr, output='sos')
    sos_lp = butter(2, 8000, btype='low', fs=sr, output='sos')
    reverb = sosfilt(sos_lp, sosfilt(sos_hp, reverb))
    return sig * (1 - wet) + reverb * wet

def generate_synthetic_ir(duration=2.0, decay_time=1.5, damping=0.7, sr=SR):
    """Generate a synthetic impulse response when no IR file is available."""
    n = int(sr * duration)
    ir = np.zeros(n)
    for _ in range(15):
        pos = int(np.random.uniform(0.005, 0.08) * sr)
        ir[pos] += np.random.uniform(0.3, 0.8) * np.exp(-pos / (sr * 0.05)) * np.random.choice([-1, 1])
    late_start = int(0.08 * sr)
    late = np.random.randn(n - late_start) * np.exp(-np.arange(n - late_start) / (sr * decay_time))
    sos = butter(1, 2000 + (1 - damping) * 8000, btype='low', fs=sr, output='sos')
    ir[late_start:] += sosfilt(sos, late) * 0.3
    return ir / (np.max(np.abs(ir)) + 1e-10)
```

## Chebyshev Waveshaping

Precise harmonic generation using polynomial waveshaping:

```python
def chebyshev_waveshaper(sig, harmonic_amps):
    """harmonic_amps[n] = amplitude of (n+1)th harmonic.
    T_n(cos(x)) = cos(n*x) -- produces pure harmonics from sine input."""
    def cheb(n, x):
        if n == 0: return np.ones_like(x)
        if n == 1: return x
        t_prev, t_curr = np.ones_like(x), x
        for _ in range(2, n + 1):
            t_next = 2 * x * t_curr - t_prev
            t_prev, t_curr = t_curr, t_next
        return t_curr
    x = np.clip(sig, -1, 1)
    output = sum(amp * cheb(n + 1, x) for n, amp in enumerate(harmonic_amps))
    return output / (np.max(np.abs(output)) + 1e-10)

# Tube warmth (even harmonics): chebyshev_waveshaper(sig, [1, 0.3, 0.1, 0.05, 0.02])
# Odd distortion (square-like): chebyshev_waveshaper(sig, [1, 0, 0.4, 0, 0.15, 0, 0.05])
```

## Ring Modulation

```python
def ring_mod(sig, mod_freq, depth=1.0, sr=SR):
    """Low mod_freq (1-20Hz) = tremolo. High (50-2000Hz) = metallic/bell tones."""
    t = np.arange(len(sig)) / sr
    return sig * (1 - depth + depth * np.sin(2 * np.pi * mod_freq * t))
```

## Wavetable Morphing (Vectorized)

```python
def wavetable_osc(freq, duration, wavetables, morph_env, sr=SR):
    """Morph between wavetables over time.
    wavetables: list of 1D arrays (same length, one cycle each).
    morph_env: array of morph positions (0 to len(wavetables)-1)."""
    n = int(sr * duration)
    phase = np.cumsum(np.full(n, freq / sr)) % 1.0
    table_size = len(wavetables[0])
    wt = np.array(wavetables)
    morph = np.clip(morph_env[:n], 0, len(wavetables) - 1 - 1e-9)
    lo = morph.astype(int); hi = lo + 1; fm = morph - lo
    pos = phase * table_size
    ti = pos.astype(int) % table_size
    ti_next = (ti + 1) % table_size
    ft = pos - np.floor(pos)
    s_lo = (1 - ft) * wt[lo, ti] + ft * wt[lo, ti_next]
    s_hi = (1 - ft) * wt[hi, ti] + ft * wt[hi, ti_next]
    return (1 - fm) * s_lo + fm * s_hi
```
