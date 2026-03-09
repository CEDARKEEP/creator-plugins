# Post-Generation Quality Validation

Audio analysis pipeline for validating generated .wav files. LUFS metering, clipping detection, frequency balance, stereo analysis. Shared across all audio generation skills.

## Quick Validation

Top-level entry point. Reads the wav, runs all checks, returns a unified results dict.

```python
import numpy as np
from scipy.io import wavfile
from scipy.signal import sosfilt, butter, welch

def validate_wav(filename, energy_map=None, genre='generic', sr=44100):
    """Full validation pipeline for a generated wav file.

    Args:
        filename: Path to .wav file
        energy_map: Optional dict with section info for energy comparison
        genre: Genre key for target matching (default: 'generic')
        sr: Expected sample rate

    Returns:
        dict with pass/fail for each check and detailed metrics
    """
    file_sr, raw = wavfile.read(filename)
    assert file_sr == sr, f"Sample rate mismatch: expected {sr}, got {file_sr}"

    # Normalize to float64 [-1, 1]
    if raw.dtype == np.int16:
        data = raw.astype(np.float64) / 32768.0
    elif raw.dtype == np.int32:
        data = raw.astype(np.float64) / 2147483648.0
    elif raw.dtype == np.float32 or raw.dtype == np.float64:
        data = raw.astype(np.float64)
    else:
        data = raw.astype(np.float64) / np.max(np.abs(raw))

    # Ensure stereo (N, 2)
    if data.ndim == 1:
        data = np.stack([data, data], axis=-1)

    results = {}
    targets = GENRE_TARGETS.get(genre, GENRE_TARGETS['generic'])

    # Peak & LUFS
    results['peak'] = peak_analysis(data)
    results['lufs'] = measure_lufs_integrated(data, sr)
    results['lufs_pass'] = abs(results['lufs'] - targets['lufs_target']) <= targets['lufs_tolerance']
    results['peak_pass'] = results['peak']['peak_db'] <= targets['max_peak_dbtp']

    # Clipping
    results['clipping'] = detect_clipping(data)
    results['clipping_pass'] = results['clipping']['severity'] == 'none'

    # Frequency balance
    results['freq_balance'] = frequency_balance(data, sr)
    spectral_key = targets.get('spectral_profile_key', genre)
    if spectral_key in GENRE_SPECTRAL_TARGETS:
        ref = GENRE_SPECTRAL_TARGETS[spectral_key]
        max_dev = max(abs(results['freq_balance'][b] - ref[b]) for b in ref if b in results['freq_balance'])
        results['freq_balance_pass'] = max_dev < 8.0  # 8 dB tolerance
    else:
        results['freq_balance_pass'] = True

    # Stereo
    results['stereo'] = stereo_analysis(data)
    results['stereo_pass'] = not results['stereo']['phase_issues']

    # Crest factor
    cf = results['peak']['crest_factor']
    cf_lo, cf_hi = targets['crest_factor_range']
    results['crest_pass'] = cf_lo <= cf <= cf_hi

    # Energy map comparison
    if energy_map is not None:
        results['energy_map'] = compare_to_energy_map(data, energy_map, sr)
        results['energy_map_pass'] = len(results['energy_map']['flagged_sections']) == 0

    # Overall
    pass_keys = [k for k in results if k.endswith('_pass')]
    results['overall_pass'] = all(results[k] for k in pass_keys)

    print_validation_report(results)
    return results
```

## Peak & LUFS Analysis

Loudness measurement per ITU-R BS.1770-4 using K-weighting and gating, implemented with scipy SOS filters.

```python
def peak_analysis(data):
    """Analyze peak and RMS levels.

    Args:
        data: numpy array, shape (N,) or (N, 2), float64 in [-1, 1]

    Returns:
        dict with peak_db, rms_db, crest_factor
    """
    peak = np.max(np.abs(data))
    peak_db = 20.0 * np.log10(peak + 1e-12)
    rms = np.sqrt(np.mean(data ** 2))
    rms_db = 20.0 * np.log10(rms + 1e-12)
    crest_factor = peak_db - rms_db
    return {
        'peak': float(peak),
        'peak_db': float(peak_db),
        'rms_db': float(rms_db),
        'crest_factor': float(crest_factor),
    }


def k_weight_filter(sr):
    """K-weighting filter coefficients per ITU-R BS.1770-4.

    Two cascaded stages:
      1. Pre-filter: high-shelf boosting +4 dB at 1681 Hz (models head diffraction)
      2. RLB high-pass: 2nd-order Butterworth at 38 Hz (revised low-frequency)

    Returns:
        sos: numpy array of SOS sections (concatenated), suitable for sosfilt()
    """
    # Stage 1 — high-shelf filter (+4 dB above ~1681 Hz)
    # Designed as a biquad; coefficients from BS.1770-4 reference at 48 kHz,
    # recalculated via bilinear transform for arbitrary sr
    f0 = 1681.974450955533
    G = 3.999843853973347   # dB gain
    Q = 0.7071752369554196

    K = np.tan(np.pi * f0 / sr)
    Vh = 10.0 ** (G / 20.0)
    Vb = Vh ** 0.4996667741545416

    a0 = 1.0 + K / Q + K * K
    b0 = (Vh + Vb * K / Q + K * K) / a0
    b1 = 2.0 * (K * K - Vh) / a0
    b2 = (Vh - Vb * K / Q + K * K) / a0
    a1 = 2.0 * (K * K - 1.0) / a0
    a2 = (1.0 - K / Q + K * K) / a0

    shelf_sos = np.array([[b0, b1, b2, 1.0, a1, a2]])

    # Stage 2 — RLB high-pass (2nd-order Butterworth at 38 Hz)
    rlb_sos = butter(2, 38.0, btype='high', fs=sr, output='sos')

    # Concatenate both stages into one SOS array
    sos = np.vstack([shelf_sos, rlb_sos])
    return sos


def measure_lufs_integrated(data, sr):
    """Integrated loudness in LUFS per ITU-R BS.1770-4.

    Steps:
      1. K-weight each channel
      2. Measure mean-square per 400 ms block (75% overlap)
      3. Absolute gate at -70 LUFS
      4. Relative gate at -10 dB below absolute-gated loudness
      5. Return integrated loudness

    Args:
        data: (N,) mono or (N, 2) stereo, float64
        sr: sample rate

    Returns:
        float: integrated loudness in LUFS
    """
    if data.ndim == 1:
        data = data[:, np.newaxis]

    num_channels = data.shape[1]
    sos = k_weight_filter(sr)

    # Channel weights per BS.1770 (1.0 for L/R, 1.41 for surround — we handle stereo)
    weights = [1.0] * num_channels

    # K-weight each channel
    k_weighted = np.zeros_like(data)
    for ch in range(num_channels):
        k_weighted[:, ch] = sosfilt(sos, data[:, ch])

    # 400 ms blocks with 75% overlap
    block_samples = int(0.4 * sr)
    step_samples = int(0.1 * sr)  # 100 ms step = 75% overlap
    num_blocks = max(0, (k_weighted.shape[0] - block_samples) // step_samples + 1)

    if num_blocks == 0:
        return -70.0  # Too short to measure

    # Mean square per block per channel, then weighted sum
    block_loudness = np.zeros(num_blocks)
    for i in range(num_blocks):
        start = i * step_samples
        end = start + block_samples
        for ch in range(num_channels):
            block_loudness[i] += weights[ch] * np.mean(k_weighted[start:end, ch] ** 2)

    # Convert to LUFS for gating
    block_lufs = -0.691 + 10.0 * np.log10(block_loudness + 1e-30)

    # Absolute gate: -70 LUFS
    abs_mask = block_lufs >= -70.0
    if not np.any(abs_mask):
        return -70.0

    abs_gated_power = np.mean(block_loudness[abs_mask])
    abs_gated_lufs = -0.691 + 10.0 * np.log10(abs_gated_power + 1e-30)

    # Relative gate: -10 dB below absolute-gated loudness
    rel_threshold = abs_gated_lufs - 10.0
    rel_mask = block_lufs >= rel_threshold
    if not np.any(rel_mask):
        return abs_gated_lufs

    rel_gated_power = np.mean(block_loudness[rel_mask])
    integrated_lufs = -0.691 + 10.0 * np.log10(rel_gated_power + 1e-30)

    return float(integrated_lufs)
```

## Clipping Detection

Detect true clipping vs. single-sample transient peaks. Consecutive clipped samples indicate real clipping artifacts.

```python
def detect_clipping(data, threshold=0.999):
    """Detect clipping in audio data.

    Args:
        data: numpy array, float64 in [-1, 1], shape (N,) or (N, 2)
        threshold: amplitude level to consider clipped (default 0.999)

    Returns:
        dict with clip_count, clip_percentage, max_consecutive, severity
    """
    # Flatten to mono for clipping analysis
    if data.ndim == 2:
        flat = np.max(np.abs(data), axis=1)
    else:
        flat = np.abs(data)

    clipped = flat >= threshold
    clip_count = int(np.sum(clipped))
    clip_percentage = float(clip_count / len(flat) * 100.0)

    # Find max consecutive clipped samples
    max_consecutive = 0
    current_run = 0
    for val in clipped:
        if val:
            current_run += 1
            max_consecutive = max(max_consecutive, current_run)
        else:
            current_run = 0

    # Severity: >3 consecutive = true clip (not just a transient peak)
    if clip_count == 0:
        severity = 'none'
    elif max_consecutive <= 3:
        severity = 'mild'  # Likely transient peaks, not audible distortion
    else:
        severity = 'severe'  # True clipping — audible distortion likely

    return {
        'clip_count': clip_count,
        'clip_percentage': clip_percentage,
        'max_consecutive': max_consecutive,
        'severity': severity,
    }
```

## Frequency Balance Check

Split the spectrum into five standard bands and measure energy per band. Compare against genre-specific targets.

```python
def frequency_balance(data, sr):
    """Measure RMS energy in 5 frequency bands.

    Bands: sub (20-60Hz), low (60-250Hz), mid (250-4000Hz),
           high (4000-12000Hz), air (12000-20000Hz)

    Args:
        data: (N,) or (N, 2) float64
        sr: sample rate

    Returns:
        dict mapping band name to RMS energy in dB
    """
    # Mix to mono for spectral analysis
    if data.ndim == 2:
        mono = np.mean(data, axis=1)
    else:
        mono = data

    nyq = sr / 2.0
    bands = {
        'sub':  (20, 60),
        'low':  (60, 250),
        'mid':  (250, 4000),
        'high': (4000, 12000),
        'air':  (12000, min(20000, nyq - 1)),
    }

    result = {}
    for name, (lo, hi) in bands.items():
        # Bandpass filter: 4th-order Butterworth
        sos = butter(4, [lo, hi], btype='band', fs=sr, output='sos')
        filtered = sosfilt(sos, mono)
        rms = np.sqrt(np.mean(filtered ** 2))
        result[name] = float(20.0 * np.log10(rms + 1e-12))

    return result


# Relative band levels in dB (referenced to mid band = 0 dB)
# Domain-specific skills extend these targets with their own entries
# (see mixing-music.md for music genre targets such as lo-fi, metal, hip-hop, etc.)
GENRE_SPECTRAL_TARGETS = {
    'generic':    {'sub': -1, 'low': 0, 'mid': 0, 'high': -1, 'air': -3},
    'electronic': {'sub': 2, 'low': 1, 'mid': 0, 'high': -1, 'air': -4},
    'acoustic':   {'sub': -6, 'low': -1, 'mid': 0, 'high': -2, 'air': -5},
    'ambient':    {'sub': -1, 'low': 0, 'mid': 0, 'high': -1, 'air': 1},
}
```

## Stereo Analysis

Evaluate stereo field health: correlation, width, and phase coherence.

```python
def stereo_analysis(data):
    """Analyze stereo field properties.

    Args:
        data: (N, 2) float64 stereo audio

    Returns:
        dict with correlation, width, mid_side_ratio, phase_issues
    """
    if data.ndim == 1:
        return {
            'correlation': 1.0,
            'width': 0.0,
            'mid_side_ratio_db': float('inf'),
            'phase_issues': False,
        }

    left = data[:, 0]
    right = data[:, 1]

    # Pearson correlation coefficient (overall)
    denom = np.sqrt(np.sum(left ** 2) * np.sum(right ** 2))
    if denom < 1e-12:
        correlation = 1.0
    else:
        correlation = float(np.sum(left * right) / denom)

    # Mid/side analysis
    mid = (left + right) * 0.5
    side = (left - right) * 0.5
    mid_rms = np.sqrt(np.mean(mid ** 2))
    side_rms = np.sqrt(np.mean(side ** 2))

    if side_rms < 1e-12:
        ms_ratio_db = 60.0  # Effectively mono
    else:
        ms_ratio_db = float(20.0 * np.log10(mid_rms / side_rms))

    # Stereo width estimate: 0 = mono, 1 = normal, >1 = wide/out-of-phase
    if mid_rms < 1e-12:
        width = 2.0  # All side, no mid — extreme width / phase cancel
    else:
        width = float(side_rms / mid_rms)

    # Phase issues: check for sustained negative correlation
    # Analyze in 50ms windows
    window_size = int(0.05 * 44100)
    num_windows = len(left) // window_size
    negative_windows = 0
    for i in range(num_windows):
        s = i * window_size
        e = s + window_size
        l_win = left[s:e]
        r_win = right[s:e]
        d = np.sqrt(np.sum(l_win ** 2) * np.sum(r_win ** 2))
        if d > 1e-12:
            win_corr = np.sum(l_win * r_win) / d
            if win_corr < 0:
                negative_windows += 1

    # Phase issues if >20% of windows have negative correlation
    phase_issues = (negative_windows / max(num_windows, 1)) > 0.2

    return {
        'correlation': correlation,
        'width': width,
        'mid_side_ratio_db': ms_ratio_db,
        'phase_issues': phase_issues,
    }
```

## Energy Map Comparison

Compare the generated audio's dynamics against the planned energy map to ensure the arrangement follows the intended intensity curve.

```python
def compare_to_energy_map(data, energy_map, sr):
    """Compare actual audio energy to the planned energy map.

    Args:
        data: (N, 2) float64 stereo audio
        energy_map: dict with keys:
            'bpm': float,
            'sections': list of dicts, each with:
                'name': str (e.g. 'intro', 'chorus'),
                'bars': int,
                'intensity': float (0.0 to 1.0)
        sr: sample rate

    Returns:
        dict with per-section comparison, flagged sections, climax match
    """
    bpm = energy_map['bpm']
    seconds_per_bar = 4.0 * 60.0 / bpm  # Assuming 4/4 time

    # Mix to mono for energy analysis
    if data.ndim == 2:
        mono = np.mean(data, axis=1)
    else:
        mono = data

    sections = energy_map['sections']
    comparisons = []
    max_planned_intensity = 0.0
    max_planned_idx = 0
    max_actual_rms = -120.0
    max_actual_idx = 0

    cursor = 0.0  # Current position in seconds
    for idx, section in enumerate(sections):
        start_sample = int(cursor * sr)
        duration = section['bars'] * seconds_per_bar
        end_sample = min(int((cursor + duration) * sr), len(mono))

        if start_sample >= len(mono):
            break

        segment = mono[start_sample:end_sample]
        rms = np.sqrt(np.mean(segment ** 2)) if len(segment) > 0 else 1e-12
        rms_db = 20.0 * np.log10(rms + 1e-12)

        # Track planned climax
        if section['intensity'] > max_planned_intensity:
            max_planned_intensity = section['intensity']
            max_planned_idx = idx
        if rms_db > max_actual_rms:
            max_actual_rms = rms_db
            max_actual_idx = idx

        comparisons.append({
            'name': section['name'],
            'planned_intensity': section['intensity'],
            'actual_rms_db': float(rms_db),
        })
        cursor += duration

    # Normalize actual RMS values relative to max for comparison
    if comparisons:
        actual_rms_vals = [c['actual_rms_db'] for c in comparisons]
        rms_range = max(actual_rms_vals) - min(actual_rms_vals)
        if rms_range > 0:
            for c in comparisons:
                c['actual_intensity'] = (c['actual_rms_db'] - min(actual_rms_vals)) / rms_range
        else:
            for c in comparisons:
                c['actual_intensity'] = 1.0

    # Flag sections deviating >3 dB from expected relative level
    # Map intensity to expected dB: intensity 1.0 = 0 dB, 0.0 = -24 dB
    flagged = []
    for c in comparisons:
        expected_db = -24.0 + c['planned_intensity'] * 24.0 + max(actual_rms_vals)
        # Offset so max expected = max actual
        expected_db = max(actual_rms_vals) - (1.0 - c['planned_intensity']) * 24.0
        deviation = abs(c['actual_rms_db'] - expected_db)
        c['expected_rms_db'] = float(expected_db)
        c['deviation_db'] = float(deviation)
        if deviation > 3.0:
            flagged.append(c['name'])

    climax_match = max_planned_idx == max_actual_idx

    return {
        'sections': comparisons,
        'flagged_sections': flagged,
        'climax_match': climax_match,
        'planned_climax': sections[max_planned_idx]['name'] if sections else None,
        'actual_climax': comparisons[max_actual_idx]['name'] if comparisons else None,
    }
```

## Auto-Fix Routines

Automatic correction for common issues. Applies non-destructive fixes and re-exports with a `_fixed` suffix.

```python
def auto_fix(filename, issues, sr=44100):
    """Attempt automatic fixes for detected issues.

    Args:
        filename: Path to the .wav file
        issues: Results dict from validate_wav()
        sr: sample rate

    Returns:
        list of fix descriptions applied
    """
    file_sr, raw = wavfile.read(filename)
    if raw.dtype == np.int16:
        data = raw.astype(np.float64) / 32768.0
    elif raw.dtype == np.int32:
        data = raw.astype(np.float64) / 2147483648.0
    else:
        data = raw.astype(np.float64)

    if data.ndim == 1:
        data = np.stack([data, data], axis=-1)

    fixes = []

    # Fix 1: LUFS normalization to -14 LUFS (streaming target)
    if not issues.get('lufs_pass', True):
        current_lufs = issues['lufs']
        target_lufs = -14.0
        diff_db = target_lufs - current_lufs
        gain = 10.0 ** (diff_db / 20.0)
        data = data * gain
        fixes.append(f"Normalized loudness: {current_lufs:.1f} -> {target_lufs:.1f} LUFS (gain {diff_db:+.1f} dB)")

    # Fix 2: Soft limiter for clipping
    if not issues.get('clipping_pass', True) and issues['clipping']['severity'] == 'severe':
        peak = np.max(np.abs(data))
        if peak > 1.0:
            # Soft clip via tanh, then scale to restore perceived loudness
            headroom_needed = 20.0 * np.log10(peak)
            data = np.tanh(data * 1.2) / np.tanh(1.2)  # Gentle saturation curve
            fixes.append(f"Applied soft limiter (tanh) — peak was {headroom_needed:+.1f} dB over")

    # Fix 3: Trim leading/trailing silence >500ms to 100ms
    mono_env = np.max(np.abs(data), axis=1)
    silence_threshold = 0.001  # -60 dBFS
    above_thresh = np.where(mono_env > silence_threshold)[0]
    if len(above_thresh) > 0:
        first_sound = above_thresh[0]
        last_sound = above_thresh[-1]

        pad_samples = int(0.1 * sr)  # 100ms pad
        max_silence = int(0.5 * sr)  # 500ms threshold

        trim_start = False
        trim_end = False

        if first_sound > max_silence:
            new_start = max(0, first_sound - pad_samples)
            trim_start = True
        else:
            new_start = 0

        if (len(mono_env) - last_sound) > max_silence:
            new_end = min(len(mono_env), last_sound + pad_samples)
            trim_end = True
        else:
            new_end = len(mono_env)

        if trim_start or trim_end:
            data = data[new_start:new_end]
            fixes.append(f"Trimmed silence — start: {trim_start}, end: {trim_end}")

    # Fix 4: Stereo phase warning (don't auto-fix, might be intentional)
    stereo = issues.get('stereo', {})
    if stereo.get('correlation', 1.0) < 0.1:
        fixes.append("WARNING: Very low stereo correlation ({:.2f}) — check for phase issues (not auto-fixed)".format(
            stereo['correlation']))

    # Fix 5: Frequency balance suggestion (don't auto-apply EQ)
    if not issues.get('freq_balance_pass', True):
        fb = issues.get('freq_balance', {})
        fixes.append(f"SUGGESTION: Frequency balance out of range — consider EQ adjustments. Current: {fb}")

    # Re-export with '_fixed' suffix
    if any(not f.startswith('WARNING') and not f.startswith('SUGGESTION') for f in fixes):
        # Ensure no clipping after all processing
        peak = np.max(np.abs(data))
        if peak > 0.999:
            data = data * (0.999 / peak)

        out_filename = filename.rsplit('.', 1)[0] + '_fixed.wav'
        out_int16 = np.clip(data * 32768.0, -32768, 32767).astype(np.int16)
        wavfile.write(out_filename, sr, out_int16)
        fixes.append(f"Exported fixed file: {out_filename}")

    return fixes
```

## Genre-Specific Targets

Loudness and dynamics targets. Domain-specific skills extend these targets with their own entries (see mixing-music.md for music genre targets).

```python
GENRE_TARGETS = {
    'generic': {
        'lufs_target': -14,
        'lufs_tolerance': 3,
        'crest_factor_range': (6, 18),
        'max_peak_dbtp': -1.0,
        'spectral_profile_key': 'generic',
    },
    'ambient': {
        'lufs_target': -18,
        'lufs_tolerance': 4,
        'crest_factor_range': (8, 20),
        'max_peak_dbtp': -1.0,
        'spectral_profile_key': 'ambient',
    },
}
```

## Validation Report Template

Formatted ASCII report with pass/fail indicators for each check.

```python
def print_validation_report(results):
    """Print a formatted validation report to console.

    Args:
        results: dict from validate_wav()
    """
    def icon(passed):
        return '\u2713' if passed else '\u2717'

    print("\n" + "=" * 56)
    print("  POST-GENERATION QUALITY VALIDATION REPORT")
    print("=" * 56)

    # Peak & loudness
    peak = results.get('peak', {})
    lufs = results.get('lufs', -70)
    print(f"\n  Peak Level")
    print(f"    {icon(results.get('peak_pass', False))}  Peak:          {peak.get('peak_db', -99):.1f} dBFS")
    print(f"       RMS:           {peak.get('rms_db', -99):.1f} dBFS")
    print(f"       Crest Factor:  {peak.get('crest_factor', 0):.1f} dB")

    print(f"\n  Loudness")
    print(f"    {icon(results.get('lufs_pass', False))}  Integrated:    {lufs:.1f} LUFS")

    # Crest factor
    print(f"\n  Dynamics")
    print(f"    {icon(results.get('crest_pass', False))}  Crest Factor:  {peak.get('crest_factor', 0):.1f} dB")

    # Clipping
    clip = results.get('clipping', {})
    print(f"\n  Clipping")
    print(f"    {icon(results.get('clipping_pass', False))}  Severity:      {clip.get('severity', '?')}")
    print(f"       Clipped:       {clip.get('clip_count', 0)} samples ({clip.get('clip_percentage', 0):.3f}%)")
    print(f"       Max Run:       {clip.get('max_consecutive', 0)} consecutive")

    # Frequency balance
    fb = results.get('freq_balance', {})
    print(f"\n  Frequency Balance")
    print(f"    {icon(results.get('freq_balance_pass', False))}  Band Energies (dBFS):")
    for band in ['sub', 'low', 'mid', 'high', 'air']:
        if band in fb:
            print(f"       {band:>5s}: {fb[band]:>7.1f} dB")

    # Stereo
    st = results.get('stereo', {})
    print(f"\n  Stereo Field")
    print(f"    {icon(results.get('stereo_pass', False))}  Correlation:   {st.get('correlation', 0):.3f}")
    print(f"       Width:         {st.get('width', 0):.3f}")
    print(f"       M/S Ratio:     {st.get('mid_side_ratio_db', 0):.1f} dB")
    print(f"       Phase Issues:  {'Yes' if st.get('phase_issues') else 'No'}")

    # Energy map (optional)
    if 'energy_map' in results:
        em = results['energy_map']
        print(f"\n  Energy Map Match")
        print(f"    {icon(results.get('energy_map_pass', False))}  Climax Match:  {'Yes' if em.get('climax_match') else 'No'}")
        print(f"       Planned:       {em.get('planned_climax', '?')}")
        print(f"       Actual:        {em.get('actual_climax', '?')}")
        if em.get('flagged_sections'):
            print(f"       Flagged:       {', '.join(em['flagged_sections'])}")

    # Overall verdict
    overall = results.get('overall_pass', False)
    print("\n" + "-" * 56)
    if overall:
        print(f"  VERDICT:  \u2713  PASS  — Ready to present")
    else:
        failed = [k.replace('_pass', '') for k in results if k.endswith('_pass') and not results[k]]
        print(f"  VERDICT:  \u2717  FAIL  — Issues: {', '.join(failed)}")
    print("=" * 56 + "\n")
```
