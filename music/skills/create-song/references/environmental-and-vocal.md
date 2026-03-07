# Environmental Sounds, Vocal Processing & Body Percussion

Rain, wind, thunder, ocean waves, vocal effects, processing chains, and body percussion. All numpy/scipy.

## Environmental Sounds

### Rain

```python
def env_rain(duration, intensity=0.7, sr=SR):
    """Rain: continuous wash + individual droplet impulses.
    intensity 0.0-1.0 controls density and volume."""
    n = int(sr * duration)
    # Wash (bandpassed noise)
    wash = np.random.randn(n)
    sos = butter(2, [1000, 12000], btype='band', fs=sr, output='sos')
    wash = sosfilt(sos, wash) * intensity * 0.3
    # Droplets (short sine bursts with Hanning windows)
    drops = np.zeros(n)
    n_drops = int(duration * 30 * intensity)
    for _ in range(n_drops):
        pos = np.random.randint(0, n)
        freq = np.random.uniform(2000, 8000)
        length = int(np.random.uniform(0.001, 0.008) * sr)
        if pos + length < n:
            t = np.arange(length) / sr
            drop = np.sin(2 * np.pi * freq * t) * np.hanning(length)
            drops[pos:pos + length] += drop * np.random.uniform(0.01, 0.05)
    return wash + drops
```

### Wind

```python
def env_wind(duration, intensity=0.5, sr=SR):
    """Wind: slowly modulated filtered noise with organic LFO."""
    n = int(sr * duration)
    noise = np.random.randn(n)
    t = np.arange(n) / sr
    # Dual LFOs at irrational ratio for organic feel
    lfo1 = np.sin(2 * np.pi * 0.15 * t)
    lfo2 = np.sin(2 * np.pi * 0.15 * 2.37 * t)  # Irrational ratio
    cutoff = 300 + 2700 * (0.5 + 0.3 * lfo1 + 0.2 * lfo2) * intensity
    amp_mod = 0.3 + 0.7 * (0.5 + 0.3 * lfo1 + 0.2 * lfo2) * intensity
    block = 256
    zi = np.zeros((2, 2))
    out = np.zeros(n)
    for s in range(0, n, block):
        e = min(s + block, n)
        fc = np.clip(cutoff[s], 20, sr/2 - 100)
        sos = butter(2, fc, btype='low', fs=sr, output='sos')
        out[s:e], zi = sosfilt(sos, noise[s:e], zi=zi)
    return out * amp_mod
```

### Thunder

```python
def env_thunder(duration=4.0, distance_km=2.0, sr=SR):
    """Thunder: sharp crack + delayed low rumble + echoes.
    distance_km affects delay and high-freq rolloff."""
    n = int(sr * duration)
    t = np.arange(n) / sr
    delay_s = distance_km / 0.343  # Speed of sound ~343 m/s
    # Crack (sharp, broadband)
    crack_pos = int(min(delay_s * 0.1, 0.5) * sr)  # Slight pre-delay
    crack = np.zeros(n)
    crack_len = int(0.05 * sr)
    if crack_pos + crack_len < n:
        crack[crack_pos:crack_pos + crack_len] = np.random.randn(crack_len) * np.exp(-np.arange(crack_len) / (0.02 * sr))
    sos_c = butter(2, min(8000 / (1 + distance_km), sr/2 - 100), btype='low', fs=sr, output='sos')
    crack = sosfilt(sos_c, crack)
    # Rumble (delayed, low)
    rumble_pos = int(delay_s * 0.3 * sr)
    rumble = np.zeros(n)
    rumble_len = n - rumble_pos
    if rumble_len > 0:
        rumble[rumble_pos:] = np.random.randn(rumble_len) * np.exp(-np.arange(rumble_len) / (1.5 * sr))
    sos_r = butter(2, [20, 150], btype='band', fs=sr, output='sos')
    rumble = sosfilt(sos_r, rumble)
    # Echoes (3-5 delayed copies of crack with decay)
    echoes = np.zeros(n)
    for i in range(4):
        echo_delay = int((0.3 + i * 0.4) * sr)
        echo_amp = 0.4 * (0.6 ** i)
        if echo_delay < n:
            echo_len = min(crack_len, n - echo_delay)
            echoes[echo_delay:echo_delay + echo_len] += crack[crack_pos:crack_pos + echo_len] * echo_amp
    sos_e = butter(2, [30, 300], btype='band', fs=sr, output='sos')
    echoes = sosfilt(sos_e, echoes)
    return (crack * 0.4 + rumble * 0.4 + echoes * 0.2) / 1.0
```

### Ocean Waves

```python
def env_ocean(duration, wave_period=8.0, sr=SR):
    """Ocean: periodic surge + foam hiss at wave peaks."""
    n = int(sr * duration)
    t = np.arange(n) / sr
    noise = np.random.randn(n)
    # Surge envelope (periodic, asymmetric)
    surge = (0.5 + 0.5 * np.sin(2 * np.pi * t / wave_period - np.pi/2)) ** 2
    surge = 0.3 + 0.7 * surge
    # Filter opens with surge (300-4000Hz)
    cutoff = 300 + 3700 * surge
    block = 256
    zi = np.zeros((2, 2))
    wave = np.zeros(n)
    for s in range(0, n, block):
        e = min(s + block, n)
        fc = np.clip(cutoff[s], 20, sr/2 - 100)
        sos = butter(2, fc, btype='low', fs=sr, output='sos')
        wave[s:e], zi = sosfilt(sos, noise[s:e], zi=zi)
    wave *= surge
    # Foam hiss at wave peaks
    foam = np.random.randn(n)
    sos_f = butter(2, [2000, 12000], btype='band', fs=sr, output='sos')
    foam = sosfilt(sos_f, foam) * np.clip(surge - 0.6, 0, 1) * 0.3
    return wave * 0.7 + foam * 0.3
```

## Vocal Synthesis & Processing

### Vocoder

See [synthesis-techniques.md](synthesis-techniques.md) for the `vocoder()` function.

### Auto-Tune (Pitch Correction)

```python
def auto_tune(sig, scale_freqs, correction_strength=0.8, block_size=2048, sr=SR):
    """Simple pitch correction. scale_freqs = list of allowed frequencies.
    correction_strength: 0.3=subtle, 1.0=hard T-Pain effect."""
    from scipy.signal import fftconvolve
    hop = block_size // 4
    n_blocks = (len(sig) - block_size) // hop
    output = np.zeros(len(sig))
    window = np.hanning(block_size)
    win_sum = np.zeros(len(sig))
    for i in range(n_blocks):
        start = i * hop
        frame = sig[start:start + block_size] * window
        # Autocorrelation pitch detection
        corr = fftconvolve(frame, frame[::-1], mode='full')
        corr = corr[len(corr)//2:]
        min_lag = int(sr / 1000)  # Max 1000Hz
        max_lag = int(sr / 50)    # Min 50Hz
        if max_lag > len(corr):
            max_lag = len(corr) - 1
        search = corr[min_lag:max_lag]
        if len(search) == 0:
            output[start:start + block_size] += frame
            win_sum[start:start + block_size] += window
            continue
        lag = min_lag + np.argmax(search)
        detected_freq = sr / lag
        # Find nearest scale frequency
        nearest = min(scale_freqs, key=lambda f: abs(f - detected_freq))
        # Pitch shift amount
        shift_ratio = nearest / detected_freq
        shift_amount = 1.0 + (shift_ratio - 1.0) * correction_strength
        # Resample frame
        indices = np.linspace(0, len(frame) - 1, int(len(frame) / shift_amount))
        indices = np.clip(indices, 0, len(frame) - 1)
        idx0 = indices.astype(int)
        frac = indices - idx0
        shifted = frame[idx0] * (1 - frac) + frame[np.minimum(idx0 + 1, len(frame) - 1)] * frac
        # Overlap-add
        actual = min(len(shifted), block_size)
        output[start:start + actual] += shifted[:actual] * window[:actual]
        win_sum[start:start + actual] += window[:actual]
    win_sum[win_sum < 1e-10] = 1
    return output / win_sum
```

### Harmonizer

```python
def harmonizer(sig, intervals_semitones=[4, 7], sr=SR):
    """Create harmony voices at specified intervals. Uses phase vocoder pitch shift."""
    output = sig.copy()
    for semitones in intervals_semitones:
        # Use pitch_shift from spectral-processing.md
        shifted = pitch_shift(sig, semitones, sr=sr)
        output += shifted * 0.6  # Harmony voices slightly quieter
    return output / (np.max(np.abs(output)) + 1e-10)
```

### Vocal Doubling

```python
def vocal_double(sig, detune_cents=10, delay_ms=20, sr=SR):
    """Thicken a vocal/lead by detuning + delaying a copy."""
    # Detune
    ratio = 2 ** (detune_cents / 1200)
    n_out = int(len(sig) / ratio)
    indices = np.linspace(0, len(sig) - 1, n_out)
    idx0 = indices.astype(int)
    frac = indices - idx0
    detuned = sig[idx0] * (1 - frac) + sig[np.minimum(idx0 + 1, len(sig) - 1)] * frac
    # Pad back to original length
    if len(detuned) < len(sig):
        detuned = np.pad(detuned, (0, len(sig) - len(detuned)))
    else:
        detuned = detuned[:len(sig)]
    # Delay
    delay_samples = int(delay_ms * sr / 1000)
    delayed = np.concatenate([np.zeros(delay_samples), detuned])[:len(sig)]
    return sig + delayed * 0.5
```

### Whisper Synthesis

```python
def synth_whisper(duration, vowel='a', syllable_rate=4, sr=SR):
    """Pure noise through formant filters. No pitch (no voicing)."""
    FORMANTS = {
        'a': [(800,80), (1150,70), (2800,100)],
        'e': [(400,70), (1600,80), (2700,100)],
        'i': [(270,60), (2300,90), (3000,100)],
        'o': [(500,70), (700,80), (2800,100)],
        'u': [(300,60), (640,70), (2700,100)],
    }
    n = int(sr * duration)
    noise = np.random.randn(n)
    t = np.arange(n) / sr
    # Syllable rhythm via amplitude modulation
    syllable_env = 0.5 + 0.5 * np.sin(2 * np.pi * syllable_rate * t)
    syllable_env = syllable_env ** 3  # Sharpen peaks
    # Formant filtering
    output = np.zeros(n)
    for fc, bw in FORMANTS.get(vowel, FORMANTS['a']):
        sos = butter(2, [max(20, fc-bw/2), min(sr/2-100, fc+bw/2)], btype='band', fs=sr, output='sos')
        output += sosfilt(sos, noise)
    return output * syllable_env
```

## Processing Effects

### Telephone / Radio

```python
def fx_telephone(sig, sr=SR):
    """ITU-T telephone bandwidth: 300-3400Hz + light saturation."""
    sos = butter(4, [300, 3400], btype='band', fs=sr, output='sos')
    filtered = sosfilt(sos, sig)
    return np.tanh(filtered * 2.0) * 0.7
```

### Full Lo-Fi Chain

```python
def fx_lofi_chain(sig, sr=SR):
    """Complete lo-fi processing: tape wobble + bitcrush + LP + saturation."""
    # Tape wobble (8 cents at 0.5Hz)
    t = np.arange(len(sig)) / sr
    mod = (8 / 1200.0) * np.sin(2 * np.pi * 0.5 * t)
    indices = np.arange(len(sig)) + mod * sr
    indices = np.clip(indices, 0, len(sig) - 2)
    idx0 = indices.astype(int); frac = indices - idx0
    sig = sig[idx0] * (1 - frac) + sig[np.minimum(idx0 + 1, len(sig) - 1)] * frac
    # Gentle bitcrush (12-bit, sample reduce 2)
    levels = 2 ** 12
    sig = np.round(sig * levels / 2) / (levels / 2)
    for i in range(len(sig)):
        if i % 2 != 0:
            sig[i] = sig[i - 1]
    # Lowpass at 4kHz
    sos = butter(2, 4000, btype='low', fs=sr, output='sos')
    sig = sosfilt(sos, sig)
    # Tape saturation
    sig = np.tanh(sig * 1.5) / np.tanh(1.5)
    return sig
```

## Body Percussion Kit

```python
def synth_body_stomp(duration=0.3, sr=SR):
    """Floor stomp: low sine + noise."""
    t = np.arange(int(sr * duration)) / sr
    body = np.sin(2 * np.pi * 80 * t) * np.exp(-t / 0.08)
    noise = np.random.randn(len(t)) * np.exp(-t / 0.02) * 0.3
    sos = butter(2, 500, btype='low', fs=sr, output='sos')
    sig = sosfilt(sos, body + noise)
    sig[:int(0.003*sr)] *= np.linspace(0, 1, int(0.003*sr)) ** 2
    return sig / (np.max(np.abs(sig)) + 1e-10)

def synth_finger_snap(duration=0.08, sr=SR):
    """Finger snap: bandpassed noise burst."""
    t = np.arange(int(sr * duration)) / sr
    noise = np.random.randn(len(t)) * np.exp(-t / 0.008)
    sos = butter(2, [1500, 8000], btype='band', fs=sr, output='sos')
    sig = sosfilt(sos, noise)
    sig[:int(0.002*sr)] *= np.linspace(0, 1, int(0.002*sr))
    return sig / (np.max(np.abs(sig)) + 1e-10)

def synth_chest_hit(duration=0.2, sr=SR):
    """Chest hit: mid-range thump."""
    t = np.arange(int(sr * duration)) / sr
    body = np.sin(2 * np.pi * 120 * t) * np.exp(-t / 0.05)
    noise = np.random.randn(len(t)) * np.exp(-t / 0.015) * 0.4
    sos = butter(2, [60, 1000], btype='band', fs=sr, output='sos')
    sig = sosfilt(sos, body + noise)
    sig[:int(0.003*sr)] *= np.linspace(0, 1, int(0.003*sr)) ** 2
    return sig / (np.max(np.abs(sig)) + 1e-10)

def synth_thigh_slap(duration=0.1, sr=SR):
    """Thigh slap: short, snappy."""
    t = np.arange(int(sr * duration)) / sr
    body = np.sin(2 * np.pi * 200 * t) * np.exp(-t / 0.025)
    noise = np.random.randn(len(t)) * np.exp(-t / 0.01) * 0.5
    sos = butter(2, [200, 4000], btype='band', fs=sr, output='sos')
    sig = sosfilt(sos, body + noise)
    sig[:int(0.002*sr)] *= np.linspace(0, 1, int(0.002*sr))
    return sig / (np.max(np.abs(sig)) + 1e-10)
```
