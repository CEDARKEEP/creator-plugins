# Advanced Production Techniques

Multiband compression, transient shaping, parallel compression, mid/side processing, stereo widening, and sidechain techniques. All numpy/scipy.

## Multiband Compression

```python
def multiband_compress(sig, crossovers=[200, 2000, 8000], sr=SR):
    """Split signal into 4 bands, compress each independently."""
    bands = []
    remaining = sig.copy()
    for freq in crossovers:
        band = sosfilt(butter(4, freq, btype='low', fs=sr, output='sos'), remaining)
        remaining = sosfilt(butter(4, freq, btype='high', fs=sr, output='sos'), remaining)
        bands.append(band)
    bands.append(remaining)
    settings = [
        (-20, 3.0, 10, 200),   # Sub/bass: gentle, slow
        (-18, 2.0, 8, 150),    # Low-mids: gentle
        (-15, 2.5, 5, 100),    # High-mids: tighter
        (-12, 3.0, 3, 80),     # Highs: fast, aggressive
    ]
    return sum(compressor(b, threshold_db=s[0], ratio=s[1], attack_ms=s[2], release_ms=s[3], sr=sr)
               for b, s in zip(bands, settings))
```

## Transient Shaper

```python
def transient_shaper(sig, attack_gain_db=6, sustain_gain_db=0, sr=SR):
    """Shape attack transients and sustain independently.
    attack_gain_db > 0 = punchier. sustain_gain_db > 0 = fatter."""
    n = len(sig)
    abs_sig = np.abs(sig)
    fast_env = np.zeros(n); slow_env = np.zeros(n)
    fa, fr = np.exp(-1/(0.001*sr)), np.exp(-1/(0.01*sr))
    sa, srel = np.exp(-1/(0.02*sr)), np.exp(-1/(0.2*sr))
    fe = se = 0.0
    for i in range(n):
        fe = fa*fe + (1-fa)*abs_sig[i] if abs_sig[i] > fe else fr*fe + (1-fr)*abs_sig[i]
        se = sa*se + (1-sa)*abs_sig[i] if abs_sig[i] > se else srel*se + (1-srel)*abs_sig[i]
        fast_env[i] = fe; slow_env[i] = se
    transient = np.clip(fast_env - slow_env, 0, None)
    ag = 10 ** (attack_gain_db / 20); sg = 10 ** (sustain_gain_db / 20)
    gain = 1 + (ag - 1) * transient / (np.max(transient) + 1e-10) + \
           (sg - 1) * slow_env / (np.max(slow_env) + 1e-10)
    return sig * gain
```

## Parallel Compression (NY Compression)

```python
def parallel_compress(sig, threshold_db=-25, ratio=8, blend=0.4, sr=SR):
    """Blend heavily compressed with dry. Adds weight without losing transients."""
    compressed = compressor(sig, threshold_db=threshold_db, ratio=ratio,
                           attack_ms=1, release_ms=100, sr=sr)
    return sig + compressed * blend
```

## Mid/Side Processing

```python
def mid_side_process(left, right, sr=SR):
    """Mid/side mastering: mono bass, presence in center, air on sides."""
    mid = (left + right) * 0.5
    side = (left - right) * 0.5
    side = sosfilt(butter(2, 200, btype='high', fs=sr, output='sos'), side)  # Mono bass
    mid = peak_eq(mid, 3000, 1.5, Q=1.0, sr=sr)  # Center presence
    side = high_shelf(side, 8000, 2.0, sr=sr)  # Side air
    return mid + side, mid - side
```

## Frequency-Dependent Stereo Widener

```python
def freq_stereo_widener(left, right, width=1.5, sr=SR):
    """Keep bass mono, widen highs. Mono-compatible."""
    sos_lp = butter(2, 300, btype='low', fs=sr, output='sos')
    sos_hp = butter(2, 300, btype='high', fs=sr, output='sos')
    low = (sosfilt(sos_lp, left) + sosfilt(sos_lp, right)) * 0.5
    hi_l, hi_r = sosfilt(sos_hp, left), sosfilt(sos_hp, right)
    mid_h = (hi_l + hi_r) * 0.5
    side_h = (hi_l - hi_r) * 0.5 * width
    return low + mid_h + side_h, low + mid_h - side_h
```

## Sidechain Envelope (Without Trigger Signal)

```python
def sidechain_envelope(n_samples, bpm, pattern='four_on_floor', depth=0.8,
                       release_ms=200, sr=SR):
    """Generate a sidechain ducking envelope without a kick track.
    Multiply your pad/chord signal by this."""
    beat_samples = int(60 * sr / bpm)
    env = np.ones(n_samples)
    release = int(release_ms * sr / 1000)
    step = beat_samples if pattern == 'four_on_floor' else beat_samples * 2
    for pos in range(0, n_samples, step):
        for i in range(min(release, n_samples - pos)):
            t = i / release
            gain = (1 - depth) + depth * (1 - np.exp(-t * 5))
            env[pos + i] = min(env[pos + i], gain)
    return env
```

## Haas Effect (Stereo Widening)

```python
def haas_effect(mono, delay_ms=15, sr=SR):
    """Widen mono source via short delay. 10-30ms. NOT mono-compatible."""
    delay = int(delay_ms * sr / 1000)
    delayed = np.concatenate([np.zeros(delay), mono])[:len(mono)]
    return mono, delayed  # left, right
```
