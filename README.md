# GuitarBuddy

A browser-based learning tool that listens to your guitar and shows on a fretboard which note you played, colored by whether it's in a chosen scale (green = in scale, red = out).

Single HTML file. No build, no dependencies.

## Run

```bash
python3 -m http.server 8000
```

Open `http://localhost:8000`. Mic access requires `localhost` or HTTPS.

## What it does

- **Listens** via Web Audio API (microphone input, no AGC / noise suppression / echo cancellation, since those mangle pitch).
- **Detects pitch** using the McLeod Pitch Method (NSDF autocorrelation with parabolic peak interpolation and an octave-up guard for guitar's strong 2nd harmonic).
- **Picks fretboard position** for the detected MIDI value using:
  - A ±3 fret hard window around the previous note (human hand reach)
  - Diagonal-jump penalty (penalizes simultaneous fret + string changes)
  - Multi-feature timbre distance vs calibrated fingerprints
- **Validates octave** post-detection by comparing measured timbre features against expected fingerprints for the detected MIDI and the octave above/below.
- **Renders** an SVG fretboard with scale notes highlighted, a live tuner strip (cents bar), and a pulse overlay at the picked position.

## Calibration

The hardest part of mapping guitar audio to fretboard positions is string disambiguation: the same MIDI note can be played on multiple strings, and pitch alone can't distinguish them.

Click **Calibrate** before playing. The session walks through 30 prompts (5 frets × 6 strings: frets 0, 4, 8, 12, 15). For each, it captures 12 samples of the steady-state portion of the pluck (150–600 ms after attack), trims outliers, and stores four features per (string, fret) point:

- `b`  — brightness (energy ratio of harmonics 3–8 vs 1–2)
- `h2` — 2nd-harmonic strength relative to fundamental
- `h3` — 3rd-harmonic strength relative to fundamental
- `B`  — inharmonicity coefficient (string stiffness signature)

The calibration auto-saves to `localStorage` (`guitarbuddy.calibration.v2`) and auto-loads on refresh. To wipe it:

```js
localStorage.removeItem('guitarbuddy.calibration.v2')
```

To inspect:

```js
JSON.parse(localStorage.getItem('guitarbuddy.calibration.v2')).table
```

## Architecture

```
mic ──► WebAudio ──► AnalyserNode (fftSize 2048)
                      │
                      ├─► getFloatTimeDomainData ──► MPM pitch detection ──► MIDI
                      │                                                       │
                      └─► getFloatFrequencyData ──► computeFeatures ──┐       │
                                                                      ▼       ▼
                                          calibTable ──► correctOctave / pickPosition
                                                                      │
                                                                      ▼
                                                       SVG overlay (green / red ring)
```

Tuner display updates every animation frame for responsiveness. Fretboard position pick only commits after 2 consecutive frames agree on the MIDI value, so single-frame pitch glitches don't poison the previous-position anchor.

## Configuration knobs

All tunable constants live inside the single IIFE in `index.html`:

| Constant | Default | What it does |
|---|---|---|
| `BUFFER_SIZE` | 2048 | Analyser FFT size and time-domain buffer |
| `SILENCE_RMS` | 0.006 | RMS below this = treat as silence |
| `CLARITY_FLOOR` | 0.4 | MPM NSDF threshold for accepting a detection |
| `REQUIRED_VOTES` | 2 | Consecutive frames that must agree on MIDI before re-picking |
| `POS_WINDOW` | 3 | Max fret distance from previous position (hard limit) |
| `DIAGONAL_PENALTY` | 1.2 | Cost per (Δfret × Δstring) — penalizes diagonal jumps |
| `FEATURE_W` | 0.6 | How much measured-vs-calibrated timbre distance weighs in position cost |
| `CALIB_TARGET_SAMPLES` | 12 | Brightness samples captured per calibration cell |
| `CALIB_ATTACK_SKIP_MS` | 150 | Skip the attack transient when sampling |
| `CALIB_CAPTURE_WINDOW_MS` | 600 | Stop sampling after this (decay → noise floor) |

## Known limitations

- **Monophonic only.** Chords aren't detected; polyphonic pitch detection is much harder.
- **Adjacent-string disambiguation is fundamentally noisy.** Even with multi-feature calibration, accuracy plateaus around 70–80% on real audio in real rooms. The same fret on adjacent strings often has overlapping timbre fingerprints. The reliable solution is hardware (a hex pickup, one coil per string) — which is why commercial guitar-to-MIDI products use one.
- **Bluetooth / wireless mics add latency and compression artifacts.** Wired mic or built-in laptop mic works best.
- **iOS Safari** requires a user gesture to start the AudioContext (handled by the Start button).

## Why not just train a neural net

Probably should. A small CNN on log-mel spectrograms with per-user fine-tuning would crush this. The current approach is intentionally interpretable and zero-dependency: every step is a knob you can read and tune in one HTML file.
