# Effects Processing

Audio effects for spatial, dynamic, and timbral processing. All use numpy/scipy only.

## Freeverb (Primary Reverb)

Use Freeverb for ALL reverb -- never random delay taps (sounds metallic).

```python
def freeverb(sig, room=0.84, damp=0.2, wet=0.3, sr=SR):
    """Freeverb: 8 lowpass-feedback comb filters -> 4 series allpass filters.
    Process MONO signal, then pan to stereo after.
    """
    scale = sr / 44100.0
    comb_delays = [int(d * scale) for d in [1116, 1188, 1277, 1356, 1422, 1491, 1557, 1617]]
    allpass_delays = [int(d * scale) for d in [556, 441, 341, 225]]

    feedback = room * 0.98
    damp1 = damp
    damp2 = 1.0 - damp

    n = len(sig) + int(sr * 2)  # extra for tail
    padded = np.zeros(n)
    padded[:len(sig)] = sig

    # Parallel comb filters with lowpass in feedback path
    comb_sum = np.zeros(n)
    for delay in comb_delays:
        buf = np.zeros(n)
        fstore = 0.0
        for i in range(n):
            delayed = buf[i - delay] if i >= delay else 0.0
            fstore = delayed * damp2 + fstore * damp1  # one-pole LP
            buf[i] = padded[i] + fstore * feedback
        for i in range(n):
            comb_sum[i] += (buf[i - delay] if i >= delay else 0.0)
    comb_sum /= len(comb_delays)

    # Series allpass filters for diffusion
    result = comb_sum
    for delay in allpass_delays:
        out = np.zeros(n)
        for i in range(n):
            d = out[i - delay] if i >= delay else 0.0
            s = result[i - delay] if i >= delay else 0.0
            out[i] = -0.5 * result[i] + s + 0.5 * d
        result = out

    result = result[:len(sig)]
    return (1 - wet) * sig + wet * result
```

### Reverb Guidelines

| Context | Room Size | Damping | Wet Mix | Notes |
|---------|-----------|---------|---------|-------|
| Tight room | 0.3-0.5 | 0.5-0.7 | 0.1-0.2 | Short, controlled |
| Live room | 0.6-0.75 | 0.3-0.5 | 0.15-0.25 | Natural instruments |
| Hall | 0.8-0.9 | 0.2-0.3 | 0.2-0.35 | Orchestral, ambient |
| Cathedral | 0.95+ | 0.1-0.2 | 0.3-0.5 | Huge, ethereal |
| Plate (approx) | 0.7-0.85 | 0.4-0.6 | 0.2-0.3 | Bright, smooth |

**Critical rules:**
- Bass and kick get NO reverb (muddies low end)
- EQ reverb returns: HPF 200-400Hz, LPF 6-10kHz on all reverb sends
- Every melodic/harmonic element goes through reverb and/or delay (dry synths sound cheap)

## Delay

```python
def delay(sig, ms=375, feedback=0.5, mix=0.3, sr=SR):
    """Simple delay with feedback. 375ms = dotted 8th at 120 BPM."""
    d = int(sr * ms / 1000)
    out = np.copy(sig)
    tail = int(feedback * 10 * d)
    out = np.pad(out, (0, tail))
    for i in range(d, len(out)):
        out[i] += feedback * out[i - d]
    result = np.zeros(len(out))
    result[:len(sig)] = sig * (1 - mix)
    result += out * mix
    return result[:len(sig)]
```

### Ping-Pong Delay

```python
def ping_pong_delay(sig, ms=375, feedback=0.5, mix=0.3, sr=SR):
    """Stereo ping-pong delay. Returns (left, right)."""
    d = int(sr * ms / 1000)
    n = len(sig) + d * 20
    left = np.zeros(n)
    right = np.zeros(n)
    left[:len(sig)] = sig
    for i in range(d, n):
        right[i] += feedback * left[i - d]
        left[i] += feedback * right[i - d]
    L = np.zeros(n); L[:len(sig)] = sig * (1 - mix)
    R = np.zeros(n); R[:len(sig)] = sig * (1 - mix)
    L += left * mix; R += right * mix
    return L[:len(sig)], R[:len(sig)]
```

### Tape Delay

```python
def tape_delay(sig, ms=375, feedback=0.5, wow_freq=0.3, wow_depth=0.002,
               flutter_freq=6.0, flutter_depth=0.0005, sr=SR):
    """Tape delay with wow/flutter modulation. Uses linear interpolation (never integer indexing)."""
    d = int(sr * ms / 1000)
    n = len(sig) + d * 15
    out = np.zeros(n)
    out[:len(sig)] = sig
    t = np.arange(n) / sr
    mod = wow_depth * sr * np.sin(2*np.pi*wow_freq*t) + flutter_depth * sr * np.sin(2*np.pi*flutter_freq*t)
    for i in range(d, n):
        read_pos = i - d + mod[i]
        idx = int(read_pos)
        frac = read_pos - idx
        if 0 <= idx < n - 1:
            out[i] += feedback * (out[idx]*(1-frac) + out[idx+1]*frac)
    # Tape darkening
    sos = butter(1, 4000, btype='low', fs=sr, output='sos')
    wet = sosfilt(sos, out)
    result = np.zeros(n)
    result[:len(sig)] = sig
    return (result + 0.3 * wet)[:len(sig)]
```

### Delay Time Reference

| Subdivision | Formula | At 120 BPM | At 140 BPM |
|-------------|---------|------------|------------|
| Quarter note | 60000/BPM | 500ms | 428ms |
| Dotted eighth | 60000/BPM * 0.75 | 375ms | 321ms |
| Eighth note | 60000/BPM * 0.5 | 250ms | 214ms |
| Triplet quarter | 60000/BPM * 2/3 | 333ms | 286ms |
| Sixteenth | 60000/BPM * 0.25 | 125ms | 107ms |

## Chorus

```python
def chorus(sig, rate=1.5, depth_ms=3.0, mix=0.5, voices=3, sr=SR):
    """Chorus: LFO-modulated short delay. MUST use linear interpolation."""
    n = len(sig)
    out = np.copy(sig).astype(np.float64)
    t = np.arange(n) / sr
    depth = depth_ms * sr / 1000
    base_delay = int(depth * 2)
    for v in range(voices):
        phase_off = 2 * np.pi * v / voices
        lfo = depth * np.sin(2*np.pi*rate*t + phase_off)
        voice = np.zeros(n)
        for i in range(base_delay + int(depth) + 1, n):
            read_pos = i - base_delay - lfo[i]
            idx = int(read_pos)
            frac = read_pos - idx
            if 0 <= idx < n - 1:
                voice[i] = sig[idx]*(1-frac) + sig[idx+1]*frac  # linear interp!
        out += voice / voices
    return out * mix + sig * (1 - mix)
```

## Phaser

```python
def phaser(sig, rate=0.5, depth=0.7, stages=6, sr=SR):
    """Phaser: cascade of allpass filters with LFO-swept frequency.
    stages: 4/6/8/12 typical. More = more notches."""
    n = len(sig)
    t = np.arange(n) / sr
    lfo = 0.5 + depth * 0.5 * np.sin(2*np.pi*rate*t)
    freqs = 200 * (20 ** lfo)  # log sweep 200-4000 Hz
    out = np.copy(sig)
    for _ in range(stages):
        ap = np.zeros(n)
        y1, x1 = 0.0, 0.0
        for i in range(n):
            c = (np.tan(np.pi * freqs[i] / sr) - 1) / (np.tan(np.pi * freqs[i] / sr) + 1)
            ap[i] = c * out[i] + x1 - c * y1
            x1 = out[i]; y1 = ap[i]
        out = ap
    return sig + out  # sum creates notches
```

**Flanger vs Phaser:**
- Flanger: very short modulated delay (0.1-10ms), evenly-spaced notches, jet-sweep sound
- Phaser: allpass filter cascade, non-harmonically-related notches, gentler/organic sweep

## Flanger

```python
def flanger(sig, rate=0.25, depth_ms=2.0, feedback=0.7, sr=SR):
    """Flanger: very short modulated delay with feedback."""
    n = len(sig)
    out = np.zeros(n)
    t = np.arange(n) / sr
    depth = depth_ms * sr / 1000
    base = depth + 1
    lfo = depth * np.sin(2*np.pi*rate*t)
    for i in range(int(base + depth) + 2, n):
        d = base + lfo[i]
        rp = i - d
        idx = int(rp); frac = rp - idx
        if 0 <= idx < n - 1:
            out[i] = sig[i] + feedback * (sig[idx]*(1-frac) + sig[idx+1]*frac)
        else:
            out[i] = sig[i]
    return out
```

## Dynamics: Compressor

```python
def compressor(sig, threshold_db=-20, ratio=4.0, attack_ms=10, release_ms=100,
               knee_db=6, makeup_db=0, sr=SR):
    """Dynamic range compressor with soft knee and RMS detection."""
    n = len(sig)
    attack_c = np.exp(-1.0 / (attack_ms * sr / 1000))
    release_c = np.exp(-1.0 / (release_ms * sr / 1000))
    threshold = 10 ** (threshold_db / 20)
    makeup = 10 ** (makeup_db / 20)

    # RMS envelope (~10ms window)
    window = int(sr * 0.01)
    level = np.sqrt(np.convolve(sig**2, np.ones(window)/window, mode='same'))

    env = 0.0
    gr = np.ones(n)
    for i in range(n):
        if level[i] > env:
            env = attack_c * env + (1 - attack_c) * level[i]
        else:
            env = release_c * env + (1 - release_c) * level[i]
        if env <= 0: continue
        env_db = 20 * np.log10(max(env, 1e-10))
        over_db = env_db - threshold_db
        if knee_db > 0 and -knee_db/2 < over_db < knee_db/2:
            over_db = (over_db + knee_db/2)**2 / (2 * knee_db)
        if over_db > 0:
            gr[i] = 10 ** (-(over_db * (1 - 1/ratio)) / 20)
    return sig * gr * makeup
```

### Compressor Presets

| Context | Threshold | Ratio | Attack | Release | Notes |
|---------|-----------|-------|--------|---------|-------|
| Per-track gentle | -20 dB | 2:1 | 10ms | 100ms | Tame peaks |
| Drums | -15 dB | 4:1 | 0.5-5ms | 50-100ms | Catch transients |
| Bass | -18 dB | 3:1 | 5-15ms | 100-200ms | Even out dynamics |
| Bus glue | -12 dB | 2:1 | 20-30ms | 150-300ms | Gentle on mix bus |
| Limiter | -1 dB | 20:1+ | 0.1ms | 50ms | Brickwall |
| Parallel (NY) | -25 dB | 8:1 | 1ms | 100ms | Mix compressed with dry |

## Sidechain Compressor

```python
def sidechain(sig, trigger, threshold_db=-20, ratio=10.0,
              attack_ms=1, release_ms=150, sr=SR):
    """Sidechain compression: sig is ducked by trigger signal (e.g. kick).
    Classic EDM pumping: short attack (1ms), longer release (100-300ms)."""
    ac = np.exp(-1.0 / (attack_ms * sr / 1000))
    rc = np.exp(-1.0 / (release_ms * sr / 1000))
    thr = 10 ** (threshold_db / 20)
    n = min(len(sig), len(trigger))
    out = np.zeros(n)
    env = 0.0
    for i in range(n):
        sc = abs(trigger[i])
        env = ac*env + (1-ac)*sc if sc > env else rc*env + (1-rc)*sc
        gain = thr * (env/thr)**(1/ratio - 1) if env > thr else 1.0
        out[i] = sig[i] * gain
    return out
```

## Limiter

```python
def limiter(sig, threshold_db=-1.0, release_ms=50, sr=SR):
    """Brickwall limiter. Use as final safety on master chain."""
    return compressor(sig, threshold_db=threshold_db, ratio=100.0,
                      attack_ms=0.1, release_ms=release_ms, makeup_db=0, sr=sr)
```

## Distortion & Saturation

```python
def soft_clip(sig, drive=2.0):
    """Soft clipping via tanh. Warm, tube-like saturation."""
    return np.tanh(sig * drive)

def hard_clip(sig, threshold=0.8):
    """Hard clipping. Aggressive distortion."""
    return np.clip(sig, -threshold, threshold)

def tape_saturate(sig, drive=1.5):
    """Tape-style saturation: asymmetric soft clip + even harmonics."""
    driven = sig * drive
    pos = np.tanh(driven * 0.8)
    neg = np.tanh(driven * 1.2)
    return np.where(driven >= 0, pos, neg)

def waveshaper(sig, amount=0.7):
    """Waveshaper distortion for tube emulation."""
    abs_x = np.abs(sig)
    return sig * (abs_x + amount) / (sig**2 + (amount - 1)*abs_x + 1)

def bitcrusher(sig, bits=8, sample_reduce=4):
    """Bitcrusher: reduce bit depth and sample rate.
    bits: 1-16 (8=classic lo-fi, 4=very crunchy).
    sample_reduce: hold every Nth sample (4=quarter rate)."""
    levels = 2 ** bits
    crushed = np.round(sig * levels / 2) / (levels / 2)
    if sample_reduce > 1:
        for i in range(len(crushed)):
            if i % sample_reduce != 0:
                crushed[i] = crushed[i - (i % sample_reduce)]
    return crushed
```

## Lo-Fi Effects

```python
def vinyl_crackle(duration, density=15, sr=SR):
    """Vinyl crackle: random impulses + dusty noise floor + rumble.
    density: crackles per second (~5-30 for realistic vinyl).
    Pops MUST use Hann window envelopes, min 5ms, LP at 3kHz, amplitude < 0.015."""
    n = int(sr * duration)
    crackle = np.zeros(n)
    n_crackles = int(duration * density)
    for _ in range(n_crackles):
        pos = np.random.randint(0, n)
        length = max(int(0.005 * sr), np.random.randint(5, 50))  # min 5ms
        amp = np.random.uniform(0.005, 0.015)
        if pos + length < n:
            window = 0.5 - 0.5 * np.cos(np.linspace(0, 2*np.pi, length))  # Hann
            crackle[pos:pos+length] += amp * window * (1 if np.random.random() > 0.5 else -1)
    # Lowpass the crackle at 3kHz
    sos = butter(2, 3000, btype='low', fs=sr, output='sos')
    crackle = sosfilt(sos, crackle)
    # Dusty noise floor
    dust = np.random.randn(n) * 0.01
    sos_d = butter(2, [200, 3000], btype='band', fs=sr, output='sos')
    dust = sosfilt(sos_d, dust)
    # Low rumble
    rumble = np.cumsum(np.random.randn(n)); rumble -= np.mean(rumble)
    rumble = rumble / max(1e-10, np.max(np.abs(rumble))) * 0.02
    sos_r = butter(2, 80, btype='low', fs=sr, output='sos')
    rumble = sosfilt(sos_r, rumble)
    return crackle + dust + rumble

def tape_hiss(duration, level=0.02, sr=SR):
    """Tape hiss: shaped noise with emphasis around 4-6 kHz."""
    hiss = np.random.randn(int(sr * duration)) * level
    sos = butter(2, [2000, 12000], btype='band', fs=sr, output='sos')
    return sosfilt(sos, hiss)

def tape_wobble(sig, rate=0.5, depth_cents=8, sr=SR):
    """Tape wow/flutter: slow pitch modulation. Apply via resampling or phase mod."""
    t = np.arange(len(sig)) / sr
    mod = depth_cents / 1200.0 * np.sin(2*np.pi*rate*t)
    indices = np.arange(len(sig)) + mod * sr
    indices = np.clip(indices, 0, len(sig) - 2)
    idx0 = indices.astype(int)
    frac = indices - idx0
    return sig[idx0] * (1 - frac) + sig[np.minimum(idx0 + 1, len(sig) - 1)] * frac
```

## EQ / Shelving

```python
def peak_eq(sig, freq, gain_db, Q=1.0, sr=SR):
    """Parametric peak EQ. gain_db: boost/cut at freq."""
    from scipy.signal import iirpeak
    # Manual biquad implementation
    A = 10 ** (gain_db / 40)
    w0 = 2 * np.pi * freq / sr
    alpha = np.sin(w0) / (2 * Q)
    b0 = 1 + alpha * A; b1 = -2 * np.cos(w0); b2 = 1 - alpha * A
    a0 = 1 + alpha / A; a1 = -2 * np.cos(w0); a2 = 1 - alpha / A
    sos = np.array([[b0/a0, b1/a0, b2/a0, 1.0, a1/a0, a2/a0]])
    return sosfilt(sos, sig)

def high_shelf(sig, freq, gain_db, sr=SR):
    """High shelf filter. Boost/cut all frequencies above freq."""
    A = 10 ** (gain_db / 40)
    w0 = 2 * np.pi * freq / sr
    alpha = np.sin(w0) / 2 * np.sqrt(2)
    cos_w0 = np.cos(w0)
    b0 = A*((A+1) + (A-1)*cos_w0 + 2*np.sqrt(A)*alpha)
    b1 = -2*A*((A-1) + (A+1)*cos_w0)
    b2 = A*((A+1) + (A-1)*cos_w0 - 2*np.sqrt(A)*alpha)
    a0 = (A+1) - (A-1)*cos_w0 + 2*np.sqrt(A)*alpha
    a1 = 2*((A-1) - (A+1)*cos_w0)
    a2 = (A+1) - (A-1)*cos_w0 - 2*np.sqrt(A)*alpha
    sos = np.array([[b0/a0, b1/a0, b2/a0, 1.0, a1/a0, a2/a0]])
    return sosfilt(sos, sig)

def low_shelf(sig, freq, gain_db, sr=SR):
    """Low shelf filter. Boost/cut all frequencies below freq."""
    A = 10 ** (gain_db / 40)
    w0 = 2 * np.pi * freq / sr
    alpha = np.sin(w0) / 2 * np.sqrt(2)
    cos_w0 = np.cos(w0)
    b0 = A*((A+1) - (A-1)*cos_w0 + 2*np.sqrt(A)*alpha)
    b1 = 2*A*((A-1) - (A+1)*cos_w0)
    b2 = A*((A+1) - (A-1)*cos_w0 - 2*np.sqrt(A)*alpha)
    a0 = (A+1) + (A-1)*cos_w0 + 2*np.sqrt(A)*alpha
    a1 = -2*((A-1) + (A+1)*cos_w0)
    a2 = (A+1) + (A-1)*cos_w0 - 2*np.sqrt(A)*alpha
    sos = np.array([[b0/a0, b1/a0, b2/a0, 1.0, a1/a0, a2/a0]])
    return sosfilt(sos, sig)
```

## Stereo Width

```python
def stereo_width(left, right, width=1.5):
    """Adjust stereo width. width: 0=mono, 1=unchanged, 2=exaggerated.
    Keep sub-150Hz centered (width the mid-side difference only above that)."""
    mid = (left + right) * 0.5
    side = (left - right) * 0.5
    # Widen the side signal
    side *= width
    return mid + side, mid - side
```
