# Spectral Processing & Time-Stretching

STFT-based spectral operations, phase vocoder, and Paulstretch. All numpy/scipy.

## STFT / ISTFT

```python
def stft(sig, fft_size=2048, hop_size=512):
    """Short-Time Fourier Transform. Returns complex spectrogram."""
    window = np.hanning(fft_size)
    n_frames = 1 + (len(sig) - fft_size) // hop_size
    frames = np.zeros((n_frames, fft_size // 2 + 1), dtype=complex)
    for i in range(n_frames):
        start = i * hop_size
        frames[i] = np.fft.rfft(sig[start:start + fft_size] * window)
    return frames

def istft(frames, fft_size=2048, hop_size=512):
    """Inverse STFT with overlap-add."""
    window = np.hanning(fft_size)
    n_frames = len(frames)
    out_len = fft_size + (n_frames - 1) * hop_size
    output = np.zeros(out_len)
    window_sum = np.zeros(out_len)
    for i in range(n_frames):
        start = i * hop_size
        output[start:start + fft_size] += np.fft.irfft(frames[i], n=fft_size) * window
        window_sum[start:start + fft_size] += window ** 2
    return output / np.maximum(window_sum, 1e-10)
```

## Spectral Freeze

Capture one frame, resynthesize indefinitely with random phases:

```python
def spectral_freeze(sig, freeze_time, duration, fft_size=2048, hop_size=512, sr=SR):
    frames = stft(sig, fft_size, hop_size)
    idx = min(int(freeze_time * sr / hop_size), len(frames) - 1)
    frozen_mag = np.abs(frames[idx])
    n_out = int(duration * sr / hop_size)
    out_frames = np.zeros((n_out, fft_size // 2 + 1), dtype=complex)
    for i in range(n_out):
        out_frames[i] = frozen_mag * np.exp(1j * np.random.uniform(0, 2*np.pi, len(frozen_mag)))
    return istft(out_frames, fft_size, hop_size)
```

## Spectral Morph

Interpolate magnitude spectra between two sounds:

```python
def spectral_morph(sig_a, sig_b, morph_curve=None, fft_size=2048, hop_size=512):
    frames_a, frames_b = stft(sig_a, fft_size, hop_size), stft(sig_b, fft_size, hop_size)
    n = min(len(frames_a), len(frames_b))
    if morph_curve is None:
        morph_curve = np.linspace(0, 1, n)
    out_frames = np.zeros((n, fft_size // 2 + 1), dtype=complex)
    for i in range(n):
        m = morph_curve[i]
        mag = np.abs(frames_a[i]) * (1 - m) + np.abs(frames_b[i]) * m
        out_frames[i] = mag * np.exp(1j * np.angle(frames_a[i]))
    return istft(out_frames, fft_size, hop_size)
```

## Spectral Gate / Blur

```python
def spectral_gate(sig, threshold_db=-40, fft_size=2048, hop_size=512):
    """Zero bins below threshold. Sparse, crystalline textures."""
    frames = stft(sig, fft_size, hop_size)
    threshold = 10 ** (threshold_db / 20)
    for i in range(len(frames)):
        frames[i] *= (np.abs(frames[i]) > threshold)
    return istft(frames, fft_size, hop_size)

def spectral_blur(sig, blur_width=5, fft_size=2048, hop_size=512):
    """Average adjacent frequency bins. Washy, dreamlike quality."""
    frames = stft(sig, fft_size, hop_size)
    kernel = np.ones(blur_width) / blur_width
    for i in range(len(frames)):
        mag = np.convolve(np.abs(frames[i]), kernel, mode='same')
        frames[i] = mag * np.exp(1j * np.angle(frames[i]))
    return istft(frames, fft_size, hop_size)
```

## Phase Vocoder

Time-stretch without pitch change, pitch-shift without time change.

```python
def phase_vocoder_stretch(sig, stretch_factor, fft_size=2048, hop_size=512, sr=SR):
    """Time-stretch. stretch_factor > 1 = slower, < 1 = faster. Does NOT change pitch."""
    window = np.hanning(fft_size)
    n_frames = 1 + (len(sig) - fft_size) // hop_size
    hop_out = int(hop_size * stretch_factor)
    omega = 2 * np.pi * np.arange(fft_size // 2 + 1) * hop_size / fft_size
    out_len = int(len(sig) * stretch_factor) + fft_size
    output = np.zeros(out_len)
    win_sum = np.zeros(out_len)
    prev_phase = np.zeros(fft_size // 2 + 1)
    cum_phase = np.zeros(fft_size // 2 + 1)
    for i in range(n_frames):
        start = i * hop_size
        frame = sig[start:start + fft_size]
        if len(frame) < fft_size:
            frame = np.pad(frame, (0, fft_size - len(frame)))
        spec = np.fft.rfft(frame * window)
        mag, phase = np.abs(spec), np.angle(spec)
        phase_diff = phase - prev_phase - omega
        phase_diff -= 2 * np.pi * np.round(phase_diff / (2 * np.pi))
        cum_phase += (omega + phase_diff) * stretch_factor
        syn_frame = np.fft.irfft(mag * np.exp(1j * cum_phase), n=fft_size) * window
        out_start = i * hop_out
        if out_start + fft_size <= out_len:
            output[out_start:out_start + fft_size] += syn_frame
            win_sum[out_start:out_start + fft_size] += window ** 2
        prev_phase = phase
    return (output / np.maximum(win_sum, 1e-10))[:int(len(sig) * stretch_factor)]

def pitch_shift(sig, semitones, fft_size=2048, hop_size=512, sr=SR):
    """Pitch-shift without changing duration. Time-stretch then resample."""
    ratio = 2 ** (semitones / 12.0)
    stretched = phase_vocoder_stretch(sig, 1.0 / ratio, fft_size, hop_size, sr)
    indices = np.linspace(0, len(stretched) - 1, len(sig))
    idx0 = np.clip(indices.astype(int), 0, len(stretched) - 2)
    frac = indices - idx0
    return stretched[idx0] * (1 - frac) + stretched[idx0 + 1] * frac
```

## Paulstretch (Extreme Time-Stretch)

FFT each frame, keep magnitudes, randomize phases. Creates smooth ambient textures from any sound.

```python
def paulstretch(sig, stretch_factor=8.0, window_size_s=0.25, sr=SR):
    """Extreme time-stretching. stretch_factor 8-100x for ambient textures."""
    window_samples = int(window_size_s * sr)
    if window_samples % 2 != 0:
        window_samples += 1
    half_window = window_samples // 2
    window = np.hanning(window_samples)
    hop_in = half_window
    hop_out = int(hop_in * stretch_factor)
    n_frames = max(1, (len(sig) - window_samples) // hop_in)
    out_len = n_frames * hop_out + window_samples
    output = np.zeros(out_len)
    win_sum = np.zeros(out_len)
    for i in range(n_frames):
        start = i * hop_in
        frame = sig[start:start + window_samples]
        if len(frame) < window_samples:
            frame = np.pad(frame, (0, window_samples - len(frame)))
        spec = np.fft.rfft(frame * window)
        mag = np.abs(spec)
        random_phase = np.exp(1j * np.random.uniform(0, 2*np.pi, len(mag)))
        syn_frame = np.fft.irfft(mag * random_phase, n=window_samples) * window
        out_start = i * hop_out
        output[out_start:out_start + window_samples] += syn_frame
        win_sum[out_start:out_start + window_samples] += window ** 2
    return output / np.maximum(win_sum, 1e-10)
```
