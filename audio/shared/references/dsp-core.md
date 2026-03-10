# DSP Core: Helpers, Oscillators, Filters, Envelopes

Foundational DSP primitives for all music generation. Import these in every script.

## Core Helpers

```python
import numpy as np
from scipy.signal import butter, sosfilt
from scipy.io import wavfile

SR = 44100
NYQUIST = SR / 2  # 22050 Hz

def place(buf, sound, position_seconds):
    """Place a sound into a buffer at a given time. Safe against overruns."""
    start = int(position_seconds * SR)
    if start < 0 or start >= len(buf):
        return
    end = min(start + len(sound), len(buf))
    buf[start:end] += sound[:end - start]

def note_freq(name):
    """Convert note name to frequency. e.g. 'C4' -> 261.63, 'A#3' -> 233.08"""
    notes = {'C':0,'C#':1,'Db':1,'D':2,'D#':3,'Eb':3,'E':4,'F':5,
             'F#':6,'Gb':6,'G':7,'G#':8,'Ab':8,'A':9,'A#':10,'Bb':10,'B':11}
    octave = int(name[-1])
    note = name[:-1]
    semitones = notes[note] + (octave - 4) * 12
    return 261.63 * (2 ** (semitones / 12))

def midi_to_freq(midi_note):
    """MIDI note number to Hz. A4=440Hz is MIDI 69."""
    return 440.0 * (2 ** ((midi_note - 69) / 12))

def smoothstep(t):
    """Smooth interpolation for fades and transitions."""
    t = np.clip(t, 0, 1)
    return t * t * (3 - 2 * t)

rng = np.random.RandomState(42)  # seeded for reproducibility
```

## Stereo Helpers

```python
def pan_mono(mono, position=0.5):
    """Equal-power pan. position: 0=left, 0.5=center, 1=right. Returns (L, R)."""
    angle = position * np.pi * 0.5
    return mono * np.cos(angle), mono * np.sin(angle)

def stereo_place(buf_L, buf_R, sound_L, sound_R, position_seconds):
    place(buf_L, sound_L, position_seconds)
    place(buf_R, sound_R, position_seconds)

```

See [effects.md](effects.md) for `stereo_width()` (mid-side processing with sub-150Hz centering).

Panning positions (keep sub-150Hz centered):

| Element | Pan | Notes |
|---------|-----|-------|
| Kick, Bass, Sub | 0.5 (center) | Always center for low end |
| Snare, Clap | 0.5 | Center anchor |
| Hi-hat | 0.35-0.4 | Slightly left |
| Pad L / Pad R | 0.25 / 0.75 | Wide stereo field |
| Lead melody | 0.45-0.55 | Near center |
| Arps | 0.3-0.7 | Spread via chorus or ping-pong |
| FX, Atmosphere | 0.2-0.8 | Wide for immersion |

## Oscillators (PolyBLEP Anti-Aliased)

Always use these -- naive sawtooth/square alias badly above ~2kHz at 44100 Hz.

```python
def polyblep(phase, dt):
    """Vectorized PolyBLEP correction for anti-aliased waveforms."""
    out = np.zeros_like(phase)
    mask1 = phase < dt
    x1 = phase[mask1] / dt
    out[mask1] = x1 + x1 - x1 * x1 - 1.0
    mask2 = phase > (1.0 - dt)
    x2 = (phase[mask2] - 1.0) / dt
    out[mask2] = x2 * x2 + x2 + x2 + 1.0
    return out

def sine(freq, duration, sr=SR):
    t = np.linspace(0, duration, int(sr * duration), endpoint=False)
    return np.sin(2 * np.pi * freq * t)

def saw(freq, duration, sr=SR):
    """Anti-aliased sawtooth using PolyBLEP."""
    n = int(sr * duration)
    dt = freq / sr
    phase = np.cumsum(np.full(n, dt)) % 1.0
    return 2.0 * phase - 1.0 - polyblep(phase, dt)

def square(freq, duration, duty=0.5, sr=SR):
    """Anti-aliased square/pulse using PolyBLEP."""
    n = int(sr * duration)
    dt = freq / sr
    phase = np.cumsum(np.full(n, dt)) % 1.0
    value = np.where(phase < duty, 1.0, -1.0)
    value += polyblep(phase, dt)
    value -= polyblep((phase + (1.0 - duty)) % 1.0, dt)
    return value

def triangle(freq, duration, sr=SR):
    t = np.linspace(0, duration, int(sr * duration), endpoint=False)
    return 2.0 * np.abs(2.0 * (t * freq - np.floor(t * freq + 0.5))) - 1.0

def noise(duration, sr=SR):
    return np.random.uniform(-1, 1, int(sr * duration))

def pink_noise(duration, sr=SR):
    """Pink noise (1/f) via FFT spectral shaping."""
    n = int(sr * duration)
    white = np.random.randn(n)
    spectrum = np.fft.rfft(white)
    freqs = np.fft.rfftfreq(n, 1.0 / sr)
    freqs[0] = 1
    spectrum *= 1.0 / np.sqrt(freqs)
    return np.fft.irfft(spectrum, n=n)

def brown_noise(duration, sr=SR):
    """Brown/red noise via cumulative sum. -6 dB/octave. Rumbly."""
    n = int(sr * duration)
    sig = np.cumsum(np.random.randn(n))
    return sig / (np.max(np.abs(sig)) + 1e-10)
```

### Additive Synthesis

Build complex timbres by summing sine partials:

```python
def additive_synth(freq, duration, amplitudes, sr=SR):
    """amplitudes: list where index 0 = fundamental."""
    t = np.linspace(0, duration, int(sr * duration), endpoint=False)
    output = np.zeros_like(t)
    for i, amp in enumerate(amplitudes):
        h_freq = freq * (i + 1)
        if h_freq >= NYQUIST:
            break
        output += amp * np.sin(2 * np.pi * h_freq * t)
    return output
```

Amplitude falloff patterns:

| Waveform | Harmonic amplitudes | Harmonics present | Character |
|----------|-------------------|-------------------|-----------|
| Sawtooth | `1/n` | All (1,2,3...) | Bright, buzzy |
| Square | `1/n` | Odd only (1,3,5...) | Hollow, woody |
| Triangle | `1/n^2` | Odd only | Mellow, flute-like |
| Brass-like | ~equal first 20-30 | All | Bright, brassy |
| Strings | `1/n` with randomness | All | Warm, rich |

### FM Synthesis

```python
def fm_synth(carrier_freq, mod_ratio, mod_index, duration, sr=SR):
    """mod_ratio: fm/fc. mod_index: I, controls spectral complexity (~I+1 sideband pairs)."""
    t = np.linspace(0, duration, int(sr * duration), endpoint=False)
    mod_freq = carrier_freq * mod_ratio
    modulator = mod_index * np.sin(2 * np.pi * mod_freq * t)
    return np.sin(2 * np.pi * carrier_freq * t + modulator)

def fm_synth_2op(carrier_freq, mod_ratio, mod_index_env, duration, sr=SR):
    """FM with time-varying modulation index (envelope on brightness)."""
    t = np.linspace(0, duration, int(sr * duration), endpoint=False)
    mod_freq = carrier_freq * mod_ratio
    modulator = mod_index_env * np.sin(2 * np.pi * mod_freq * t)
    return np.sin(2 * np.pi * carrier_freq * t + modulator)
```

FM ratios for specific timbres:

| Timbre | C:M Ratio | Mod Index | Notes |
|--------|-----------|-----------|-------|
| Bell/chime | 1:3.5 | 5-10, decaying | Inharmonic = bell-like |
| Metallic bell | 1:1.414 | 8-15 | Irrational ratio |
| Bass (warm) | 1:1 | 1-3 | Adds odd+even harmonics |
| Electric piano tine | 1:14 | 1.5-3, fast decay | DX7 Rhodes character |
| Brass | 1:1 | 3-5 | High index = blaring |
| Brass stab | 1:1 | 0->5 fast env | Index env = filter opening |
| Clarinet | 1:3 | 2-4 | Odd harmonic emphasis |
| Plucked string | 1:1 | 5->0 fast decay | Fast index decay = pluck |
| Marimba | 1:4 | 3-8, very fast decay | Short transient |
| Vibraphone | 1:3.5 | 1-3 | Like bell but lower index |

### Wavetable Synthesis

```python
class WavetableOscillator:
    def __init__(self, table, sr=SR):
        self.table = np.array(table, dtype=np.float64)
        self.table_size = len(table)
        self.sr = sr

    @classmethod
    def from_harmonics(cls, amplitudes, table_size=2048, sr=SR):
        t = np.linspace(0, 2 * np.pi, table_size, endpoint=False)
        table = np.zeros(table_size)
        for i, amp in enumerate(amplitudes):
            table += amp * np.sin((i + 1) * t)
        table /= np.max(np.abs(table)) if np.max(np.abs(table)) > 0 else 1
        return cls(table, sr)

    def generate(self, freq, duration):
        n_samples = int(self.sr * duration)
        phase_inc = freq * self.table_size / self.sr
        indices = np.arange(n_samples) * phase_inc % self.table_size
        idx0 = indices.astype(int)
        idx1 = (idx0 + 1) % self.table_size
        frac = indices - idx0
        return self.table[idx0] * (1 - frac) + self.table[idx1] * frac
```

## Filters (Always use sosfilt -- NEVER lfilter)

Critical: `lfilter` with `butter` in `ba` form is numerically unstable for orders >= 4. Always use `output='sos'` and `sosfilt`. Max stable order with SOS: ~12.

Rolloff rates: Order N = N*6 dB/octave (order 4 = 24 dB/oct, classic synth filter).

```python
def lowpass(sig, cutoff, sr=SR, order=4):
    nyq = sr / 2
    sos = butter(order, min(cutoff / nyq, 0.99), btype='low', output='sos')
    return sosfilt(sos, sig)

def highpass(sig, cutoff, sr=SR, order=2):
    nyq = sr / 2
    sos = butter(order, max(min(cutoff / nyq, 0.99), 0.001), btype='high', output='sos')
    return sosfilt(sos, sig)

def bandpass(sig, low, high, sr=SR, order=2):
    nyq = sr / 2
    sos = butter(order, [max(low/nyq, 0.001), min(high/nyq, 0.99)], btype='band', output='sos')
    return sosfilt(sos, sig)

# For peak_eq(), high_shelf(), low_shelf() — see effects.md (EQ & Tone Shaping section)
```

### State Variable Filter (for resonant sweeps, acid squelch)

Use when you need time-varying cutoff. For static filtering, use `lowpass()`/`highpass()` above.

```python
def svf_lowpass(sig, cutoff_array, Q=1.0, sr=SR):
    """Chamberlin SVF with per-sample cutoff. cutoff_array = array of cutoff freqs."""
    n = len(sig)
    out = np.zeros(n)
    low = band = high = 0.0
    q = 1.0 / Q
    for i in range(n):
        f = 2 * np.sin(np.pi * min(cutoff_array[i], sr * 0.24) / sr)
        low += f * band
        high = sig[i] - low - q * band
        band += f * high
        out[i] = low
    return out
```

### Formant Filters (Vocal Synthesis)

Vowel formant frequencies (Hz, adult male voice):

| Vowel | F1 | F2 | F3 | Bandwidth |
|-------|-----|------|------|-----------|
| /a/ (ah) | 800 | 1150 | 2800 | F1:80, F2:70, F3:100 |
| /e/ (eh) | 400 | 1600 | 2700 | Similar |
| /i/ (ee) | 270 | 2300 | 3000 | F1:60, F2:90, F3:100 |
| /o/ (oh) | 500 | 700 | 2800 | Similar |
| /u/ (oo) | 300 | 640 | 2700 | Similar |

Filter choice guide:

| Use Case | Filter |
|----------|--------|
| Static EQ, basic lowpass/highpass | `butter(output='sos')` + `sosfilt` |
| Filter sweeps, envelope-controlled cutoff | SVF (`svf_lowpass`) |
| Warm resonant bass / acid | SVF with Q=2-4 |
| Vocal/choir synthesis | Parallel bandpass at formant freqs |

## Envelopes

Click prevention: All envelopes enforce a minimum 2ms attack and apply cosine micro-fades at edges.

```python
def env_adsr(n_samples, attack=0.01, decay=0.05, sustain=0.7, release=0.1, sr=SR):
    attack = max(attack, 0.002)
    release = max(release, 0.005)
    a = max(int(attack * sr), 1)
    d = max(int(decay * sr), 1)
    r = max(int(release * sr), 1)
    s = max(0, n_samples - a - d - r)
    env = np.concatenate([
        np.linspace(0, 1, a),
        np.linspace(1, sustain, d),
        np.full(s, sustain),
        np.linspace(sustain, 0, r),
    ])[:n_samples]
    fade_len = min(int(0.002 * sr), n_samples // 4)
    if fade_len > 0:
        fade_in = 0.5 - 0.5 * np.cos(np.linspace(0, np.pi, fade_len))
        fade_out = 0.5 + 0.5 * np.cos(np.linspace(0, np.pi, fade_len))
        env[:fade_len] *= fade_in
        env[-fade_len:] *= fade_out
    return env

def env_decay(n_samples, speed=10, sr=SR):
    t = np.linspace(0, n_samples / sr, n_samples, endpoint=False)
    env = np.exp(-t * speed)
    fade_len = min(int(0.002 * sr), n_samples // 4)
    if fade_len > 0:
        env[:fade_len] *= 0.5 - 0.5 * np.cos(np.linspace(0, np.pi, fade_len))
    return env

def env_swell(n_samples, attack_frac=0.5, decay_frac=0.5):
    """Swell envelope for pads -- slow rise, slow fall."""
    a = int(n_samples * attack_frac)
    d = n_samples - a
    env = np.concatenate([
        np.linspace(0, 1, max(a, 1)) ** 1.5,
        np.linspace(1, 0, max(d, 1)) ** 2,
    ])[:n_samples]
    fade_len = min(int(0.005 * SR), n_samples // 4)
    if fade_len > 0:
        env[:fade_len] *= 0.5 - 0.5 * np.cos(np.linspace(0, np.pi, fade_len))
        env[-fade_len:] *= 0.5 + 0.5 * np.cos(np.linspace(0, np.pi, fade_len))
    return env
```

Exponential envelopes sound more natural because human hearing is logarithmic. Linear ramps sound like they accelerate. For natural-sounding instruments, use exponential decay: `np.exp(-t * speed)`.

### Typical Envelope Shapes by Instrument

| Instrument | Attack | Decay | Sustain | Release | Character |
|-----------|--------|-------|---------|---------|-----------|
| Kick drum | 0.5-2ms | 50-200ms | 0.0 | 50-100ms | Percussive |
| Snare | 0.5-1ms | 80-150ms | 0.0 | 80-150ms | Percussive |
| Hi-hat closed | 0.5ms | 30-80ms | 0.0 | 30-50ms | Very short |
| Pluck/guitar | 1-5ms | 200-500ms | 0.0-0.3 | 100-300ms | Fast attack |
| Piano | 1-5ms | 1-5s | 0.0-0.2 | 0.5-2s | Long decay |
| Pad | 200-1000ms | 0.5-2s | 0.6-0.9 | 0.5-3s | Slow everything |
| Brass stab | 10-30ms | 100-200ms | 0.7-0.9 | 50-150ms | Medium attack |
| String ensemble | 300-800ms | 200-500ms | 0.8-0.95 | 300-800ms | Slow attack |
| Organ | 1-5ms | 0-10ms | 1.0 | 5-20ms | Instant on/off |
| Bell/chime | 0.5-2ms | 2-8s | 0.0 | 2-5s | Very long decay |

### DC Blocking Filter

Use after summing multiple detuned oscillators (pads, supersaw):

```python
def dc_block(sig, alpha=0.995):
    out = np.zeros_like(sig)
    prev_in = 0.0; prev_out = 0.0
    for i in range(len(sig)):
        out[i] = sig[i] - prev_in + alpha * prev_out
        prev_in = sig[i]; prev_out = out[i]
    return out
```

## Key Constants and Conversions

```python
def db_to_linear(db): return 10 ** (db / 20.0)
def linear_to_db(amp): return 20 * np.log10(max(abs(amp), 1e-10))
def bpm_to_ms(bpm, division=1): return 60000 / bpm * division
def cents_to_ratio(cents): return 2 ** (cents / 1200.0)
```

Frequency ranges:

| Range | Hz | Content |
|-------|------|---------|
| Sub bass | 20-60 | Feel more than hear |
| Bass | 60-250 | Bass instruments fundamental |
| Low mids | 250-500 | Warmth, muddiness |
| Mids | 500-2k | Body of most instruments |
| Upper mids | 2-4k | Presence, ear sensitivity peak |
| Treble | 4-20k | Air, brilliance |
