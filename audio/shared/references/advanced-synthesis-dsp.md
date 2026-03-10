# Advanced Sound Synthesis Mathematics & DSP Algorithms

Deep reference for studio-quality audio generation with Python/numpy/scipy.

## Contents

- [Anti-Aliased Oscillators Beyond PolyBLEP](#1-anti-aliased-oscillators-beyond-polyblep)
- [Physical Modeling Synthesis (Extended)](#2-physical-modeling-synthesis-extended)
- [FM Synthesis Deep Dive](#3-fm-synthesis-deep-dive)
- [Subtractive Synthesis -- Advanced Filter Modeling](#4-subtractive-synthesis----advanced-filter-modeling)
- [Granular Synthesis for Production](#5-granular-synthesis-for-production)
- [Additive Synthesis and Resynthesis](#6-additive-synthesis-and-resynthesis)
- [Wavetable Synthesis (Advanced)](#7-wavetable-synthesis-advanced)
- [Noise Synthesis and Spectral Shaping](#8-noise-synthesis-and-spectral-shaping)
- [Convolution Reverb Implementation](#9-convolution-reverb-implementation)
- [Phase Vocoder and Time-Stretching](#10-phase-vocoder-and-time-stretching)
- [Nonlinear Dynamics in Synthesis](#11-nonlinear-dynamics-in-synthesis)
- [Supersaw / Unison Optimization](#12-supersaw--unison-optimization)
- [Quick Reference: Which Technique for Which Sound?](#quick-reference-which-technique-for-which-sound)

---

## 1. Anti-Aliased Oscillators Beyond PolyBLEP

### The Aliasing Problem

A naive sawtooth `2*(freq*t % 1) - 1` has infinite harmonics. At sample rate `sr`, harmonics above `sr/2` fold back as inharmonic artifacts. For a 1kHz saw at 44100Hz, harmonics 23+ alias. The higher the pitch, the worse it gets.

### PolyBLEP (Polynomial Band-Limited Step)

The existing codebase uses 2-sample PolyBLEP. Quality scales with polynomial order:

| Samples | Order | Quality | Use Case |
|---------|-------|---------|----------|
| 2 | Linear | Good for bass/mid | Real-time, CPU-limited |
| 4 | Quadratic | Good to ~8kHz fundamentals | General purpose |
| 6 | Cubic | Excellent to ~12kHz | High-quality offline |
| 8 | Quartic | Near-perfect | Mastering quality |

For offline numpy rendering, 4-sample PolyBLEP is the sweet spot:

```python
def polyblep4(phase, dt):
    """4-sample (quadratic) PolyBLEP. Better HF rejection than 2-sample."""
    out = np.zeros_like(phase)
    # Transition region: 2*dt wide centered on discontinuity
    dt2 = 2 * dt

    # Near phase=0 (rising edge)
    mask1 = phase < dt2
    t1 = phase[mask1] / dt2  # 0 to 1
    out[mask1] = -(t13 / 3 - t12 + t1 - 1/3)

    # Near phase=1 (wrapping)
    mask2 = phase > (1.0 - dt2)
    t2 = (phase[mask2] - 1.0) / dt2  # -1 to 0
    out[mask2] = -(t23 / 3 + t22 + t2 + 1/3)

    return out

def saw_polyblep4(freq, duration, sr=44100):
    n = int(sr * duration)
    dt = freq / sr
    phase = np.cumsum(np.full(n, dt)) % 1.0
    return 2.0 * phase - 1.0 - polyblep4(phase, dt)
```

### MinBLEP (Minimum-Phase Band-Limited Step)

MinBLEP pre-computes a minimum-phase kernel from a windowed sinc, then inserts it at each discontinuity. Superior to PolyBLEP at high frequencies because the correction is exact in the band-limited sense.

Theory: Start with a band-limited step (integrated sinc), window it, convert to minimum phase via cepstral technique, then differentiate to get the MinBLEP kernel.

```python
def generate_minblep_table(n_zeros=16, oversampling=64):
    """Generate MinBLEP lookup table.
    n_zeros: sinc zero-crossings (more = steeper rolloff, 8-32 typical).
    oversampling: sub-sample resolution (32-128 typical).
    """
    # 1. Generate windowed sinc (band-limited impulse)
    n = 2 * n_zeros * oversampling + 1
    t = np.linspace(-n_zeros, n_zeros, n)
    sinc = np.sinc(t)  # sin(pi*t)/(pi*t)
    window = np.blackman(n)
    blit = sinc * window

    # 2. Integrate to get band-limited step (BLEP)
    blep = np.cumsum(blit)
    blep /= blep[-1]  # normalize to 0..1

    # 3. Convert to minimum phase via real cepstrum
    # This concentrates energy at the start, reducing latency
    n_fft = 1 << int(np.ceil(np.log2(len(blep))) + 1)
    spectrum = np.fft.rfft(blep, n_fft)
    log_mag = np.log(np.maximum(np.abs(spectrum), 1e-20))
    cepstrum = np.fft.irfft(log_mag)
    # Fold cepstrum to make minimum phase
    n_cep = len(cepstrum)
    min_phase_cep = np.zeros(n_cep)
    min_phase_cep[0] = cepstrum[0]
    min_phase_cep[1:n_cep//2] = 2 * cepstrum[1:n_cep//2]
    # min_phase_cep[n_cep//2+1:] = 0  (already zero)
    min_phase_spec = np.exp(np.fft.rfft(min_phase_cep))
    min_blep = np.fft.irfft(min_phase_spec)[:len(blep)]

    # 4. Subtract trivial step to get residual (what we add to naive waveform)
    trivial_step = np.concatenate([np.zeros(len(min_blep)//2), np.ones(len(min_blep) - len(min_blep)//2)])
    minblep_residual = min_blep - trivial_step

    return minblep_residual, oversampling

def saw_minblep(freq, duration, minblep_table, oversampling, sr=44100):
    """MinBLEP sawtooth. Insert pre-computed correction at each phase reset."""
    n = int(sr * duration)
    dt = freq / sr
    phase = np.zeros(n)
    output = np.zeros(n + len(minblep_table))
    p = 0.0

    for i in range(n):
        p += dt
        if p >= 1.0:
            p -= 1.0
            # Fractional sample position of the discontinuity
            frac = p / dt  # 0..1, how far past the transition
            # Insert MinBLEP at sub-sample precision
            sample_offset = int(frac * oversampling)
            kernel_start = sample_offset
            kernel_len = len(minblep_table) // oversampling
            for k in range(min(kernel_len, n - i)):
                table_idx = kernel_start + k * oversampling
                if table_idx < len(minblep_table):
                    output[i + k] -= 2.0 * minblep_table[table_idx]
        phase[i] = p
        output[i] += 2.0 * p - 1.0

    return output[:n]
```

When to use MinBLEP vs PolyBLEP: MinBLEP is better when fundamentals exceed ~4kHz or for master-quality rendering. PolyBLEP is simpler and sufficient for most musical content. For a numpy offline renderer, MinBLEP with n_zeros=16 is effectively transparent.

### BLIT (Band-Limited Impulse Train)

Generate a train of band-limited impulses, then integrate to form sawtooth/square. The BLIT itself is a ratio of sine functions:

```
BLIT(t) = (M / P) * sin(pi * M * t / P) / sin(pi * t / P)
```

where P = period in samples, M = number of harmonics (odd, <= sr/(2*freq)).

```python
def blit_saw(freq, duration, sr=44100):
    """BLIT-based sawtooth via leaky integration of impulse train."""
    n = int(sr * duration)
    period = sr / freq
    M = 2 * int(sr / (2 * freq)) + 1  # odd number of harmonics

    t = np.arange(n)
    phase = (t % period) / period  # 0 to 1

    # BLIT formula: sin(M * pi * phase) / (M * sin(pi * phase))
    denom = np.sin(np.pi * phase)
    numer = np.sin(np.pi * M * phase)

    blit = np.where(np.abs(denom) < 1e-7, 1.0, numer / (M * denom))

    # Leaky integrator to convert impulse train to sawtooth
    # DC offset removal: subtract 1/period per sample
    blit -= 1.0 / period

    saw = np.zeros(n)
    leak = 0.999  # slight leak prevents DC drift
    for i in range(1, n):
        saw[i] = leak * saw[i-1] + blit[i]

    return saw / (np.max(np.abs(saw)) + 1e-10)
```

### DPW (Differentiated Parabolic Waveform)

Cheapest anti-aliasing method. Square the naive sawtooth (creating a parabola), then differentiate. Differentiation of the parabola reconstructs the sawtooth but with attenuated harmonics above Nyquist.

```python
def dpw_saw(freq, duration, sr=44100):
    """First-order DPW sawtooth. Simple and effective."""
    n = int(sr * duration)
    dt = freq / sr
    phase = np.cumsum(np.full(n, dt)) % 1.0

    # Naive saw, squared to get parabola
    naive = 2.0 * phase - 1.0
    parabola = naive * naive  # x^2

    # Differentiate (first difference) and scale
    dpw = np.diff(parabola, prepend=parabola[0])
    scale = sr / (4.0 * freq)  # normalization factor

    return dpw * scale

def dpw2_saw(freq, duration, sr=44100):
    """Second-order DPW (DPW2). Better HF rolloff than DPW1.
    Uses x^3/6 polynomial, differentiated twice."""
    n = int(sr * duration)
    dt = freq / sr
    phase = np.cumsum(np.full(n, dt)) % 1.0
    naive = 2.0 * phase - 1.0

    # Third-order polynomial
    poly = naive ** 3 / 6.0

    # Differentiate twice
    d1 = np.diff(poly, prepend=poly[0])
    d2 = np.diff(d1, prepend=d1[0])

    # Scale factor for DPW2
    c = sr * sr / (freq * freq * 8)
    return np.clip(d2 * c, -1, 1)
```

### Quality Comparison

| Method | Alias Rejection | CPU Cost | Latency | Best For |
|--------|----------------|----------|---------|----------|
| PolyBLEP-2 | -40 dB | Very low | 0 samples | Real-time |
| PolyBLEP-4 | -60 dB | Low | 0 samples | General offline |
| DPW1 | -40 dB | Very low | 0 samples | Quick renders |
| DPW2 | -70 dB | Low | 0 samples | Good quality, simple |
| BLIT+integrator | -90 dB | Medium | 0 samples | High quality |
| MinBLEP-16 | -96 dB | Medium | ~16 samples | Mastering quality |
| Wavetable mipmap | -90+ dB | Low (lookup) | 0 samples | Best for polyphony |

### Wavetable with Mip-Mapping

Pre-compute band-limited versions of the wavetable at octave intervals. At playback, select the table with harmonics just below Nyquist.

```python
class MipMappedWavetable:
    """Anti-aliased wavetable using mip-mapped octave bands."""

    def __init__(self, single_cycle, sr=44100):
        self.sr = sr
        self.table_size = len(single_cycle)
        self.tables = []  # tables[i] valid for fundamentals up to sr / (2^(n-i))

        # Build mip-map: progressively remove top harmonics
        spectrum = np.fft.rfft(single_cycle)
        n_harmonics = len(spectrum)

        # Determine how many octave levels we need
        # Level 0 = full bandwidth, Level N = only fundamental
        max_freq = sr / 2
        n_levels = int(np.log2(n_harmonics)) + 1

        for level in range(n_levels):
            # Keep only harmonics that fit below Nyquist at this octave
            max_harmonic = max(1, n_harmonics >> level)
            limited = spectrum.copy()
            limited[max_harmonic:] = 0
            self.tables.append(np.fft.irfft(limited, n=self.table_size))

        self.n_levels = n_levels
        self.max_harmonics = [max(1, n_harmonics >> l) for l in range(n_levels)]

    def generate(self, freq, duration):
        """Generate anti-aliased output by selecting appropriate mip level."""
        n = int(self.sr * duration)

        # Select mip level: which table has harmonics fitting below Nyquist?
        max_harmonic = int(self.sr / (2 * freq))
        level = 0
        for i in range(self.n_levels):
            if self.max_harmonics[i] <= max_harmonic:
                level = i
                break

        table = self.tables[level]

        # Phase accumulator with linear interpolation
        phase_inc = freq * self.table_size / self.sr
        indices = np.arange(n) * phase_inc % self.table_size
        idx0 = indices.astype(int)
        idx1 = (idx0 + 1) % self.table_size
        frac = indices - idx0
        return table[idx0] * (1 - frac) + table[idx1] * frac
```

---

## 2. Physical Modeling Synthesis (Extended)

### Karplus-Strong with Allpass Fractional Delay

Integer delay lines only produce exact pitches at `sr/N` Hz. For arbitrary frequencies, use a first-order allpass filter for fractional delay interpolation:

```python
def ks_allpass(freq, duration=2.0, damping=0.996, brightness=0.5, sr=44100):
    """Karplus-Strong with allpass fractional delay for exact tuning.
    brightness: 0=dark/nylon, 0.5=steel, 1.0=glass."""
    n = int(sr * duration)

    # Fractional delay decomposition
    delay_exact = sr / freq
    delay_int = int(delay_exact)
    delay_frac = delay_exact - delay_int

    # First-order allpass coefficient for fractional delay
    # Thiran allpass: H(z) = (a + z^-1) / (1 + a*z^-1)
    # For fractional delay d: a = (1 - d) / (1 + d)
    a = (1.0 - delay_frac) / (1.0 + delay_frac)

    # Delay line
    buf = np.zeros(delay_int + 2)
    # Excitation: filtered noise burst
    burst_len = min(delay_int, int(sr * 0.003))
    buf[:burst_len] = np.random.uniform(-1, 1, burst_len)

    output = np.zeros(n)
    ap_state = 0.0  # allpass state
    lp_state = 0.0  # loop filter state

    for i in range(n):
        idx = i % (delay_int + 1)

        # Read from delay line with allpass interpolation
        read_idx = (i - delay_int) % (delay_int + 1)
        read_idx_prev = (i - delay_int - 1) % (delay_int + 1)

        # Allpass interpolation for fractional delay
        x_n = buf[read_idx]
        allpass_out = a * x_n + ap_state
        ap_state = x_n - a * allpass_out

        # Loop filter: one-pole lowpass for damping + brightness control
        # brightness=0: heavy filtering (dark), brightness=1: minimal filtering
        coeff = 0.1 + 0.85 * brightness
        lp_state = coeff * allpass_out + (1 - coeff) * lp_state

        # Write back with damping
        buf[idx] = lp_state * damping
        output[i] = lp_state

    return output / (np.max(np.abs(output)) + 1e-10)
```

### Material-Specific Loop Filter Coefficients

The loop filter inside the Karplus-Strong feedback loop determines the material timbre. These are one-pole lowpass coefficients `y[n] = b*x[n] + (1-b)*y[n-1]`:

| Material | Loop Filter Coeff (b) | Damping | Extra |
|----------|----------------------|---------|-------|
| Nylon string | 0.3-0.4 | 0.998 | Very dark, quick HF decay |
| Steel string | 0.5-0.7 | 0.996 | Bright attack, moderate decay |
| Piano wire | 0.7-0.85 | 0.9995 | Bright, very long sustain |
| Metal bar | 0.85-0.95 | 0.9998 | Very bright, bell-like |
| Wood (marimba) | 0.2-0.3 | 0.990 | Dark, fast decay |
| Glass | 0.9-0.99 | 0.9999 | Extremely bright, long ring |
| Rubber | 0.1-0.2 | 0.98 | Very dark, dead sounding |

### Digital Waveguide String with Dispersion

Real strings have frequency-dependent wave speed (dispersion). High harmonics travel slightly faster, causing inharmonicity (piano-like quality). Model with an allpass chain in the delay loop:

```python
def waveguide_string_dispersive(freq, duration=3.0, inharmonicity=0.0003,
                                 pluck_pos=0.25, damping=0.997, sr=44100):
    """Digital waveguide with stiffness-induced dispersion.
    inharmonicity B: 0=ideal string, 0.0001=guitar, 0.001=piano, 0.01=very stiff bar.
    f_n = n * f0 * sqrt(1 + B * n^2) -- partials go progressively sharp.
    """
    n_samples = int(sr * duration)
    delay_total = int(sr / freq)
    half_delay = delay_total // 2

    # Forward and backward delay lines
    fwd = np.zeros(n_samples)
    bwd = np.zeros(n_samples)

    # Pluck excitation: triangular shape
    pluck_idx = max(1, int(pluck_pos * delay_total))
    excitation = np.zeros(delay_total)
    for i in range(delay_total):
        if i <= pluck_idx:
            excitation[i] = i / pluck_idx
        else:
            excitation[i] = (delay_total - i) / (delay_total - pluck_idx)
    excitation *= 0.8

    fwd[:delay_total] = excitation * 0.5
    bwd[:delay_total] = excitation * 0.5

    # Dispersion filter: second-order allpass per waveguide section
    # Coefficient derived from inharmonicity parameter B
    # For small B: allpass coeff ~ -B * (pi * freq / sr)^2
    disp_coeff = -inharmonicity * (np.pi * freq / sr) ** 2
    disp_coeff = np.clip(disp_coeff, -0.9, 0.9)

    # State variables for allpass dispersion filters
    ap_x1, ap_x2, ap_y1, ap_y2 = 0.0, 0.0, 0.0, 0.0
    lp_state = 0.0

    for i in range(delay_total, n_samples):
        # Read from forward delay
        sample = fwd[i - half_delay]

        # Dispersion allpass: H(z) = (a2 + a1*z^-1 + z^-2) / (1 + a1*z^-1 + a2*z^-2)
        a1 = -2.0 * disp_coeff * 0.5
        a2 = disp_coeff
        ap_out = a2 * sample + a1 * ap_x1 + ap_x2 - a1 * ap_y1 - a2 * ap_y2
        ap_x2, ap_x1 = ap_x1, sample
        ap_y2, ap_y1 = ap_y1, ap_out

        # Damping filter (one-pole lowpass)
        lp_state = 0.5 * ap_out + 0.5 * lp_state

        # Reflection with loss
        bwd[i] = -lp_state * damping
        fwd[i] = -bwd[i - half_delay] * damping if i >= half_delay else 0

    # Pickup: sum of both traveling waves at pickup position
    pickup = int(0.15 * delay_total)
    output = np.zeros(n_samples)
    for i in range(n_samples):
        f = fwd[i - pickup] if i >= pickup else 0
        b = bwd[i - (half_delay - pickup)] if i >= (half_delay - pickup) else 0
        output[i] = f + b

    return output / (np.max(np.abs(output)) + 1e-10)
```

### Waveguide Tube Model (Flute/Clarinet)

Cylindrical bore instruments use a waveguide with a reflection filter at the tone-hole end and a nonlinear excitation (jet for flute, reed for clarinet):

```python
def waveguide_flute(freq, duration=2.0, breath_pressure=0.4,
                     jet_delay_ratio=0.4, noise_gain=0.1, sr=44100):
    """Waveguide flute model.
    jet_delay_ratio: jet length / bore length (affects octave transitions).
    Overblowing: increase breath_pressure past 0.6 for second register."""
    n = int(sr * duration)
    bore_delay = max(2, int(sr / (2 * freq)))  # half-wavelength for open tube
    jet_delay = max(1, int(bore_delay * jet_delay_ratio))

    bore_fwd = np.zeros(n)
    bore_bwd = np.zeros(n)
    jet_line = np.zeros(n)
    output = np.zeros(n)

    # Breath envelope (smooth attack)
    breath = np.ones(n) * breath_pressure
    att = int(0.08 * sr)
    breath[:att] = np.linspace(0, breath_pressure, att) ** 2

    lp1, lp2 = 0.0, 0.0

    for i in range(max(bore_delay, jet_delay) + 1, n):
        # Jet nonlinearity: tanh jet deflection model
        jet_sample = jet_line[i - jet_delay] if i >= jet_delay else 0
        jet_input = breath[i] + jet_sample

        # Jet drive with noise (turbulence)
        turbulence = noise_gain * np.random.randn() * breath[i]
        jet_out = np.tanh(jet_input + turbulence) - np.tanh(jet_input)

        # Into bore forward wave
        bore_fwd[i] = jet_out * 0.5

        # Bore end reflection (open end: inverted, with radiation loss)
        reflected = -bore_fwd[i - bore_delay] * 0.95 if i >= bore_delay else 0

        # Tone hole lowpass (models radiation impedance)
        lp1 = 0.7 * reflected + 0.3 * lp1
        bore_bwd[i] = lp1

        # Feedback into jet delay line
        bore_feedback = bore_bwd[i - bore_delay] if i >= bore_delay else 0
        lp2 = 0.6 * bore_feedback + 0.4 * lp2
        jet_line[i] = -lp2 * 0.4

        output[i] = bore_fwd[i] + bore_bwd[i]

    return output / (np.max(np.abs(output)) + 1e-10)
```

### Commuted Synthesis

Key insight for efficiency: the body resonance of instruments (guitar body, piano soundboard) is a linear filter. Instead of modeling it in real-time, convolve the excitation signal with the body impulse response, then feed that into the string model. This is called "commuted" because we commute the order of operations.

```python
def commuted_guitar(freq, duration=2.0, body_ir=None, sr=44100):
    """Commuted synthesis guitar. Body IR pre-filters the excitation.
    If no IR provided, generates a synthetic guitar body response."""
    if body_ir is None:
        # Synthetic guitar body: resonances at ~100, 200, 400 Hz
        ir_len = int(0.05 * sr)
        body_ir = np.zeros(ir_len)
        for res_freq, res_amp, res_decay in [(100, 1.0, 0.02), (200, 0.7, 0.015),
                                               (400, 0.5, 0.01), (800, 0.2, 0.005)]:
            t_ir = np.arange(ir_len) / sr
            body_ir += res_amp * np.sin(2*np.pi*res_freq*t_ir) * np.exp(-t_ir/res_decay)
        body_ir /= np.max(np.abs(body_ir))

    # Excitation: noise burst convolved with body IR
    burst_len = min(int(sr / freq), int(sr * 0.005))
    burst = np.random.randn(burst_len) * np.exp(-np.arange(burst_len) / (burst_len * 0.3))

    from scipy.signal import fftconvolve
    excitation = fftconvolve(burst, body_ir, mode='full')[:int(sr / freq)]

    # Feed into Karplus-Strong
    delay_len = max(2, int(sr / freq))
    n = int(sr * duration)
    out = np.zeros(n)
    out[:len(excitation)] = excitation

    lp_state = 0.0
    for i in range(delay_len, n):
        lp_state = 0.6 * out[i - delay_len] + 0.4 * lp_state
        out[i] += lp_state * 0.996

    return out / (np.max(np.abs(out)) + 1e-10)
```

### Membrane Model (Drums via 2D Waveguide)

```python
def waveguide_membrane(freq=80, duration=0.5, tension=0.5, damping=0.95, sr=44100):
    """Simplified 2D waveguide membrane for drum synthesis.
    Models circular membrane modes: f_mn = f_01 * bessel_zeros[m][n] / bessel_zeros[0][1].
    Key Bessel zero ratios for circular membrane:
    (0,1)=1.000, (1,1)=1.594, (2,1)=2.136, (0,2)=2.296, (3,1)=2.653, (1,2)=2.918
    """
    # Circular membrane mode ratios (from Bessel function zeros)
    mode_ratios = [1.000, 1.594, 2.136, 2.296, 2.653, 2.918, 3.156, 3.501]
    mode_amps = [1.0, 0.8, 0.6, 0.5, 0.3, 0.2, 0.15, 0.1]
    mode_decays = [0.3, 0.2, 0.15, 0.12, 0.08, 0.06, 0.05, 0.04]

    n = int(sr * duration)
    t = np.arange(n) / sr
    output = np.zeros(n)

    for ratio, amp, decay in zip(mode_ratios, mode_amps, mode_decays):
        mode_freq = freq * ratio * (0.5 + tension * 0.5)
        if mode_freq >= sr / 2:
            continue
        decay_adj = decay * damping * 2
        output += amp * np.sin(2 * np.pi * mode_freq * t) * np.exp(-t / decay_adj)

    # Excitation envelope
    attack = int(0.001 * sr)
    output[:attack] *= np.linspace(0, 1, attack)

    return output / (np.max(np.abs(output)) + 1e-10)
```

---

## 3. FM Synthesis Deep Dive

### Phase Modulation vs Frequency Modulation

The DX7 and most "FM" synths actually use Phase Modulation (PM), not true FM. The distinction matters:

- True FM: `y(t) = sin(2*pi*fc*t + I * integral(sin(2*pi*fm*t)))`
- Phase Modulation: `y(t) = sin(2*pi*fc*t + I * sin(2*pi*fm*t))`

PM is self-consistent (no DC drift from integration), and the spectrum is identical to FM when the modulator is a sine wave. The DX7 uses PM.

```python
def pm_operator(carrier_freq, mod_signal, mod_index, duration, sr=44100):
    """Single PM operator. mod_signal can be another operator's output."""
    t = np.linspace(0, duration, int(sr * duration), endpoint=False)
    phase = 2 * np.pi * carrier_freq * t
    return np.sin(phase + mod_index * mod_signal)
```

### DX7 Algorithm Topology

The DX7 has 6 operators arranged in 32 algorithms. Key algorithm patterns:

```python
def dx7_algorithm(freqs, ratios, indices, envelopes, algorithm, duration, sr=44100):
    """Generic DX7-style 6-operator FM.
    freqs: base frequency for each op (usually all same, scaled by ratio).
    ratios: frequency ratio for each operator (1-6).
    indices: modulation index for each operator (0-5).
    envelopes: amplitude envelope for each operator, shape (6, n_samples).
    algorithm: int 1-32 (see topology below).

    Key algorithms:
    1: [6->5->4->3->2->1]  -- serial chain, very complex spectrum
    5: [6->5->4->3], [2->1], out=1+3  -- two chains mixed
    7: [6->5->4], [3->2->1], out=1+4  -- two 3-op stacks
    13: [6->5], [4->3], [2->1], out=1+3+5  -- three parallel 2-op pairs
    32: [1]+[2]+[3]+[4]+[5]+[6]  -- all carriers (additive)
    """
    n = int(sr * duration)
    t = np.linspace(0, duration, n, endpoint=False)
    ops = [np.zeros(n) for _ in range(6)]

    # Compute each operator's base phase
    phases = [2 * np.pi * freqs[i] * ratios[i] * t for i in range(6)]

    if algorithm == 1:
        # Serial: 6->5->4->3->2->1, output = op1
        ops[5] = envelopes[5] * np.sin(phases[5])
        ops[4] = envelopes[4] * np.sin(phases[4] + indices[5] * ops[5])
        ops[3] = envelopes[3] * np.sin(phases[3] + indices[4] * ops[4])
        ops[2] = envelopes[2] * np.sin(phases[2] + indices[3] * ops[3])
        ops[1] = envelopes[1] * np.sin(phases[1] + indices[2] * ops[2])
        ops[0] = envelopes[0] * np.sin(phases[0] + indices[1] * ops[1])
        return ops[0]

    elif algorithm == 5:
        # Two stacks: 6->5->4->3 and 2->1, out = op1 + op3
        ops[5] = envelopes[5] * np.sin(phases[5])
        ops[4] = envelopes[4] * np.sin(phases[4] + indices[5] * ops[5])
        ops[3] = envelopes[3] * np.sin(phases[3] + indices[4] * ops[4])
        ops[2] = envelopes[2] * np.sin(phases[2] + indices[3] * ops[3])
        ops[1] = envelopes[1] * np.sin(phases[1])
        ops[0] = envelopes[0] * np.sin(phases[0] + indices[1] * ops[1])
        return ops[0] + ops[2]

    elif algorithm == 32:
        # All carriers (pure additive)
        for i in range(6):
            ops[i] = envelopes[i] * np.sin(phases[i])
        return sum(ops)

    # Default: algorithm 1
    ops[5] = envelopes[5] * np.sin(phases[5])
    for i in range(4, -1, -1):
        ops[i] = envelopes[i] * np.sin(phases[i] + indices[i+1] * ops[i+1])
    return ops[0]
```

### Feedback FM

An operator modulating itself creates a rich, saw-like waveform. The DX7 allows feedback on one operator per algorithm:

```python
def fm_feedback_operator(freq, feedback_amount, duration, sr=44100):
    """Self-modulating FM operator (feedback FM).
    feedback_amount: 0=sine, 0.5=saw-like, 1.0=chaotic.
    Must be computed sample-by-sample (recursive dependency).
    """
    n = int(sr * duration)
    output = np.zeros(n)
    phase = 0.0
    dt = 2 * np.pi * freq / sr
    prev_out = 0.0

    for i in range(n):
        output[i] = np.sin(phase + feedback_amount * prev_out)
        prev_out = output[i]
        phase += dt

    return output
```

| Feedback Amount | Character |
|----------------|-----------|
| 0.0 | Pure sine |
| 0.2-0.4 | Warm, slightly bright (like soft saw) |
| 0.5-0.7 | Sawtooth-like, rich harmonics |
| 0.8-1.0 | Harsh, buzzy |
| >1.2 | Chaotic, noise-like |
| pi | Full chaos (white noise) |

### Classic FM Patch Recipes

#### DX7 Electric Piano (Rhodes-like)

```python
def fm_electric_piano(freq, duration=2.0, velocity=0.8, sr=44100):
    """Classic DX7 E.Piano 1. Algorithm 5 topology.
    Key insight: modulation index decays faster than carrier amplitude.
    Velocity controls both volume and brightness (higher velocity = more harmonics).
    """
    n = int(sr * duration)
    t = np.linspace(0, duration, n, endpoint=False)

    # Modulation index envelope: fast decay (tine brightness fades quickly)
    mod_env = velocity * 2.5 * np.exp(-t * 8.0)

    # Carrier envelope: slower decay (tone sustains longer)
    car_env = velocity * np.exp(-t * 1.5)

    # Operator 2 (modulator): ratio 14:1 gives the "tine" attack
    mod1 = mod_env * np.sin(2 * np.pi * freq * 14.0 * t)

    # Operator 1 (carrier): fundamental
    carrier = car_env * np.sin(2 * np.pi * freq * t + mod1)

    # Second pair for body: ratio 1:1 with lower index
    mod2_env = velocity * 1.0 * np.exp(-t * 3.0)
    mod2 = mod2_env * np.sin(2 * np.pi * freq * 1.0 * t)
    body = car_env * 0.6 * np.sin(2 * np.pi * freq * t + mod2)

    output = carrier + body

    # Velocity-dependent lowpass (softer = darker)
    cutoff = 1000 + velocity * 6000
    sos = butter(2, min(cutoff, sr/2 - 100), btype='low', fs=sr, output='sos')
    output = sosfilt(sos, output)

    return output / (np.max(np.abs(output)) + 1e-10)
```

#### FM Bass

```python
def fm_bass(freq, duration=0.5, drive=0.7, sr=44100):
    """Punchy FM bass. 1:1 ratio, index envelope for pluck character."""
    n = int(sr * duration)
    t = np.linspace(0, duration, n, endpoint=False)

    # Sharp index envelope: high initial mod creates bright attack, then settles
    mod_index = drive * 5.0 * np.exp(-t * 15.0) + drive * 0.5

    modulator = mod_index * np.sin(2 * np.pi * freq * t)
    carrier = np.sin(2 * np.pi * freq * t + modulator)

    # Amplitude envelope
    amp_env = np.exp(-t * 3.0)
    amp_env[:int(0.005 * sr)] *= np.linspace(0, 1, int(0.005 * sr))

    return carrier * amp_env
```

#### FM Bells

```python
def fm_bell(freq, duration=4.0, sr=44100):
    """FM bell/chime. Inharmonic ratios create metallic character.
    C:M = 1:1.414 (sqrt(2)) or 1:3.5 for different bell types."""
    n = int(sr * duration)
    t = np.linspace(0, duration, n, endpoint=False)

    # Two modulator pairs for complex bell spectrum
    mod1_env = 8.0 * np.exp(-t * 1.5)
    mod1 = mod1_env * np.sin(2 * np.pi * freq * 3.5 * t)

    mod2_env = 3.0 * np.exp(-t * 2.0)
    mod2 = mod2_env * np.sin(2 * np.pi * freq * 7.01 * t)  # slightly detuned

    carrier = np.sin(2 * np.pi * freq * t + mod1 + mod2)

    # Long amplitude decay
    amp = np.exp(-t * 0.8)
    amp[:int(0.001 * sr)] *= np.linspace(0, 1, int(0.001 * sr))

    return carrier * amp
```

#### FM Brass

```python
def fm_brass(freq, duration=1.0, attack_time=0.05, sr=44100):
    """FM brass. 1:1 ratio. Mod index envelope mimics filter opening on attack."""
    n = int(sr * duration)
    t = np.linspace(0, duration, n, endpoint=False)

    # Index envelope: rises with attack (brass blare), sustains, then falls
    att_samples = int(attack_time * sr)
    mod_index = np.zeros(n)
    mod_index[:att_samples] = np.linspace(0.5, 5.0, att_samples)
    mod_index[att_samples:] = 5.0 * np.exp(-(t[att_samples:] - attack_time) * 2.0) + 1.5

    modulator = mod_index * np.sin(2 * np.pi * freq * t)
    carrier = np.sin(2 * np.pi * freq * t + modulator)

    # Amplitude envelope
    amp = np.ones(n)
    amp[:att_samples] = np.linspace(0, 1, att_samples) ** 0.5
    rel = int(0.1 * sr)
    amp[-rel:] = np.linspace(1, 0, rel)

    return carrier * amp
```

### Multi-Carrier FM and Modulation Index Mathematics

The FM spectrum of `sin(wc*t + I*sin(wm*t))` produces sidebands at `fc +/- n*fm` with amplitudes given by Bessel functions of the first kind:

```
Amplitude of sideband n = J_n(I)
```

where I is the modulation index. Key properties:
- Number of significant sidebands ~ I + 1
- Bandwidth ~ 2 * fm * (I + 1) (Carson's rule)
- As I increases, energy spreads to more sidebands
- J_0(I) = 0 at I ~ 2.4, 5.5, 8.65... (fundamental disappears!)

```python
from scipy.special import jn as bessel_j

def fm_spectrum_amplitudes(mod_index, n_sidebands=20):
    """Predict FM sideband amplitudes using Bessel functions."""
    amplitudes = {}
    for n in range(-n_sidebands, n_sidebands + 1):
        amp = bessel_j(abs(n), mod_index)
        if n < 0 and n % 2 != 0:
            amp = -amp  # J_-n(x) = (-1)^n * J_n(x)
        amplitudes[n] = amp
    return amplitudes

# Example: mod_index=1 -> ~2 significant sideband pairs
# mod_index=5 -> ~6 significant sideband pairs
# mod_index=10 -> ~11 significant sideband pairs
```

---

## 4. Subtractive Synthesis -- Advanced Filter Modeling

### Moog Ladder Filter (4-pole, 24dB/oct)

The Moog ladder filter is four cascaded one-pole sections with nonlinear feedback. The key is the tanh saturation in the feedback path that creates the warm, fat sound:

```python
def moog_ladder(sig, cutoff, resonance=0.0, drive=1.0, sr=44100):
    """Moog ladder filter with nonlinear feedback.
    cutoff: Hz. resonance: 0-1 (1 = self-oscillation).
    drive: input saturation (1=clean, 2-4=warm, >4=aggressive).

    Based on Huovilainen's improved model (2004).
    """
    n = len(sig)
    output = np.zeros(n)

    # Four integrator states
    s = [0.0, 0.0, 0.0, 0.0]

    # Compensation for resonance volume loss
    comp = 1.0 + resonance * 0.5

    for i in range(n):
        # Get per-sample cutoff (allow arrays)
        fc = cutoff[i] if hasattr(cutoff, '__len__') else cutoff
        fc = min(fc, sr * 0.45)

        # Frequency warping for better HF accuracy
        g = np.tan(np.pi * fc / sr)
        G = g / (1.0 + g)

        # Feedback with nonlinear saturation (the "Moog sound")
        feedback = resonance * 4.0 * np.tanh(s[3])

        # Input with drive saturation
        x = np.tanh(sig[i] * drive) * comp - feedback

        # Four cascaded one-pole filters
        for j in range(4):
            v = G * (np.tanh(x) - np.tanh(s[j]))
            y = v + s[j]
            s[j] = y + v  # trapezoidal integration
            x = y

        output[i] = y

    return output

def moog_ladder_vectorized(sig, cutoff, resonance=0.0, sr=44100):
    """Faster Moog ladder using numpy where possible.
    Still requires per-sample loop for the nonlinear feedback."""
    n = len(sig)
    output = np.zeros(n)
    s = np.zeros(4)

    if not hasattr(cutoff, '__len__'):
        cutoff = np.full(n, cutoff)

    fc = np.clip(cutoff, 20, sr * 0.45)
    g = np.tan(np.pi * fc / sr)
    G = g / (1.0 + g)

    for i in range(n):
        fb = resonance * 4.0 * np.tanh(s[3])
        x = np.tanh(sig[i]) - fb

        for j in range(4):
            v = G[i] * (np.tanh(x) - np.tanh(s[j]))
            y = v + s[j]
            s[j] = y + v
            x = y

        output[i] = y

    return output
```

### Roland TB-303 Diode Ladder Filter

The 303's distinctive "acid" sound comes from its diode ladder filter, which has different nonlinear characteristics than the Moog:

```python
def tb303_filter(sig, cutoff, resonance=0.5, env_mod=0.0,
                  accent=False, sr=44100):
    """TB-303 diode ladder filter model.
    Key differences from Moog:
    - Diode clipping (softer than tanh)
    - Resonance affects cutoff tracking
    - Accent increases resonance AND envelope depth simultaneously
    """
    n = len(sig)
    output = np.zeros(n)

    if accent:
        resonance = min(resonance + 0.3, 0.95)
        env_mod *= 1.5

    s = [0.0, 0.0, 0.0, 0.0]

    for i in range(n):
        fc = cutoff[i] if hasattr(cutoff, '__len__') else cutoff
        fc = min(fc, sr * 0.45)

        # 303's cutoff tracking with resonance interaction
        # Higher resonance slightly reduces effective cutoff
        fc_eff = fc * (1.0 - resonance * 0.15)

        g = np.tan(np.pi * fc_eff / sr)
        G = g / (1.0 + g)

        # Diode clipping: softer than tanh
        def diode_clip(x):
            return x / (1.0 + abs(x))

        # Feedback
        feedback = resonance * 4.0 * diode_clip(s[3])
        x = diode_clip(sig[i] * 1.5) - feedback

        for j in range(4):
            v = G * (diode_clip(x) - diode_clip(s[j]))
            y = v + s[j]
            s[j] = y + v
            x = y

        output[i] = y

    return output
```

### Oberheim SEM State-Variable Filter (12dB/oct, 2-pole)

The SEM filter has a distinctive character: smooth resonance, continuous blending between LP/BP/HP, and a notch mode. It uses the state-variable filter topology:

```python
def oberheim_sem(sig, cutoff, resonance=0.5, mode='low', sr=44100):
    """Oberheim SEM 2-pole state-variable filter.
    mode: 'low', 'band', 'high', 'notch', or float 0-1 for LP-HP blend.
    The SEM's character: musical resonance that doesn't thin out the bass.
    """
    n = len(sig)
    output = np.zeros(n)

    low = band = 0.0
    q = 1.0 / max(0.5 + resonance * 10, 0.5)  # Q scaling for SEM character

    for i in range(n):
        fc = cutoff[i] if hasattr(cutoff, '__len__') else cutoff
        fc = min(fc, sr * 0.45)

        # Frequency coefficient with pre-warping
        f = 2.0 * np.sin(np.pi * fc / sr)

        # State-variable filter update
        high = sig[i] - low - q * band
        band_new = f * high + band
        low_new = f * band_new + low

        # Soft saturation on the band state (SEM character)
        band = np.tanh(band_new)
        low = low_new

        if mode == 'low':
            output[i] = low
        elif mode == 'band':
            output[i] = band
        elif mode == 'high':
            output[i] = high
        elif mode == 'notch':
            output[i] = low + high
        elif isinstance(mode, (int, float)):
            # Continuous LP->HP morph
            output[i] = low * (1 - mode) + high * mode

    return output
```

### Zero-Delay Feedback (ZDF) Filter Topology

Traditional digital filters (like the Chamberlin SVF) have an implicit one-sample delay in the feedback path, causing frequency warping and resonance instability at high frequencies. The ZDF approach solves this using the trapezoidal integrator (bilinear transform applied at the topology level):

```python
def zdf_svf(sig, cutoff, resonance=0.5, mode='low', sr=44100):
    """Zero-Delay Feedback state-variable filter.
    Based on Zavalishin's "The Art of VA Filter Design" (2012).
    Exact frequency response at all frequencies, no warping artifacts.
    """
    n = len(sig)
    output = np.zeros(n)

    # State variables (integrator memories)
    ic1eq = 0.0  # first integrator state
    ic2eq = 0.0  # second integrator state

    for i in range(n):
        fc = cutoff[i] if hasattr(cutoff, '__len__') else cutoff
        fc = min(fc, sr * 0.49)

        # Pre-warped frequency coefficient
        g = np.tan(np.pi * fc / sr)
        k = 2.0 - 2.0 * resonance  # damping (resonance 0-1 maps to k 2-0)

        # Coefficient calculations
        a1 = 1.0 / (1.0 + g * (g + k))
        a2 = g * a1
        a3 = g * a2

        # Solve the implicit equation (ZDF magic -- no delay in feedback)
        v3 = sig[i] - ic2eq
        v1 = a1 * ic1eq + a2 * v3
        v2 = ic2eq + a2 * ic1eq + a3 * v3

        # Update integrator states
        ic1eq = 2 * v1 - ic1eq
        ic2eq = 2 * v2 - ic2eq

        # Output modes
        if mode == 'low':
            output[i] = v2
        elif mode == 'band':
            output[i] = v1
        elif mode == 'high':
            output[i] = sig[i] - k * v1 - v2
        elif mode == 'notch':
            output[i] = sig[i] - k * v1
        elif mode == 'peak':
            output[i] = sig[i] - k * v1 - 2 * v2
        elif mode == 'allpass':
            output[i] = sig[i] - 2 * k * v1

    return output
```

### Analog Warmth: What Creates It Mathematically

"Analog warmth" is the combined effect of several nonlinear behaviors:

1. Soft saturation: `tanh(x)` in feedback paths generates even harmonics
2. Component variation: Capacitor/resistor tolerances cause slight detuning between filter stages
3. Thermal noise: Low-level noise floor that "fills in" the sound
4. Asymmetric clipping: Even harmonics from asymmetric transfer curves

```python
def analog_warmth(sig, amount=0.5, sr=44100):
    """Add analog-style warmth to a signal.
    Models: soft saturation + subtle even harmonics + noise floor + HF rolloff.
    amount: 0=clean, 0.3=subtle, 0.7=obvious, 1.0=heavy.
    """
    # 1. Soft saturation (even harmonics)
    driven = sig * (1 + amount * 2)
    warm = np.tanh(driven) / (1 + amount)

    # 2. Asymmetric saturation (more even harmonics)
    pos = np.tanh(warm * (1 + amount * 0.3))
    neg = np.tanh(warm * (1 - amount * 0.1))
    warm = np.where(warm >= 0, pos, neg)

    # 3. Subtle noise floor (-80 dB typical analog)
    noise_level = amount * 0.0001
    warm += np.random.randn(len(sig)) * noise_level

    # 4. Gentle HF rolloff (analog circuits have limited bandwidth)
    fc = 18000 - amount * 6000  # 18kHz at 0, 12kHz at max
    sos = butter(1, min(fc, sr/2 - 100), btype='low', fs=sr, output='sos')
    warm = sosfilt(sos, warm)

    return warm
```

---

## 5. Granular Synthesis for Production

### Grain Scheduling Algorithms

The existing codebase has basic granular synthesis. Here are the production-grade extensions:

#### Synchronous Granular Synthesis

Grains are triggered at regular intervals locked to the source pitch. Essential for pitch-shifting without artifacts:

```python
def granular_pitch_shift(source, semitones, grain_size_ms=40,
                          overlap=4, sr=44100):
    """Synchronous granular pitch shifter.
    overlap: number of overlapping grains (2-8, 4 is standard).
    Uses Hann window with proper overlap-add for seamless output.
    """
    n = len(source)
    ratio = 2 ** (semitones / 12.0)
    grain_samples = int(grain_size_ms * sr / 1000)
    hop = grain_samples // overlap
    window = np.hanning(grain_samples)

    output = np.zeros(n)
    win_sum = np.zeros(n)

    # Read pointer advances at different rate than write pointer
    read_pos = 0.0
    write_pos = 0

    while write_pos + grain_samples < n:
        read_idx = int(read_pos) % (n - grain_samples)

        # Extract and window grain
        grain = source[read_idx:read_idx + grain_samples] * window

        # Overlap-add at write position
        end = min(write_pos + grain_samples, n)
        length = end - write_pos
        output[write_pos:end] += grain[:length]
        win_sum[write_pos:end] += window[:length]

        # Advance pointers
        write_pos += hop
        read_pos += hop * ratio  # read faster/slower for pitch shift

    # Normalize by window sum to maintain amplitude
    output /= np.maximum(win_sum, 1e-10)
    return output
```

#### Asynchronous Granular (Cloud) Synthesis

Grains triggered stochastically, creating texture clouds. Key parameters and their perceptual effects:

```python
def granular_cloud(source, duration, grain_size_ms=50, density=30,
                    position=0.5, position_spread=0.1,
                    pitch_spread_cents=0, amp_spread=0.0,
                    window_type='hann', sr=44100):
    """Production-grade granular cloud generator.
    position: 0-1, where in source to read from.
    position_spread: randomization of read position.
    pitch_spread_cents: random pitch variation per grain.
    amp_spread: random amplitude variation (0-1).
    window_type: 'hann', 'gaussian', 'tukey', 'triangle'.
    """
    n_out = int(sr * duration)
    n_src = len(source)
    grain_samples = int(grain_size_ms * sr / 1000)
    output = np.zeros(n_out)

    # Window selection (affects grain character)
    if window_type == 'hann':
        window = np.hanning(grain_samples)
    elif window_type == 'gaussian':
        # Gaussian: sigma=0.4 gives good overlap behavior
        x = np.linspace(-3, 3, grain_samples)
        window = np.exp(-0.5 * (x / 0.4) ** 2)
    elif window_type == 'tukey':
        # Tukey: flat top with cosine tapers (good for longer grains)
        from scipy.signal import windows
        window = windows.tukey(grain_samples, alpha=0.5)
    elif window_type == 'triangle':
        window = 1.0 - np.abs(np.linspace(-1, 1, grain_samples))

    n_grains = int(duration * density)

    for _ in range(n_grains):
        # Random output position
        out_pos = np.random.randint(0, max(1, n_out - grain_samples))

        # Source position with spread
        src_center = int(position * n_src)
        src_spread = int(position_spread * n_src)
        src_pos = src_center + np.random.randint(-src_spread, src_spread + 1)
        src_pos = np.clip(src_pos, 0, n_src - grain_samples)

        grain = source[src_pos:src_pos + grain_samples].copy()

        # Pitch variation via resampling
        if pitch_spread_cents > 0:
            cents = np.random.uniform(-pitch_spread_cents, pitch_spread_cents)
            ratio = 2 ** (cents / 1200.0)
            if abs(ratio - 1.0) > 0.001:
                new_len = int(len(grain) / ratio)
                indices = np.linspace(0, len(grain) - 1, new_len)
                idx0 = np.clip(indices.astype(int), 0, len(grain) - 2)
                frac = indices - idx0
                grain = grain[idx0] * (1 - frac) + grain[idx0 + 1] * frac
                # Re-window to new length
                if len(grain) != grain_samples:
                    if window_type == 'hann':
                        win = np.hanning(len(grain))
                    else:
                        win = np.hanning(len(grain))  # fallback
                    grain *= win
                else:
                    grain *= window
            else:
                grain *= window
        else:
            grain *= window

        # Amplitude variation
        if amp_spread > 0:
            grain *= 1.0 - np.random.uniform(0, amp_spread)

        # Add to output
        end = min(out_pos + len(grain), n_out)
        output[out_pos:end] += grain[:end - out_pos]

    return output / (np.max(np.abs(output)) + 1e-10)
```

#### Granular Time-Stretching

```python
def granular_time_stretch(source, stretch_factor, grain_size_ms=60,
                           overlap=4, sr=44100):
    """Time-stretch without pitch change using granular overlap-add.
    stretch_factor: >1 = slower, <1 = faster.
    Works better than phase vocoder for transient-heavy material.
    """
    n_src = len(source)
    n_out = int(n_src * stretch_factor)
    grain_samples = int(grain_size_ms * sr / 1000)
    hop_out = grain_samples // overlap
    hop_in = int(hop_out / stretch_factor)
    window = np.hanning(grain_samples)

    output = np.zeros(n_out)
    win_sum = np.zeros(n_out)

    read_pos = 0
    write_pos = 0

    while write_pos + grain_samples < n_out and read_pos + grain_samples < n_src:
        grain = source[read_pos:read_pos + grain_samples] * window

        end = min(write_pos + grain_samples, n_out)
        length = end - write_pos
        output[write_pos:end] += grain[:length]
        win_sum[write_pos:end] += window[:length]

        read_pos += hop_in
        write_pos += hop_out

    output /= np.maximum(win_sum, 1e-10)
    return output
```

### Window Function Selection Guide

| Window | Overlap Ratio | Character | Best For |
|--------|--------------|-----------|----------|
| Hann | 50-75% | Smooth, general purpose | Standard granular |
| Gaussian (s=0.4) | 60-80% | Very smooth, no clicks | Ambient pads |
| Triangle | 50% | Slight edge, efficient | Rhythmic granular |
| Tukey (a=0.5) | 30-50% | Flat top preserves content | Time-stretching |
| Blackman-Harris | 75% | Ultra-smooth transitions | Spectral work |

### Ableton Granulator Behavior

Ableton's Granulator II by Robert Henke uses these principles:
- Grain sizes: 1-500ms (default ~50ms)
- Density linked to grain size for constant coverage
- Spray parameter = position randomization
- Scan = automated source position movement
- File position mapped to a 2D X-Y pad with interpolation
- Critical detail: it uses 3x overlap with Hann windows as default

---

## 6. Additive Synthesis and Resynthesis

### Efficient Additive via Inverse FFT (Overlap-Add)

Summing thousands of sine waves individually is expensive. Instead, place partial amplitudes/phases directly into an FFT bin array and use inverse FFT:

```python
def additive_ifft(partials_freq, partials_amp, partials_phase,
                   duration, fft_size=4096, hop_size=1024, sr=44100):
    """Efficient additive synthesis using inverse FFT overlap-add.
    partials_freq: array of shape (n_partials,) or (n_frames, n_partials) for time-varying.
    partials_amp: same shape as partials_freq.
    partials_phase: initial phases, shape (n_partials,).

    This is ~100x faster than summing individual sines for >50 partials.
    """
    n_samples = int(sr * duration)
    n_frames = n_samples // hop_size + 1
    n_bins = fft_size // 2 + 1
    window = np.hanning(fft_size)

    output = np.zeros(n_samples + fft_size)
    win_sum = np.zeros(n_samples + fft_size)

    # Handle static vs time-varying partials
    if partials_freq.ndim == 1:
        p_freq = np.tile(partials_freq, (n_frames, 1))
        p_amp = np.tile(partials_amp, (n_frames, 1))
    else:
        p_freq = partials_freq[:n_frames]
        p_amp = partials_amp[:n_frames]

    freq_resolution = sr / fft_size
    phases = partials_phase.copy()

    for frame_idx in range(n_frames):
        spectrum = np.zeros(n_bins, dtype=complex)

        for p in range(len(phases)):
            freq = p_freq[frame_idx, p] if frame_idx < len(p_freq) else p_freq[-1, p]
            amp = p_amp[frame_idx, p] if frame_idx < len(p_amp) else p_amp[-1, p]

            if freq <= 0 or freq >= sr / 2 or amp < 1e-6:
                continue

            # Map frequency to nearest FFT bin
            bin_idx = int(round(freq / freq_resolution))
            if 0 < bin_idx < n_bins:
                spectrum[bin_idx] += amp * np.exp(1j * phases[p])

            # Advance phase
            phases[p] += 2 * np.pi * freq * hop_size / sr
            phases[p] %= 2 * np.pi

        # Inverse FFT and overlap-add
        frame = np.fft.irfft(spectrum, n=fft_size) * window
        start = frame_idx * hop_size
        output[start:start + fft_size] += frame
        win_sum[start:start + fft_size] += window ** 2

    output /= np.maximum(win_sum, 1e-10)
    return output[:n_samples]
```

### Partial Tracking (Analysis)

Extract individual partials from a recording for resynthesis or manipulation:

```python
def track_partials(sig, n_partials=30, fft_size=4096, hop_size=1024,
                    min_amp_db=-60, sr=44100):
    """Track sinusoidal partials in an audio signal.
    Returns arrays of (frequency, amplitude, phase) per frame.
    Based on McAulay-Quatieri partial tracking.
    """
    n_frames = (len(sig) - fft_size) // hop_size + 1
    window = np.blackman(fft_size)  # Blackman for better frequency resolution
    freq_resolution = sr / fft_size

    # Output arrays
    freqs = np.zeros((n_frames, n_partials))
    amps = np.zeros((n_frames, n_partials))
    phases_out = np.zeros((n_frames, n_partials))

    min_amp = 10 ** (min_amp_db / 20)

    for i in range(n_frames):
        start = i * hop_size
        frame = sig[start:start + fft_size] * window
        spectrum = np.fft.rfft(frame)
        magnitudes = np.abs(spectrum)
        phase = np.angle(spectrum)

        # Find spectral peaks (local maxima above threshold)
        peaks = []
        for b in range(1, len(magnitudes) - 1):
            if (magnitudes[b] > magnitudes[b-1] and
                magnitudes[b] > magnitudes[b+1] and
                magnitudes[b] > min_amp):

                # Parabolic interpolation for sub-bin frequency precision
                alpha = np.log(magnitudes[b-1] + 1e-20)
                beta = np.log(magnitudes[b] + 1e-20)
                gamma = np.log(magnitudes[b+1] + 1e-20)
                p = 0.5 * (alpha - gamma) / (alpha - 2*beta + gamma)

                true_freq = (b + p) * freq_resolution
                true_amp = magnitudes[b]  # could interpolate amplitude too
                true_phase = phase[b]

                peaks.append((true_freq, true_amp, true_phase))

        # Sort by amplitude, keep top n_partials
        peaks.sort(key=lambda x: -x[1])
        peaks = peaks[:n_partials]

        # Sort by frequency for consistent ordering
        peaks.sort(key=lambda x: x[0])

        for j, (f, a, p) in enumerate(peaks):
            freqs[i, j] = f
            amps[i, j] = a
            phases_out[i, j] = p

    return freqs, amps, phases_out
```

### Spectral Modeling Synthesis (SMS)

SMS decomposes sound into deterministic (sinusoidal) + stochastic (residual) components:

```python
def sms_analyze(sig, n_partials=30, fft_size=4096, hop_size=1024, sr=44100):
    """SMS analysis: decompose into sinusoidal + residual components."""
    # 1. Track partials
    freqs, amps, phases = track_partials(sig, n_partials, fft_size, hop_size, sr=sr)

    # 2. Resynthesize the sinusoidal component
    n_frames = len(freqs)
    n_samples = len(sig)
    sinusoidal = np.zeros(n_samples)

    for p in range(n_partials):
        phase = 0.0
        for i in range(n_frames - 1):
            start = i * hop_size
            end = min(start + hop_size, n_samples)
            length = end - start

            # Interpolate frequency and amplitude across hop
            f0, f1 = freqs[i, p], freqs[i+1, p]
            a0, a1 = amps[i, p], amps[i+1, p]

            if f0 < 1 and f1 < 1:
                continue

            t = np.arange(length) / length
            freq_interp = f0 + (f1 - f0) * t
            amp_interp = a0 + (a1 - a0) * t

            for j in range(length):
                if start + j < n_samples:
                    sinusoidal[start + j] += amp_interp[j] * np.sin(phase)
                phase += 2 * np.pi * freq_interp[j] / sr

    # 3. Residual = original - sinusoidal
    residual = sig[:n_samples] - sinusoidal[:n_samples]

    # 4. Model residual as time-varying spectral envelope
    residual_env = []
    for i in range(n_frames):
        start = i * hop_size
        frame = residual[start:start + fft_size] if start + fft_size <= n_samples else \
                np.pad(residual[start:], (0, fft_size - (n_samples - start)))
        env = np.abs(np.fft.rfft(frame * np.hanning(fft_size)))
        residual_env.append(env)

    return freqs, amps, phases, np.array(residual_env)

def sms_resynthesize(freqs, amps, phases, residual_env,
                      fft_size=4096, hop_size=1024, sr=44100):
    """SMS resynthesis: reconstruct from sinusoidal + stochastic model.
    Can modify partials before resynthesis (transpose, time-stretch, morph)."""
    n_frames = len(freqs)
    n_samples = n_frames * hop_size + fft_size

    # Sinusoidal part (additive)
    sinusoidal = np.zeros(n_samples)
    for p in range(freqs.shape[1]):
        phase = phases[0, p] if len(phases) > 0 else 0
        for i in range(n_frames - 1):
            start = i * hop_size
            f = freqs[i, p]
            a = amps[i, p]
            if f < 1:
                continue
            for j in range(hop_size):
                if start + j < n_samples:
                    sinusoidal[start + j] += a * np.sin(phase)
                phase += 2 * np.pi * f / sr

    # Stochastic part: filtered white noise matching residual envelope
    stochastic = np.zeros(n_samples)
    window = np.hanning(fft_size)
    win_sum = np.zeros(n_samples)

    for i in range(n_frames):
        # Generate noise shaped by residual spectral envelope
        noise_spec = residual_env[i] * np.exp(1j * np.random.uniform(0, 2*np.pi, len(residual_env[i])))
        noise_frame = np.fft.irfft(noise_spec, n=fft_size) * window

        start = i * hop_size
        end = min(start + fft_size, n_samples)
        length = end - start
        stochastic[start:end] += noise_frame[:length]
        win_sum[start:end] += window[:length] ** 2

    stochastic /= np.maximum(win_sum, 1e-10)

    return sinusoidal + stochastic
```

### Creating Evolving Pads with Time-Varying Partials

```python
def evolving_pad(freq, duration, n_partials=20, sr=44100):
    """Create an evolving pad by slowly modulating partial amplitudes.
    Each partial has its own LFO rate (incommensurate for non-repeating pattern).
    """
    n = int(sr * duration)
    t = np.arange(n) / sr
    output = np.zeros(n)

    # Golden ratio-based LFO rates for each partial (maximally incommensurate)
    golden = (1 + np.sqrt(5)) / 2
    base_lfo_rate = 0.05  # Hz, very slow

    for i in range(n_partials):
        h = i + 1  # harmonic number
        h_freq = freq * h
        if h_freq >= sr / 2:
            break

        # Base amplitude: 1/h with some randomness
        base_amp = 1.0 / h * np.random.uniform(0.5, 1.5)

        # LFO for this partial: each partial moves at golden-ratio-related rate
        lfo_rate = base_lfo_rate * (golden ** i % 3.0)
        lfo_phase = np.random.uniform(0, 2 * np.pi)
        amp_envelope = base_amp * (0.5 + 0.5 * np.sin(2*np.pi*lfo_rate*t + lfo_phase))

        # Slight frequency drift for organic quality
        freq_drift = 1.0 + 0.002 * np.sin(2*np.pi*lfo_rate*0.7*t + lfo_phase*1.3)

        output += amp_envelope * np.sin(2 * np.pi * h_freq * freq_drift * t)

    # Smooth amplitude envelope
    att = int(1.5 * sr)
    rel = int(2.0 * sr)
    env = np.ones(n)
    env[:att] = np.linspace(0, 1, att) ** 2
    if rel < n:
        env[-rel:] = np.linspace(1, 0, rel) ** 0.5

    return (output * env) / (np.max(np.abs(output)) + 1e-10)
```

---

## 7. Wavetable Synthesis (Advanced)

### Wavetable Construction from Harmonics

```python
def build_wavetable_set(n_frames=256, table_size=2048):
    """Build a morphable wavetable set with spectral evolution.
    Frame 0 = sine, Frame 128 = saw-like, Frame 255 = complex/noisy.
    """
    wavetables = np.zeros((n_frames, table_size))
    t = np.linspace(0, 2 * np.pi, table_size, endpoint=False)
    max_harmonics = table_size // 2

    for frame in range(n_frames):
        morph = frame / (n_frames - 1)  # 0 to 1

        for h in range(1, min(int(2 + morph * 60), max_harmonics)):
            # Amplitude evolves: starts with few harmonics, adds more
            amp = (1.0 / h) * min(1.0, morph * 4)

            # Phase evolves for timbral movement
            phase_offset = morph * np.pi * h * 0.1

            wavetables[frame] += amp * np.sin(h * t + phase_offset)

        # Normalize each frame
        peak = np.max(np.abs(wavetables[frame]))
        if peak > 0:
            wavetables[frame] /= peak

    return wavetables
```

### Spectral Morphing Between Wavetable Frames

Linear crossfade between frames creates amplitude modulation artifacts. Spectral morphing interpolates in the frequency domain for smoother transitions:

```python
def spectral_morph_frames(frame_a, frame_b, morph):
    """Morph between two wavetable frames using spectral interpolation.
    morph: 0=frame_a, 1=frame_b.
    Interpolates log-magnitude and phase separately.
    """
    spec_a = np.fft.rfft(frame_a)
    spec_b = np.fft.rfft(frame_b)

    # Log-magnitude interpolation (perceptually linear)
    mag_a = np.log(np.abs(spec_a) + 1e-10)
    mag_b = np.log(np.abs(spec_b) + 1e-10)
    mag = np.exp(mag_a * (1 - morph) + mag_b * morph)

    # Phase interpolation (unwrap for smooth transition)
    phase_a = np.angle(spec_a)
    phase_b = np.angle(spec_b)
    # Shortest-path phase interpolation
    phase_diff = phase_b - phase_a
    phase_diff = (phase_diff + np.pi) % (2 * np.pi) - np.pi
    phase = phase_a + morph * phase_diff

    result = np.fft.irfft(mag * np.exp(1j * phase), n=len(frame_a))
    return result
```

### Band-Limited Wavetable Playback (Mip-Mapped)

This extends the MipMappedWavetable from Section 1 for full morphable wavetable synthesis:

```python
class ProductionWavetable:
    """Full wavetable synth with mip-mapping, morphing, and anti-aliasing.
    Modeled after Serum/Vital-style wavetable playback."""

    def __init__(self, wavetables, sr=44100):
        """wavetables: shape (n_frames, table_size)."""
        self.sr = sr
        self.n_frames = len(wavetables)
        self.table_size = len(wavetables[0])

        # Build mip-mapped versions of each frame
        self.mip_levels = int(np.log2(self.table_size))
        self.tables = []  # [level][frame] = band-limited table

        for level in range(self.mip_levels):
            level_tables = []
            max_harmonic = max(1, (self.table_size // 2) >> level)
            for frame in wavetables:
                spectrum = np.fft.rfft(frame)
                spectrum[max_harmonic:] = 0
                level_tables.append(np.fft.irfft(spectrum, n=self.table_size))
            self.tables.append(np.array(level_tables))

    def generate(self, freq, duration, morph_env=None):
        """Generate audio with anti-aliased wavetable playback.
        morph_env: array of frame positions (0 to n_frames-1) over time.
        """
        n = int(self.sr * duration)

        if morph_env is None:
            morph_env = np.zeros(n)

        # Select mip level based on frequency
        max_harmonic = int(self.sr / (2 * freq))
        level = 0
        for l in range(self.mip_levels):
            if (self.table_size // 2) >> l <= max_harmonic:
                level = l
                break

        tables = self.tables[level]

        # Phase accumulator
        phase_inc = freq * self.table_size / self.sr
        output = np.zeros(n)

        phase = 0.0
        for i in range(n):
            # Frame interpolation
            morph = np.clip(morph_env[i], 0, self.n_frames - 1 - 1e-9)
            frame_lo = int(morph)
            frame_hi = min(frame_lo + 1, self.n_frames - 1)
            frame_frac = morph - frame_lo

            # Table interpolation (linear between adjacent samples)
            idx = int(phase) % self.table_size
            idx_next = (idx + 1) % self.table_size
            sample_frac = phase - int(phase)

            # Bilinear interpolation: frame and table position
            s00 = tables[frame_lo][idx]
            s01 = tables[frame_lo][idx_next]
            s10 = tables[frame_hi][idx]
            s11 = tables[frame_hi][idx_next]

            s0 = s00 + sample_frac * (s01 - s00)
            s1 = s10 + sample_frac * (s11 - s10)

            output[i] = s0 + frame_frac * (s1 - s0)

            phase = (phase + phase_inc) % self.table_size

        return output
```

### Building Wavetables from Audio (Serum-Style)

```python
def wavetable_from_audio(audio, n_frames=256, table_size=2048, sr=44100):
    """Extract a wavetable from audio by slicing into single-cycle frames.
    Method 1: Equal spacing (simple).
    Method 2: Pitch-synchronous (better for pitched content).
    """
    # Method 1: Equal spacing
    hop = max(1, (len(audio) - table_size) // n_frames)
    wavetables = np.zeros((n_frames, table_size))

    for i in range(n_frames):
        start = i * hop
        if start + table_size > len(audio):
            start = len(audio) - table_size
        frame = audio[start:start + table_size]
        # Window to force periodicity
        frame *= np.hanning(table_size)
        # Normalize
        peak = np.max(np.abs(frame))
        if peak > 0:
            frame /= peak
        wavetables[i] = frame

    return wavetables

def wavetable_from_harmonics_evolution(harmonic_profiles, table_size=2048):
    """Build wavetable from a list of harmonic amplitude profiles.
    harmonic_profiles: list of lists, each inner list = amplitudes for harmonics 1,2,3...
    Creates smooth interpolation between profiles.
    """
    n_frames = 256
    n_profiles = len(harmonic_profiles)
    wavetables = np.zeros((n_frames, table_size))
    t = np.linspace(0, 2 * np.pi, table_size, endpoint=False)

    for frame in range(n_frames):
        # Interpolate between profiles
        pos = frame / (n_frames - 1) * (n_profiles - 1)
        lo = int(pos)
        hi = min(lo + 1, n_profiles - 1)
        frac = pos - lo

        max_h = max(len(harmonic_profiles[lo]), len(harmonic_profiles[hi]))
        for h in range(max_h):
            amp_lo = harmonic_profiles[lo][h] if h < len(harmonic_profiles[lo]) else 0
            amp_hi = harmonic_profiles[hi][h] if h < len(harmonic_profiles[hi]) else 0
            amp = amp_lo * (1 - frac) + amp_hi * frac

            if (h + 1) * 1.0 < table_size / 2:  # stay below Nyquist for table
                wavetables[frame] += amp * np.sin((h + 1) * t)

        peak = np.max(np.abs(wavetables[frame]))
        if peak > 0:
            wavetables[frame] /= peak

    return wavetables
```

---

## 8. Noise Synthesis and Spectral Shaping

### Colored Noise -- Precise Spectral Slopes

The existing codebase has basic FFT noise. Here are precise implementations with correct spectral slopes:

| Color | Spectral Density | Slope | Perceptual Quality |
|-------|-----------------|-------|--------------------|
| White | Flat | 0 dB/oct | Harsh, hissy |
| Pink | 1/f | -3 dB/oct | Natural, balanced |
| Brown/Red | 1/f^2 | -6 dB/oct | Rumbly, thunder-like |
| Blue | f | +3 dB/oct | Hissy, sparkly |
| Violet | f^2 | +6 dB/oct | Very bright, thin |
| Grey | Equal loudness weighted | Perceptually flat | Balanced to human ear |

### Voss-McCartney Algorithm for Pink Noise

The FFT method works for offline generation, but Voss-McCartney generates pink noise sample-by-sample (useful for real-time or modulation):

```python
def pink_noise_voss(n_samples, n_rows=16):
    """Voss-McCartney pink noise generator.
    n_rows: number of random sources (more = better 1/f approximation).
    Each row updates at half the rate of the previous, creating 1/f spectrum.
    Works sample-by-sample (suitable for real-time / modulation).
    """
    output = np.zeros(n_samples)
    rows = np.zeros(n_rows)
    running_sum = 0.0

    # Initialize
    for j in range(n_rows):
        rows[j] = np.random.uniform(-1, 1)
        running_sum += rows[j]

    max_key = (1 << n_rows) - 1

    for i in range(n_samples):
        # Determine which rows to update (bit change detection)
        key = i
        last_key = max(0, i - 1)
        diff = key ^ last_key  # XOR: bits that changed

        for j in range(n_rows):
            if diff & (1 << j):  # this bit changed
                running_sum -= rows[j]
                rows[j] = np.random.uniform(-1, 1)
                running_sum += rows[j]

        # Add one white noise sample for the highest octave
        white = np.random.uniform(-1, 1)
        output[i] = (running_sum + white) / (n_rows + 1)

    return output
```

### Grey Noise (Perceptually Flat)

Grey noise is shaped to match the inverse of the equal-loudness contour (ISO 226), so it sounds equally loud at all frequencies to the human ear:

```python
def grey_noise(duration, sr=44100):
    """Grey noise: perceptually flat (inverse equal-loudness weighting).
    Sounds 'equally loud' at all frequencies to human hearing.
    """
    n = int(sr * duration)
    white = np.random.randn(n)
    spectrum = np.fft.rfft(white)
    freqs = np.fft.rfftfreq(n, 1.0 / sr)
    freqs[0] = 1  # avoid div by zero

    # Approximate inverse A-weighting curve
    # A-weighting: emphasizes 2-5kHz, rolls off lows and highs
    # Grey noise = inverse: boost lows and highs, cut mids
    f2 = freqs ** 2
    a_weight = (12194**2 * f2**2) / (
        (f2 + 20.6**2) * np.sqrt((f2 + 107.7**2) * (f2 + 737.9**2)) * (f2 + 12194**2)
    )
    # Normalize and invert
    a_weight /= np.max(a_weight)
    grey_weight = 1.0 / (a_weight + 0.01)
    grey_weight /= np.max(grey_weight)

    spectrum *= grey_weight
    sig = np.fft.irfft(spectrum, n=n)
    return sig / (np.max(np.abs(sig)) + 1e-10)
```

### Natural Sound Synthesis via Filtered Noise

```python
def wind_noise(duration, speed=0.5, gusts=True, sr=44100):
    """Wind synthesis. speed: 0=gentle breeze, 0.5=moderate, 1=gale.
    Uses bandpass-filtered noise with LFO-modulated center frequency.
    """
    n = int(sr * duration)
    t = np.arange(n) / sr

    # Base: brown noise (low-frequency emphasis)
    noise = np.cumsum(np.random.randn(n))
    noise -= np.mean(noise)
    noise /= np.max(np.abs(noise)) + 1e-10

    # Wind center frequency varies with speed
    base_freq = 200 + speed * 800  # 200-1000 Hz
    bandwidth = 100 + speed * 500   # 100-600 Hz

    # LFO for natural variation
    lfo = np.sin(2*np.pi*0.15*t) * 0.3 + np.sin(2*np.pi*0.07*t) * 0.2

    # Gust envelope
    if gusts:
        gust_env = np.ones(n)
        n_gusts = int(duration * 0.5)  # ~1 gust every 2 seconds
        for _ in range(n_gusts):
            pos = np.random.randint(0, n)
            length = int(np.random.uniform(0.5, 2.0) * sr)
            end = min(pos + length, n)
            gust = np.hanning(length)[:end - pos]
            gust_env[pos:end] += gust * np.random.uniform(0.5, 2.0)
        noise *= gust_env

    # Apply bandpass with time-varying center
    # Use multiple static filters to approximate
    output = np.zeros(n)
    for fc_mult in [0.7, 1.0, 1.3]:
        fc = base_freq * fc_mult
        low = max(20, fc - bandwidth/2)
        high = min(sr/2 - 100, fc + bandwidth/2)
        sos = butter(3, [low, high], btype='band', fs=sr, output='sos')
        output += sosfilt(sos, noise)

    return output / (np.max(np.abs(output)) + 1e-10)

def rain_noise(duration, intensity=0.5, sr=44100):
    """Rain synthesis. intensity: 0=light drizzle, 0.5=moderate, 1=downpour.
    Combines filtered noise (continuous) with random impulses (drops).
    """
    n = int(sr * duration)

    # Continuous rain wash: shaped pink noise
    wash = np.random.randn(n)
    spectrum = np.fft.rfft(wash)
    freqs = np.fft.rfftfreq(n, 1.0 / sr)
    freqs[0] = 1
    # Rain spectral shape: peak around 2-6kHz, roll off below 500Hz
    rain_shape = np.exp(-((np.log2(freqs / 3000)) ** 2) / 2)  # Gaussian in log-freq
    rain_shape *= 1.0 / np.sqrt(freqs)  # pink-ish base
    spectrum *= rain_shape
    wash = np.fft.irfft(spectrum, n=n)

    # Individual drops: sparse impulses with short decay
    drops = np.zeros(n)
    drops_per_sec = int(20 + intensity * 200)
    for _ in range(int(duration * drops_per_sec)):
        pos = np.random.randint(0, n)
        # Each drop: short burst (1-5ms)
        drop_len = int(np.random.uniform(0.001, 0.005) * sr)
        if pos + drop_len < n:
            drop = np.random.randn(drop_len) * np.exp(-np.arange(drop_len) / (drop_len * 0.2))
            # Bandpass each drop around 2-8kHz
            drop_amp = np.random.uniform(0.01, 0.05) * intensity
            drops[pos:pos + drop_len] += drop * drop_amp

    # Highpass the drops
    sos_hp = butter(2, 1000, btype='high', fs=sr, output='sos')
    drops = sosfilt(sos_hp, drops)

    output = wash * (0.3 + intensity * 0.7) + drops
    return output / (np.max(np.abs(output)) + 1e-10)

def ocean_waves(duration, wave_period=8.0, sr=44100):
    """Ocean wave synthesis. Cyclic filtered noise with surge envelope.
    wave_period: seconds between wave crests (6-12s typical).
    """
    n = int(sr * duration)
    t = np.arange(n) / sr

    # Base noise: mix of brown (rumble) and pink (wash)
    brown = np.cumsum(np.random.randn(n))
    brown /= np.max(np.abs(brown)) + 1e-10

    white = np.random.randn(n)
    spectrum = np.fft.rfft(white)
    freqs = np.fft.rfftfreq(n, 1.0 / sr)
    freqs[0] = 1
    spectrum *= 1.0 / np.sqrt(freqs)
    pink = np.fft.irfft(spectrum, n=n)
    pink /= np.max(np.abs(pink)) + 1e-10

    # Wave envelope: asymmetric (fast rise, slow fade)
    wave_env = np.zeros(n)
    phase = (t / wave_period) % 1.0
    # Fast rise at 0.0, peak at 0.15, slow decay to 1.0
    wave_env = np.where(phase < 0.15,
                        (phase / 0.15) ** 0.5,
                        np.exp(-(phase - 0.15) * 2))
    # Add randomness
    wave_env *= 0.5 + 0.5 * np.sin(2*np.pi*0.03*t)

    # Combine: low rumble + high wash, modulated by wave envelope
    low_component = brown * wave_env
    sos_lp = butter(3, 300, btype='low', fs=sr, output='sos')
    low_component = sosfilt(sos_lp, low_component)

    high_component = pink * wave_env
    sos_bp = butter(2, [200, 4000], btype='band', fs=sr, output='sos')
    high_component = sosfilt(sos_bp, high_component)

    output = low_component * 0.4 + high_component * 0.6
    return output / (np.max(np.abs(output)) + 1e-10)
```

---

## 9. Convolution Reverb Implementation

### FFT-Based Convolution: Overlap-Save vs Overlap-Add

The existing codebase uses `fftconvolve` which is overlap-add internally. For low-latency applications, partitioned convolution is essential:

```python
def partitioned_convolution(sig, ir, block_size=1024):
    """Partitioned convolution for low-latency reverb.
    Splits the IR into blocks and processes incrementally.
    Latency = block_size samples (vs full IR length for naive FFT).

    This is how professional convolution reverbs work (e.g., Altiverb, Space Designer).
    """
    n_sig = len(sig)
    n_ir = len(ir)
    fft_size = block_size * 2  # double for linear convolution via circular

    # Partition IR into blocks and pre-compute FFTs
    n_partitions = (n_ir + block_size - 1) // block_size
    ir_ffts = []
    for p in range(n_partitions):
        start = p * block_size
        end = min(start + block_size, n_ir)
        ir_block = np.zeros(fft_size)
        ir_block[:end - start] = ir[start:end]
        ir_ffts.append(np.fft.rfft(ir_block))

    # Process signal in blocks
    output = np.zeros(n_sig + n_ir - 1)
    # FDL (Frequency-Domain Delay Line)
    fdl = [np.zeros(fft_size // 2 + 1, dtype=complex) for _ in range(n_partitions)]

    for block_idx in range(0, n_sig, block_size):
        # Get input block
        end = min(block_idx + block_size, n_sig)
        sig_block = np.zeros(fft_size)
        sig_block[:end - block_idx] = sig[block_idx:end]
        sig_fft = np.fft.rfft(sig_block)

        # Shift FDL and insert new block
        fdl.pop()
        fdl.insert(0, sig_fft)

        # Multiply-accumulate across all partitions
        acc = np.zeros(fft_size // 2 + 1, dtype=complex)
        for p in range(min(n_partitions, len(fdl))):
            acc += fdl[p] * ir_ffts[p]

        # Inverse FFT and overlap-add
        result = np.fft.irfft(acc, n=fft_size)
        out_start = block_idx
        out_end = min(out_start + fft_size, len(output))
        output[out_start:out_end] += result[:out_end - out_start]

    return output[:n_sig]
```

### Synthetic Impulse Response Design

Create realistic IRs for specific room types without recording actual spaces:

```python
def synthetic_ir(room_type='hall', duration=None, sr=44100):
    """Generate synthetic impulse responses for various room types.

    Room acoustics parameters:
    - Pre-delay: time before first reflection (room size indicator)
    - Early reflections: first 50-80ms, define room shape/size
    - Late reverb: diffuse exponential decay
    - RT60: time for reverb to decay 60dB
    - Damping: HF absorption (soft surfaces = more damping)
    """
    configs = {
        'small_room': {
            'predelay_ms': 5, 'er_density': 20, 'er_duration_ms': 40,
            'rt60': 0.4, 'damping_freq': 6000, 'diffusion': 0.7,
            'er_level': 0.8, 'late_level': 0.3
        },
        'medium_room': {
            'predelay_ms': 12, 'er_density': 25, 'er_duration_ms': 60,
            'rt60': 0.8, 'damping_freq': 5000, 'diffusion': 0.8,
            'er_level': 0.6, 'late_level': 0.5
        },
        'hall': {
            'predelay_ms': 25, 'er_density': 15, 'er_duration_ms': 80,
            'rt60': 2.0, 'damping_freq': 4000, 'diffusion': 0.9,
            'er_level': 0.4, 'late_level': 0.7
        },
        'cathedral': {
            'predelay_ms': 40, 'er_density': 10, 'er_duration_ms': 120,
            'rt60': 5.0, 'damping_freq': 2500, 'diffusion': 0.95,
            'er_level': 0.3, 'late_level': 0.8
        },
        'plate': {
            'predelay_ms': 2, 'er_density': 40, 'er_duration_ms': 30,
            'rt60': 1.5, 'damping_freq': 8000, 'diffusion': 0.95,
            'er_level': 0.5, 'late_level': 0.6
        },
        'spring': {
            'predelay_ms': 10, 'er_density': 8, 'er_duration_ms': 50,
            'rt60': 1.0, 'damping_freq': 3000, 'diffusion': 0.5,
            'er_level': 0.7, 'late_level': 0.4
        },
        'bathroom': {
            'predelay_ms': 3, 'er_density': 30, 'er_duration_ms': 25,
            'rt60': 0.6, 'damping_freq': 7000, 'diffusion': 0.85,
            'er_level': 0.9, 'late_level': 0.4
        },
        'parking_garage': {
            'predelay_ms': 15, 'er_density': 18, 'er_duration_ms': 70,
            'rt60': 3.0, 'damping_freq': 3500, 'diffusion': 0.75,
            'er_level': 0.5, 'late_level': 0.6
        }
    }

    cfg = configs.get(room_type, configs['hall'])

    if duration is None:
        duration = cfg['rt60'] * 1.5

    n = int(sr * duration)
    ir = np.zeros(n)

    # 1. Initial impulse
    ir[0] = 1.0

    # 2. Pre-delay
    pd_samples = int(cfg['predelay_ms'] * sr / 1000)

    # 3. Early reflections: discrete echoes from walls/ceiling/floor
    er_end = pd_samples + int(cfg['er_duration_ms'] * sr / 1000)
    for _ in range(cfg['er_density']):
        pos = pd_samples + np.random.randint(1, er_end - pd_samples)
        if pos < n:
            delay_ms = (pos - pd_samples) * 1000 / sr
            amp = cfg['er_level'] * np.exp(-delay_ms / (cfg['er_duration_ms'] * 0.5))
            amp *= np.random.uniform(0.5, 1.0)
            ir[pos] += amp * np.random.choice([-1, 1])

    # 4. Late reverb: exponential noise decay
    late_start = er_end
    if late_start < n:
        late_len = n - late_start
        decay_rate = 6.91 / cfg['rt60']  # -60dB in RT60 seconds
        t_late = np.arange(late_len) / sr
        late = np.random.randn(late_len) * np.exp(-decay_rate * t_late)
        late *= cfg['late_level']

        # Frequency-dependent damping
        sos = butter(2, min(cfg['damping_freq'], sr/2 - 100), btype='low', fs=sr, output='sos')
        late = sosfilt(sos, late)

        # Diffusion
        if cfg['diffusion'] > 0.5:
            diff_len = int(0.01 * sr * cfg['diffusion'])
            diff_kernel = np.random.randn(diff_len)
            diff_kernel /= np.sqrt(np.sum(diff_kernel**2))
            from scipy.signal import fftconvolve
            late = fftconvolve(late, diff_kernel, mode='same')

        ir[late_start:] += late

    ir /= np.max(np.abs(ir)) + 1e-10
    return ir
```

### IR Characteristics by Room Type

| Room | RT60 | Pre-delay | Early Ref Pattern | HF Damping | Character |
|------|------|-----------|-------------------|------------|-----------|
| Vocal booth | 0.1-0.3s | 1-3ms | Dense, fast | 8-10kHz | Dead, intimate |
| Bedroom | 0.3-0.5s | 5-8ms | Sparse | 6-8kHz | Small, warm |
| Live room | 0.5-1.0s | 8-15ms | Medium | 5-7kHz | Natural, musical |
| Concert hall | 1.5-2.5s | 20-40ms | Sparse early, dense late | 3-5kHz | Grand, spacious |
| Cathedral | 4-8s | 30-60ms | Very sparse | 2-4kHz | Huge, ethereal |
| Plate reverb | 1-3s | 0-5ms | Very dense | 8-12kHz | Bright, smooth |
| Spring reverb | 0.5-1.5s | 5-15ms | Sparse, boingy | 3-5kHz | Vintage, twangy |

---

## 10. Phase Vocoder and Time-Stretching

### Phase Locking for Improved Quality

The basic phase vocoder suffers from "phasiness" (smeared transients, metallic quality). Phase locking groups bins to the nearest spectral peak and locks their phase relationships:

```python
def phase_vocoder_locked(sig, stretch_factor, fft_size=2048, hop_size=512, sr=44100):
    """Phase vocoder with phase locking for improved quality.
    Identifies spectral peaks and locks surrounding bins to peak phases.
    Significantly reduces phasiness artifacts.
    """
    window = np.hanning(fft_size)
    n_frames = 1 + (len(sig) - fft_size) // hop_size
    hop_out = int(hop_size * stretch_factor)
    n_bins = fft_size // 2 + 1
    omega = 2 * np.pi * np.arange(n_bins) * hop_size / fft_size

    out_len = int(len(sig) * stretch_factor) + fft_size
    output = np.zeros(out_len)
    win_sum = np.zeros(out_len)
    prev_phase = np.zeros(n_bins)
    cum_phase = np.zeros(n_bins)

    for i in range(n_frames):
        start = i * hop_size
        frame = sig[start:start + fft_size]
        if len(frame) < fft_size:
            frame = np.pad(frame, (0, fft_size - len(frame)))

        spec = np.fft.rfft(frame * window)
        mag = np.abs(spec)
        phase = np.angle(spec)

        # Phase advancement
        phase_diff = phase - prev_phase - omega
        phase_diff -= 2 * np.pi * np.round(phase_diff / (2 * np.pi))
        true_freq = omega + phase_diff

        # Phase locking: find peaks and lock bins
        peaks = np.zeros(n_bins, dtype=bool)
        for b in range(1, n_bins - 1):
            if mag[b] > mag[b-1] and mag[b] > mag[b+1]:
                peaks[b] = True

        locked_phase = cum_phase + true_freq * stretch_factor
        peak_indices = np.where(peaks)[0]

        if len(peak_indices) > 0:
            for b in range(n_bins):
                nearest = peak_indices[np.argmin(np.abs(peak_indices - b))]
                phase_diff_to_peak = phase[b] - phase[nearest]
                locked_phase[b] = locked_phase[nearest] + phase_diff_to_peak

        cum_phase = locked_phase

        syn_frame = np.fft.irfft(mag * np.exp(1j * cum_phase), n=fft_size) * window
        out_start = i * hop_out
        if out_start + fft_size <= out_len:
            output[out_start:out_start + fft_size] += syn_frame
            win_sum[out_start:out_start + fft_size] += window ** 2

        prev_phase = phase

    return (output / np.maximum(win_sum, 1e-10))[:int(len(sig) * stretch_factor)]
```

### Transient Preservation

Transients (drums, plucks) smear badly in phase vocoders. Detect them and handle separately:

```python
def phase_vocoder_transient_preserving(sig, stretch_factor, fft_size=2048,
                                        hop_size=512, sr=44100):
    """Phase vocoder with transient detection and preservation.
    Transient frames reset phase accumulator to prevent smearing.
    """
    window = np.hanning(fft_size)
    n_frames = 1 + (len(sig) - fft_size) // hop_size
    n_bins = fft_size // 2 + 1
    omega = 2 * np.pi * np.arange(n_bins) * hop_size / fft_size
    hop_out = int(hop_size * stretch_factor)

    # Transient detection via spectral flux
    prev_mag = np.zeros(n_bins)
    flux = np.zeros(n_frames)
    all_specs = []

    for i in range(n_frames):
        start = i * hop_size
        frame = sig[start:start + fft_size]
        if len(frame) < fft_size:
            frame = np.pad(frame, (0, fft_size - len(frame)))
        spec = np.fft.rfft(frame * window)
        all_specs.append(spec)
        mag = np.abs(spec)
        diff = mag - prev_mag
        flux[i] = np.sum(np.maximum(diff, 0))
        prev_mag = mag.copy()

    # Adaptive threshold
    median_flux = np.median(flux)
    is_transient = flux > median_flux * 3.0

    # Synthesize
    out_len = int(len(sig) * stretch_factor) + fft_size
    output = np.zeros(out_len)
    win_sum = np.zeros(out_len)
    prev_phase = np.zeros(n_bins)
    cum_phase = np.zeros(n_bins)

    for i in range(n_frames):
        spec = all_specs[i]
        mag = np.abs(spec)
        phase = np.angle(spec)

        if is_transient[i]:
            cum_phase = phase.copy()
        else:
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
```

### Enhanced Paulstretch

```python
def paulstretch_enhanced(sig, stretch_factor=8.0, window_size_s=0.25,
                          harmonics_smoothing=0.0, freq_shift=0.0, sr=44100):
    """Enhanced Paulstretch with spectral manipulation.
    harmonics_smoothing: 0=none, 1=full (removes pitched content for pure texture).
    freq_shift: shift spectral content up/down in Hz.
    """
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

        # Spectral smoothing
        if harmonics_smoothing > 0:
            smooth_width = int(1 + harmonics_smoothing * 20)
            kernel = np.ones(smooth_width) / smooth_width
            smoothed = np.convolve(mag, kernel, mode='same')
            mag = mag * (1 - harmonics_smoothing) + smoothed * harmonics_smoothing

        # Frequency shift
        if freq_shift != 0:
            shift_bins = int(freq_shift * window_samples / sr)
            if shift_bins > 0:
                mag = np.concatenate([np.zeros(shift_bins), mag[:-shift_bins]])
            elif shift_bins < 0:
                mag = np.concatenate([mag[-shift_bins:], np.zeros(-shift_bins)])

        random_phase = np.exp(1j * np.random.uniform(0, 2*np.pi, len(mag)))
        syn_frame = np.fft.irfft(mag * random_phase, n=window_samples) * window

        out_start = i * hop_out
        output[out_start:out_start + window_samples] += syn_frame
        win_sum[out_start:out_start + window_samples] += window ** 2

    return output / np.maximum(win_sum, 1e-10)
```

### High-Quality Time Stretching (Rubber Band Style)

Professional stretching combines transient detection, phase vocoder, and granular techniques:

```python
def high_quality_stretch(sig, stretch_factor, sr=44100):
    """High-quality time stretching combining phase vocoder with transient handling.
    Splits signal at detected transients, applies appropriate method to each region.
    """
    fft_size = 2048
    hop = 512
    n_frames = (len(sig) - fft_size) // hop + 1

    # Detect transients via spectral flux
    prev_mag = np.zeros(fft_size // 2 + 1)
    onsets = []
    for i in range(n_frames):
        start = i * hop
        frame = sig[start:start + fft_size] * np.hanning(fft_size)
        mag = np.abs(np.fft.rfft(frame))
        flux_val = np.sum(np.maximum(mag - prev_mag, 0))
        if flux_val > np.mean(prev_mag) * 5 and i > 0:
            onsets.append(i * hop)
        prev_mag = mag

    if len(onsets) == 0:
        return phase_vocoder_locked(sig, stretch_factor, fft_size, hop, sr)

    # Split and process segments
    boundaries = [0] + onsets + [len(sig)]
    output_parts = []

    for s in range(len(boundaries) - 1):
        segment = sig[boundaries[s]:boundaries[s + 1]]
        if len(segment) < fft_size:
            stretched_len = int(len(segment) * stretch_factor)
            indices = np.linspace(0, len(segment) - 1, max(1, stretched_len))
            idx0 = np.clip(indices.astype(int), 0, len(segment) - 2)
            frac = indices - idx0
            stretched = segment[idx0] * (1 - frac) + segment[idx0 + 1] * frac
        else:
            stretched = phase_vocoder_locked(segment, stretch_factor, fft_size, hop, sr)
        output_parts.append(stretched)

    # Concatenate with crossfades
    xfade = int(0.005 * sr)
    output = output_parts[0]
    for part in output_parts[1:]:
        if len(output) > xfade and len(part) > xfade:
            output[-xfade:] *= np.linspace(1, 0, xfade)
            part[:xfade] *= np.linspace(0, 1, xfade)
            output[-xfade:] += part[:xfade]
            output = np.concatenate([output, part[xfade:]])
        else:
            output = np.concatenate([output, part])

    return output
```

---

## 11. Nonlinear Dynamics in Synthesis

### Chaos Oscillators

#### Lorenz Attractor

```python
def lorenz_oscillator(duration, freq_scale=1.0, sigma=10.0, rho=28.0,
                       beta=8/3, sr=44100):
    """Lorenz chaotic oscillator. Outputs 3 signals (x, y, z).
    Default params produce the classic butterfly attractor.
    freq_scale: speed of evolution (higher = higher pitch chaos).
    """
    n = int(sr * duration)
    dt = freq_scale / sr
    x, y, z = 0.1, 0.0, 0.0
    out_x = np.zeros(n)
    out_y = np.zeros(n)

    for i in range(n):
        dx = sigma * (y - x) * dt
        dy = (x * (rho - z) - y) * dt
        dz = (x * y - beta * z) * dt
        x += dx; y += dy; z += dz
        out_x[i] = x; out_y[i] = y

    out_x /= np.max(np.abs(out_x)) + 1e-10
    return out_x

def lorenz_synth(freq, duration, chaos=0.5, sr=44100):
    """Musical Lorenz. chaos: 0=periodic (rho~20), 0.5=edge, 1=full chaos (rho~40)."""
    rho = 20 + chaos * 20
    return lorenz_oscillator(duration, freq * 0.01, rho=rho, sr=sr)
```

#### Chua's Circuit

```python
def chua_oscillator(duration, alpha=15.6, beta=28.0, m0=-1.143, m1=-0.714,
                     freq_scale=1.0, sr=44100):
    """Chua's circuit: simplest electronic circuit exhibiting chaos.
    Produces double-scroll attractor with default parameters.
    """
    n = int(sr * duration)
    dt = freq_scale / sr
    x, y, z = 0.1, 0.0, 0.0
    output = np.zeros(n)

    for i in range(n):
        h = m1 * x + 0.5 * (m0 - m1) * (abs(x + 1) - abs(x - 1))
        dx = alpha * (y - x - h) * dt
        dy = (x - y + z) * dt
        dz = -beta * y * dt
        x += dx; y += dy; z += dz
        output[i] = x

    return output / (np.max(np.abs(output)) + 1e-10)
```

### Feedback Delay Networks for Living Textures

```python
def feedback_network(duration, n_delays=4, base_freq=200,
                      feedback=0.7, saturation=0.8, sr=44100):
    """Cross-coupled feedback delay network. Creates self-sustaining evolving timbres.
    n_delays: 3-8. Uses mutually prime delay lengths for maximum diffusion.
    """
    n = int(sr * duration)
    primes = [131, 137, 139, 149, 151, 157, 163, 167]
    delays = [int(sr / base_freq * primes[i % len(primes)] / 131) for i in range(n_delays)]

    buffers = [np.zeros(d + 1) for d in delays]
    write_ptrs = [0] * n_delays
    output = np.zeros(n)

    # Feedback matrix (energy-preserving with sign alternation)
    fb = feedback / np.sqrt(n_delays)
    matrix = np.zeros((n_delays, n_delays))
    for j in range(n_delays):
        for k in range(n_delays):
            if j != k:
                matrix[j][k] = fb * (1 if (j + k) % 2 == 0 else -1)

    seed_len = min(delays[0], int(sr * 0.005))

    for i in range(n):
        reads = np.zeros(n_delays)
        for d in range(n_delays):
            read_ptr = (write_ptrs[d] - delays[d]) % (delays[d] + 1)
            reads[d] = buffers[d][read_ptr]

        writes = matrix @ reads
        if i < seed_len:
            exc = np.random.randn() * np.exp(-i / (seed_len * 0.3))
            for d in range(n_delays):
                writes[d] += exc / n_delays

        for d in range(n_delays):
            writes[d] = np.tanh(writes[d] * saturation)
            buffers[d][write_ptrs[d]] = writes[d]
            write_ptrs[d] = (write_ptrs[d] + 1) % (delays[d] + 1)

        output[i] = np.sum(reads) / n_delays

    return output / (np.max(np.abs(output)) + 1e-10)
```

### Self-Oscillating Filter

```python
def self_oscillating_filter(duration, freq=440, sweep_rate=0.1,
                             sweep_depth=2.0, sr=44100):
    """SVF at extreme resonance, driven only by its own feedback.
    Creates singing tones from nothing but noise seed + feedback.
    """
    n = int(sr * duration)
    t = np.arange(n) / sr
    output = np.zeros(n)
    low = band = 0.0
    q_inv = 0.001  # Q ~1000

    fc = freq * (2 ** (sweep_depth * np.sin(2*np.pi*sweep_rate*t)))
    seed = np.random.randn(n) * 0.0001

    for i in range(n):
        f = 2 * np.sin(np.pi * min(fc[i], sr * 0.45) / sr)
        high = seed[i] - low - q_inv * band
        band += f * high
        low += f * band
        band = np.tanh(band)
        output[i] = band

    return output / (np.max(np.abs(output)) + 1e-10)
```

---

## 12. Supersaw / Unison Optimization

### The Roland JP-8000 Supersaw

7 detuned saws with specific spread and mix curves (reverse-engineered by Adam Szabo, 2010):

```python
def supersaw(freq, duration, detune=0.5, mix=0.5, n_voices=7, sr=44100):
    """JP-8000 style supersaw.
    detune: 0-1 (max spread ~100 cents at 1.0).
    mix: 0=center only, 1=all voices equal.
    """
    n = int(sr * duration)
    max_spread_cents = 100 * detune
    offsets = np.linspace(-max_spread_cents, max_spread_cents, n_voices)
    center = n_voices // 2

    amps = np.ones(n_voices)
    for v in range(n_voices):
        if v == center:
            amps[v] = 1.0 - mix * 0.5
        else:
            distance = abs(v - center) / max(center, 1)
            amps[v] = mix * (1.0 - distance * 0.3)

    output = np.zeros(n)

    for v in range(n_voices):
        ratio = 2 ** (offsets[v] / 1200.0)
        dt = freq * ratio / sr
        phase = np.cumsum(np.full(n, dt)) % 1.0
        phase = (phase + np.random.uniform(0, 1)) % 1.0

        # PolyBLEP saw
        saw = 2.0 * phase - 1.0
        blep = np.zeros(n)
        mask1 = phase < dt
        x1 = phase[mask1] / dt
        blep[mask1] = x1 + x1 - x1 * x1 - 1.0
        mask2 = phase > (1.0 - dt)
        x2 = (phase[mask2] - 1.0) / dt
        blep[mask2] = x2 * x2 + x2 + x2 + 1.0
        saw -= blep

        output += saw * amps[v]

    # DC block
    prev_in, prev_out = 0.0, 0.0
    for i in range(n):
        new_val = output[i] - prev_in + 0.995 * prev_out
        prev_in = output[i]; prev_out = new_val
        output[i] = new_val

    return output / (np.max(np.abs(output)) + 1e-10)
```

### Optimal Detune Curves

| Shape | Formula (cents per voice v) | Character |
|-------|---------------------------|-----------|
| Linear | `max_cents * v / (n/2)` | Even spread, classic |
| Exponential | `max_cents * (2^(v/(n/2)) - 1)` | Clustered center, warm |
| Parabolic | `max_cents * (v/(n/2))^2` | Wide outer, tight core |
| Random | `max_cents * random()` | Analog-like |

```python
def unison_detune(n_voices, max_cents, shape='linear'):
    """Generate detune offsets centered on 0."""
    if n_voices == 1:
        return np.array([0.0])
    if shape == 'linear':
        return np.linspace(-max_cents, max_cents, n_voices)
    elif shape == 'exponential':
        t = np.linspace(-1, 1, n_voices)
        return max_cents * np.sign(t) * (2**np.abs(t) - 1)
    elif shape == 'parabolic':
        t = np.linspace(-1, 1, n_voices)
        return max_cents * np.sign(t) * t**2
    elif shape == 'random':
        offsets = np.sort(np.random.uniform(-max_cents, max_cents, n_voices))
        return offsets - np.mean(offsets)
```

### Voice Count Guide

| Voices | Character | Best For |
|--------|-----------|----------|
| 2 | Subtle thickness | Chorus-like, mono leads |
| 3 | Noticeable width | Bass thickening |
| 5 | Full unison | Standard leads and pads |
| 7 | Classic supersaw | Trance, EDM, anthemic |
| 9-16 | Massive, washy | Huge pads, ambient walls |

Perceptual difference between 7 and 9 is small. 5-7 voices is the production sweet spot.

### Stereo Supersaw

```python
def supersaw_stereo(freq, duration, detune=0.5, n_voices=7,
                     stereo_width=0.8, sr=44100):
    """Stereo supersaw with per-voice panning. Outer voices panned wider."""
    n = int(sr * duration)
    left = np.zeros(n)
    right = np.zeros(n)
    offsets = np.linspace(-100 * detune, 100 * detune, n_voices)
    center = n_voices // 2

    for v in range(n_voices):
        dt = freq * 2 ** (offsets[v] / 1200.0) / sr
        phase = (np.cumsum(np.full(n, dt)) + np.random.uniform(0, 1)) % 1.0

        saw = 2.0 * phase - 1.0
        blep = np.zeros(n)
        m1 = phase < dt
        blep[m1] = (phase[m1]/dt)*2 - (phase[m1]/dt)**2 - 1.0
        m2 = phase > (1.0 - dt)
        x2 = (phase[m2] - 1.0) / dt
        blep[m2] = x2**2 + x2*2 + 1.0
        saw -= blep

        # Pan: center voice centered, outer voices spread
        pan = 0.5 + (v - center) / max(center, 1) * 0.5 * stereo_width
        pan = np.clip(pan, 0, 1)
        amp = 1.0 / np.sqrt(n_voices)
        left += saw * amp * np.cos(pan * np.pi * 0.5)
        right += saw * amp * np.sin(pan * np.pi * 0.5)

    # DC block both
    for ch in [left, right]:
        pi, po = 0.0, 0.0
        for i in range(n):
            nv = ch[i] - pi + 0.995 * po
            pi = ch[i]; po = nv; ch[i] = nv

    scale = max(np.max(np.abs(left)), np.max(np.abs(right))) + 1e-10
    return left / scale, right / scale
```

### Vectorized Supersaw (Fast Offline)

```python
def supersaw_fast(freq, duration, detune=0.5, n_voices=7, sr=44100):
    """~5x faster: all voices computed as matrix operations."""
    n = int(sr * duration)
    offsets = np.linspace(-100 * detune, 100 * detune, n_voices)
    dts = (freq * 2 ** (offsets / 1200.0) / sr).reshape(-1, 1)

    phases = np.cumsum(np.broadcast_to(dts, (n_voices, n)), axis=1) % 1.0
    phases = (phases + np.random.uniform(0, 1, (n_voices, 1))) % 1.0

    saws = 2.0 * phases - 1.0
    for v in range(n_voices):
        dt = dts[v, 0]
        p = phases[v]
        blep = np.zeros(n)
        m1 = p < dt
        blep[m1] = (p[m1]/dt)*2 - (p[m1]/dt)**2 - 1.0
        m2 = p > (1.0 - dt)
        x2 = (p[m2] - 1.0) / dt
        blep[m2] = x2**2 + x2*2 + 1.0
        saws[v] -= blep

    output = np.sum(saws, axis=0)

    # DC block
    pi, po = 0.0, 0.0
    for i in range(n):
        nv = output[i] - pi + 0.995 * po
        pi = output[i]; po = nv; output[i] = nv

    return output / (np.max(np.abs(output)) + 1e-10)
```

---

## Quick Reference: Which Technique for Which Sound?

| Sound Goal | Primary Technique | Secondary | Key Parameters |
|-----------|-------------------|-----------|----------------|
| Fat bass | FM (1:1) or Supersaw (3-5v) | Moog ladder | Short mod index decay |
| Trance lead | Supersaw (7v) | ZDF filter sweep | 30-60 cents detune |
| Ambient pad | Additive (evolving) or Granular | Reverb, slow LFO | Golden ratio LFOs |
| Bell/Chime | FM (1:3.5) or Modal | Long reverb | High mod index, fast decay |
| Plucked string | Karplus-Strong | Commuted body | Allpass frac delay |
| Drum | Membrane waveguide or KS drum | Transient shaping | Mode ratios |
| E. Piano | FM (1:14 + 1:1) | Gentle chorus | Velocity-sens mod index |
| Brass | FM (1:1) | Resonant filter | Rising index envelope |
| Wind/Ocean | Filtered noise | Modulated bandpass | Brown/pink base |
| Evolving texture | Feedback network or Chaos | Granular | Lorenz/Chua + reverb |
| Vocal pad | Formant + Additive | Spectral morph | Vowel interpolation |
| Time-stretch | Phase vocoder (locked) | Granular for transients | FFT 2048-4096 |
