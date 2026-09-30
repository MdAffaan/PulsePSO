# PulsePSO

A fuzzy-logic alarm for lethal ventricular arrhythmias in noisy ICU ECG, with its parameters tuned by Particle Swarm Optimization (PSO).

Built for **OptiForge 2026**, IEEE EMBS track: *Biomedical Signals & Intelligent Systems (Fuzzy Signal Classifier + Particle Swarm Optimizer)*.

## The problem

ICU ECG is corrupted by baseline wander, motion artifacts and muscle (EMG) noise. A bedside monitor must raise an alert quickly for lethal rhythms, must never suppress a real emergency, and must not bury nurses in false alarms. Its filtering must preserve QRS amplitude and the ST segment, and it must be light enough for low-power wearable hardware.

## How it works

```
ECG -> causal band-pass filter (0.5-40 Hz) -> Pan-Tompkins R-peak detection
    -> heart rate + RR irregularity over 3 s sliding windows
    -> fuzzy classifier -> danger score (0-100) -> alert when score >= threshold
                               ^
                               | PSO tunes the fuzzy breakpoints and the alert threshold
```

**1. Filter.** Causal Butterworth band-pass: 2nd-order 0.5 Hz high-pass (removes baseline wander) and 4th-order 40 Hz low-pass (removes high-frequency muscle noise). "Causal" means it uses only past samples, so it can run in real time. Its delay is 4 samples (11.1 ms at 360 Hz).

**2. R-peak detection.** Pan-Tompkins style: 5-15 Hz band-pass, derivative, squaring, 150 ms moving average, then peak picking against an adaptive threshold (35% of the running 2 s maximum) with a 250 ms refractory period. Each detection is snapped back to the true peak in the raw signal.

**3. Features.** For every 3 s window (a new decision every 0.5 s in the episode evaluation): heart rate and RR irregularity (standard deviation of RR divided by mean RR).

**4. Fuzzy classifier.** Mamdani-style with trapezoid membership functions: min for AND, max to combine rules, centroid to defuzzify into a danger score from 0 to 100.

| Rule | Output danger |
|---|---|
| heart rate is high | high |
| heart rate is low | medium |
| heart rate is normal and rhythm is regular | low |
| heart rate is normal and rhythm is irregular | medium |

**Fail-safe (on by default, `failsafe=True`):** a window with fewer than 2 detected beats is scored 100 and raises the alert. Absent QRS is what ventricular fibrillation looks like, so staying silent there would violate the requirement never to suppress a real emergency.

**5. PSO.** Each particle is 5 numbers: `[HR-high start, HR-high width, irregularity start, irregularity width, alert threshold]`. Trapezoid points are built as "start + width", so they can never cross. Settings: 20 particles, 30 iterations, inertia 0.7, own-best and swarm-best pull 1.5 each, random seed 0. PSO was chosen because sensitivity, false alarms and alert delay are not differentiable, and PSO needs few parameters and little compute. The fuzzy rules stay readable and auditable.

**Fitness (minimize):**

```
1.0 * (1 - sensitivity) + 0.5 * (1 - specificity)
+ 0.5 * (late or missed episodes / total) + 0.2 * min(false alarms per hour / 100, 2)
```

A missed lethal event carries the largest weight.

## Results

All numbers below are printed by `notebooks/Opti_Forge.ipynb`.

### Alarm on lethal-rhythm episodes (MIT-BIH)

Data: the first 10 episodes annotated as lethal ventricular rhythm (VT, VFL or VF) that last at least 3 s, each with 60 s of signal before and after (episodes capped at 90 s). Episodes are split alternately into 5 train and 5 test. PSO sees only the training episodes.

| Attempt | Split | Fitness | Sensitivity | Specificity | Late or missed | False alarms / hour |
|---|---|---|---|---|---|---|
| 1: hand-set fuzzy, threshold 70 | train | 0.869 | 0.411 | 0.944 | 1/5 | 76.2 |
| 1: hand-set fuzzy, threshold 70 | test | 0.570 | 0.729 | 0.941 | 1/5 | 84.6 |
| 2: PSO-tuned fuzzy | train | 0.586 | 0.630 | 0.922 | 0/5 | 88.9 |
| 2: PSO-tuned fuzzy | test | 0.552 | 0.800 | 0.913 | 1/5 | 104.2 |

Tuned values: HR-high starts at 102.8 bpm with width 41.2; irregularity starts at 0.164 with width 0.083; alert threshold 52.9.

Definitions: sensitivity and specificity are per 3 s window. A window counts as positive if it overlaps the episode by at least 1.5 s and negative if it is entirely outside the episode. "Late or missed" counts episodes with no alert within 3 s of onset. False alarms per hour counts alert onsets in negative windows per hour of negative time.

**What changed and why (Attempt 1 to 2):** PSO tuned the five fuzzy parameters on the training episodes because the hand-set system missed too many (train sensitivity 0.411). Test sensitivity rose from 0.729 to 0.800, but false alarms also rose from 84.6 to 104.2 per hour.

### Sanity checks

- Normal rhythm (MIT-BIH record 100, first 60 s): maximum danger 50.0 and zero alert windows, for both the hand-set and the tuned configuration.
- Synthetic tachycardia (75 bpm switching to 170 bpm at 20 s): zero false alerts before onset, and the alert fires 1.5 s after onset.

### Noise robustness of filter plus beat detection

Record 100, first 60 s (74 beats), mixed with real noise from the MIT-BIH Noise Stress Test Database (`bw` baseline wander, `em` electrode motion, `ma` muscle artifact). F1 is beat-detection F1 with a 75 ms tolerance.

| Noise | Input SNR (dB) | SNR after filter (dB) | SNR gain (dB) | F1 unfiltered | F1 filtered |
|---|---|---|---|---|---|
| baseline wander | 6 | 8.90 | 2.90 | 1.000 | 1.000 |
| baseline wander | 0 | 8.11 | 8.11 | 1.000 | 1.000 |
| baseline wander | -6 | 5.93 | 11.93 | 1.000 | 1.000 |
| baseline wander | -12 | 1.87 | 13.87 | 1.000 | 1.000 |
| electrode motion | 6 | 7.42 | 1.42 | 1.000 | 1.000 |
| electrode motion | 0 | 4.12 | 4.12 | 1.000 | 1.000 |
| electrode motion | -6 | -0.90 | 5.10 | 0.974 | 0.980 |
| electrode motion | -12 | -6.66 | 5.34 | 0.800 | 0.802 |
| muscle artifact | 6 | 6.56 | 0.56 | 1.000 | 1.000 |
| muscle artifact | 0 | 2.91 | 2.91 | 0.993 | 0.993 |
| muscle artifact | -6 | -2.21 | 3.79 | 0.899 | 0.904 |
| muscle artifact | -12 | -7.94 | 4.06 | 0.658 | 0.692 |

Muscle artifact is the hardest case: the filter gives only a small SNR gain because muscle noise overlaps the QRS frequency band.

### Timing, jitter and resources

- **R-peak timing jitter** (standard deviation, 75 ms tolerance): 1.0 ms on clean signal, 1.0 ms under baseline wander at -6 dB, 9.0 ms under muscle noise at -6 dB. Mean absolute error: 0.3 ms, 0.4 ms and 1.8 ms.
- **Speed:** about 25 ms to filter, detect beats and score 60 s of ECG (under 0.5 ms per second of signal).
- **Memory:** about 1.3 MB peak Python memory for the same 60 s (measured with `tracemalloc`). The pipeline is plain NumPy, SciPy and pandas, with no model files.

### Why the filter is fixed instead of tuned

We also ran PSO over the filter cutoffs and detector threshold (muscle noise at -6 dB). It found a 1.94 Hz high-pass, 27.4 Hz low-pass and 0.29 threshold. Across the 12 noise conditions above, its mean F1 was identical to the default filter (0.948 vs 0.948) and its mean output SNR was lower (1.56 dB vs 2.34 dB). It also preserved only 73.8% of the median QRS amplitude, against 91.6% for the fixed 0.5-40 Hz filter. A higher high-pass cutoff distorts the ST segment, so the filter stays fixed at 0.5-40 Hz and PSO tunes only the classifier.

## How each hard constraint is addressed

| Constraint | What we did | Status |
|---|---|---|
| Alert within 3 s | Decision every 0.5 s on 3 s windows. Synthetic tachycardia alerts in 1.5 s. On real test episodes, 4 of 5 alerted within 3 s. | Partly met, not guaranteed |
| Never suppress real distress | No rule suppresses alerts because of noise. Fail-safe alerts on windows with too few beats. Missed events carry the largest fitness weight. | Design addresses it, but test sensitivity is 0.800, so some lethal windows are still missed |
| Preserve QRS and ST | Causal 0.5 Hz high-pass, fixed. Median QRS amplitude kept: 91.6%. | QRS measured, ST not measured directly |
| Low memory | About 1.3 MB peak, plain NumPy, no stored models. | Met on a laptop, not tested on wearable hardware |

## Limitations

- Only 10 episodes, and the same MIT-BIH records can appear in both train and test, so results are indicative and not statistically strong.
- False alarms are high (85 to 104 per hour around episodes), which works against the alarm-fatigue objective. The fail-safe can add more when the detector misses beats. A separate lower-priority "signal loss" alert would be the natural next step.
- Episodes shorter than 3 s (most MIT-BIH VT episodes) are excluded, because they cannot be alerted within 3 s of onset.
- Beat-based features cannot describe ventricular fibrillation once no beats are detectable. A frequency-based feature (power in the 3-7 Hz band) is the next planned addition. The VFDB database is not used yet.
- ECG only: no PPG or separate EMG channel.
- Specificity counts every non-episode window, including other arrhythmias, as negative.
- Beat positions are shifted back by the filter's measured delay before scoring.
- This is a research prototype, not a medical device.

## Reproduce

1. Open `notebooks/Opti_Forge.ipynb` in Google Colab and choose **Runtime > Run all**. It streams data from PhysioNet, so it needs internet and takes a few minutes.
2. Locally: `pip install -r requirements.txt`, then open the notebook in Jupyter or VS Code.
3. Unit tests (synthetic ECG, no downloads): `pytest`.

All PSO runs use `np.random.seed(0)`.

## Repository layout

```
notebooks/Opti_Forge.ipynb   full pipeline, experiments and results
src/filtering.py             causal filter, filter delay and SNR helpers
tests/test_filtering.py      unit tests on synthetic signals
index.html                   simulated UI demo (illustration only, not real results)
requirements.txt
```

## Data and credits

MIT-BIH Arrhythmia Database and MIT-BIH Noise Stress Test Database, from PhysioNet (Goldberger et al., *Circulation*, 2000). Data is streamed at run time and is not stored in this repository.
