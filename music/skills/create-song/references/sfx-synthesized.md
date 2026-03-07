# Synthesized Sound Effects

Risers, impacts, whooshes, glitch effects, laser, explosion, tape effects, Doppler, and reverb tricks. All numpy/scipy.

## Risers and Sweeps

```python
def sfx_noise_riser(duration=4.0, start_freq=200, end_freq=16000, sr=SR):
    """Noise sweep riser. Uses overlap-add to avoid pops at block boundaries."""
    n = int(sr * duration)
    noise = np.random.randn(n)
    t = np.linspace(0, 1, n)
    freq_curve = start_freq * (end_freq / start_freq) ** t  # exponential sweep
    block, hop = 2048, 1024
    window = np.hanning(block)
    out = np.zeros(n + block)
    norm = np.zeros(n + block)
    for s in range(0, n - block, hop):
        fc = np.clip(freq_curve[s], 20, sr/2 - 100)
        bw = fc * 0.5
        sos = butter(2, [max(20, fc - bw/2), min(sr/2-100, fc + bw/2)], btype='band', fs=sr, output='sos')
        out[s:s+block] += sosfilt(sos, noise[s:s+block]) * window
        norm[s:s+block] += window
    norm[norm < 1e-10] = 1
    result = (out / norm)[:n] * np.linspace(0.2, 1.0, n) ** 2
    return result / (np.max(np.abs(result)) + 1e-10)

def sfx_pitch_riser(duration=4.0, start_freq=200, end_freq=2000, sr=SR):
    """Sine/saw pitch riser."""
    n = int(sr * duration)
    t = np.linspace(0, 1, n)
    freq_env = start_freq * (end_freq / start_freq) ** t
    phase = 2 * np.pi * np.cumsum(freq_env) / sr
    sig = np.sin(phase) * np.linspace(0.3, 1.0, n) ** 1.5
    fade_n = int(0.003 * sr)
    sig[:fade_n] *= np.linspace(0, 1, fade_n) ** 2
    return sig

def sfx_filter_sweep(sig, start_cutoff=200, end_cutoff=16000, sr=SR):
    """Apply a filter sweep to an existing signal. Block-based for smooth sweep."""
    n = len(sig)
    t = np.linspace(0, 1, n)
    cutoff_env = start_cutoff * (end_cutoff / start_cutoff) ** t
    block = 256
    zi = np.zeros((2, 2))
    out = np.zeros(n)
    for s in range(0, n, block):
        e = min(s + block, n)
        fc = np.clip(cutoff_env[s], 20, sr/2 - 100)
        sos = butter(2, fc, btype='low', fs=sr, output='sos')
        out[s:e], zi = sosfilt(sos, sig[s:e], zi=zi)
    return out
```

## Impacts and Booms

```python
def sfx_impact(duration=1.5, sub_freq=30, sr=SR):
    """Layered impact: sub sine + body pitch drop + noise crack."""
    n = int(sr * duration)
    t = np.arange(n) / sr
    # Sub layer (25-35Hz, long decay)
    sub = np.sin(2 * np.pi * sub_freq * t) * np.exp(-t / 0.8)
    # Body with pitch drop (80->35Hz)
    body_freq = 35 + 45 * np.exp(-t * 8)
    body_phase = 2 * np.pi * np.cumsum(body_freq) / sr
    body = np.sin(body_phase) * np.exp(-t / 0.3)
    # Noise crack
    crack = np.random.randn(n) * np.exp(-t / 0.015)
    sos = butter(2, [50, 4000], btype='band', fs=sr, output='sos')
    crack = sosfilt(sos, crack)
    impact = sub * 0.5 + body * 0.3 + crack * 0.2
    fade_n = int(0.003 * sr)
    impact[:fade_n] *= np.linspace(0, 1, fade_n) ** 2
    return impact / (np.max(np.abs(impact)) + 1e-10)
```

## Whooshes

```python
def sfx_whoosh(duration=0.8, sr=SR):
    """Bandpass noise with parabolic frequency arc and panning envelope."""
    n = int(sr * duration)
    t = np.linspace(0, 1, n)
    noise = np.random.randn(n)
    center = 500 + 6000 * (1 - (2*t - 1)**2)  # Peaks at midpoint
    block, hop = 1024, 512
    window = np.hanning(block)
    out = np.zeros(n + block)
    norm = np.zeros(n + block)
    for s in range(0, n - block, hop):
        fc = np.clip(center[s], 100, sr/2 - 500)
        bw = fc * 0.6
        sos = butter(2, [max(20, fc - bw/2), min(sr/2-100, fc + bw/2)], btype='band', fs=sr, output='sos')
        out[s:s+block] += sosfilt(sos, noise[s:s+block]) * window
        norm[s:s+block] += window
    norm[norm < 1e-10] = 1
    result = (out / norm)[:n] * np.sin(np.pi * t) ** 1.5  # Volume arc
    return result / (np.max(np.abs(result)) + 1e-10)
```

## Glitch Effects

```python
def sfx_buffer_repeat(sig, position, slice_ms=30, repeats=8, decay=0.85, sr=SR):
    """Grab a slice and repeat it with decay. Shorter slice = more frantic."""
    pos = int(position * sr)
    slice_len = int(slice_ms * sr / 1000)
    if pos + slice_len > len(sig):
        return sig
    slc = sig[pos:pos + slice_len].copy()
    fade = int(0.002 * sr)
    if fade > 0 and fade < len(slc):
        slc[:fade] *= np.linspace(0, 1, fade)
        slc[-fade:] *= np.linspace(1, 0, fade)
    output = np.zeros(slice_len * repeats)
    for i in range(repeats):
        start = i * slice_len
        amp = decay ** i
        output[start:start + slice_len] += slc * amp
    return output / (np.max(np.abs(output)) + 1e-10)

def sfx_stutter(sig, bpm, bars=2, sr=SR):
    """Progressive stutter: quarter -> 8th -> 16th -> 32nd over N bars."""
    beat_samples = int(60 * sr / bpm)
    bar_samples = beat_samples * 4
    total = bar_samples * bars
    output = np.zeros(min(total, len(sig)))
    subdivisions = [4, 8, 16, 32]
    beats_per_sub = bars * 4 // len(subdivisions)
    pos = 0
    for sub in subdivisions:
        slice_len = beat_samples * 4 // sub
        for _ in range(beats_per_sub * sub // 4):
            if pos + slice_len > len(output):
                break
            src_start = pos % len(sig)
            src_end = min(src_start + slice_len, len(sig))
            actual = src_end - src_start
            chunk = sig[src_start:src_end].copy()
            fade = min(int(0.002 * sr), actual // 4)
            if fade > 0:
                chunk[:fade] *= np.linspace(0, 1, fade)
                chunk[-fade:] *= np.linspace(1, 0, fade)
            output[pos:pos + actual] = chunk
            pos += slice_len
    return output / (np.max(np.abs(output)) + 1e-10)
```

## Laser / Zap

```python
def sfx_laser(duration=0.15, start_freq=4000, end_freq=200, sr=SR):
    """Fast downward pitch sweep. Layer 3-5 at random params for richness."""
    n = int(sr * duration)
    t = np.arange(n) / sr
    freq_env = end_freq + (start_freq - end_freq) * np.exp(-t / (duration * 0.3))
    phase = 2 * np.pi * np.cumsum(freq_env) / sr
    sig = np.sin(phase) * np.exp(-t / (duration * 0.7))
    fade_n = int(0.002 * sr)
    sig[:fade_n] *= np.linspace(0, 1, fade_n)
    return sig / (np.max(np.abs(sig)) + 1e-10)
```

## Explosion

```python
def sfx_explosion(duration=3.0, sr=SR):
    """Three layers: crack + sub boom + debris."""
    n = int(sr * duration)
    t = np.arange(n) / sr
    # Crack (broadband, 20ms decay)
    crack = np.random.randn(n) * np.exp(-t / 0.02)
    sos_c = butter(2, [100, 8000], btype='band', fs=sr, output='sos')
    crack = sosfilt(sos_c, crack)
    # Sub boom (pitch drop 85->25Hz)
    boom_freq = 25 + 60 * np.exp(-t * 5)
    boom_phase = 2 * np.pi * np.cumsum(boom_freq) / sr
    boom = np.sin(boom_phase) * np.exp(-t / 0.6)
    # Debris (brown noise, long decay)
    debris = np.cumsum(np.random.randn(n))
    debris = debris / (np.max(np.abs(debris)) + 1e-10) * np.exp(-t / 1.5)
    sos_d = butter(2, [30, 2000], btype='band', fs=sr, output='sos')
    debris = sosfilt(sos_d, debris)
    result = crack * 0.3 + boom * 0.5 + debris * 0.2
    fade_n = int(0.003 * sr)
    result[:fade_n] *= np.linspace(0, 1, fade_n) ** 2
    return result / (np.max(np.abs(result)) + 1e-10)
```

## Tape Stop / Start

```python
def sfx_tape_stop(sig, duration=1.0, sr=SR):
    """Decelerate playback to zero. Pitch drops naturally."""
    n = int(duration * sr)
    speed_curve = np.linspace(1, 0, n) ** 2  # Quadratic deceleration
    read_pos = np.cumsum(speed_curve)
    read_pos = np.clip(read_pos, 0, len(sig) - 2)
    idx0 = read_pos.astype(int)
    frac = read_pos - idx0
    return sig[idx0] * (1 - frac) + sig[np.minimum(idx0 + 1, len(sig) - 1)] * frac

def sfx_tape_start(sig, duration=0.5, sr=SR):
    """Accelerate from zero to full speed."""
    n = int(duration * sr)
    speed_curve = np.linspace(0, 1, n) ** 2  # Quadratic acceleration
    read_pos = np.cumsum(speed_curve)
    read_pos = np.clip(read_pos, 0, len(sig) - 2)
    idx0 = read_pos.astype(int)
    frac = read_pos - idx0
    return sig[idx0] * (1 - frac) + sig[np.minimum(idx0 + 1, len(sig) - 1)] * frac
```

## Doppler Effect

```python
def sfx_doppler(sig, speed=30, closest_distance=5, sr=SR):
    """Simulate a sound source passing by. speed in m/s, distance in meters."""
    c = 343.0  # Speed of sound
    n = len(sig)
    t = np.arange(n) / sr
    t_centered = t - t[-1] / 2
    x_pos = speed * t_centered
    distance = np.sqrt(x_pos**2 + closest_distance**2)
    v_radial = speed * x_pos / distance
    doppler_ratio = c / (c + v_radial)
    read_pos = np.cumsum(doppler_ratio)
    read_pos = np.clip(read_pos, 0, len(sig) - 2)
    idx0 = read_pos.astype(int)
    frac = read_pos - idx0
    output = sig[idx0] * (1 - frac) + sig[np.minimum(idx0 + 1, len(sig) - 1)] * frac
    # Amplitude falloff (inverse distance)
    amplitude = closest_distance / distance
    return output * amplitude
```

## Shimmer Reverb

```python
def sfx_shimmer_reverb(sig, iterations=3, decay=0.5, sr=SR):
    """Reverb -> pitch shift up octave -> repeat. Ethereal ambient/post-rock quality."""
    output = sig.copy()
    layer = sig.copy()
    for i in range(iterations):
        # Apply reverb
        layer = freeverb(layer, room=0.9, damp=0.3, wet=0.8, sr=sr)
        # Pitch shift up one octave (resample to half length, then pad)
        half = layer[::2]  # Simple decimation for octave up
        padded = np.zeros(len(layer))
        padded[:len(half)] = half
        layer = padded * decay ** (i + 1)
        output += layer
    return output / (np.max(np.abs(output)) + 1e-10)
```

## Reverse Reverb

```python
def sfx_reverse_reverb(sig, sr=SR):
    """Reverse signal, apply heavy reverb, reverse back. Creates pre-echo buildup."""
    reversed_sig = sig[::-1]
    wet = freeverb(reversed_sig, room=0.9, damp=0.2, wet=0.8, sr=sr)
    return wet[::-1]
```
