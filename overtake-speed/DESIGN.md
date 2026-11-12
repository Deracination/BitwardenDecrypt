# Design: Overtake speed measurement with a handlebar-mounted iPhone

Status: Draft · Companion to [PROBLEM.md](PROBLEM.md)

## 1. Overview

An iOS app ("the app") records audio and location continuously during a ride. An
on-device signal-processing pipeline:

1. finds overtakes as loudness peaks with a falling-pitch signature,
2. tracks the vehicle's sound frequency over time during each overtake,
3. fits a physical Doppler model to that frequency track to get the vehicle's speed
   relative to the bike (and the passing distance),
4. adds the rider's GPS speed to get the vehicle's ground speed,
5. scores confidence, and stores the pass in a ride log, which can trigger a spoken
   cue if enabled.

The design favours classical DSP plus model fitting over end-to-end machine learning.
Classical methods need no large labelled dataset, explain their results, and each stage
can be tested in isolation. ML can come in later for event classification and confidence
calibration once real data has been collected.

```
 Mic(s) ──► Capture ──► Pre-process ──► Feature extraction ──┬──► Event detector
 (48 kHz)   AVAudio-    (decimate,       (STFT, log-freq      │        │ candidate window
            Engine      HPF, wind        spectra, loudness,   │        ▼
                        suppression)     front/back balance)  └──► Frequency tracker
                                                                        │ f(t), weights
 GPS ──► Location service ──► rider speed v_b(t) ─────────┐             ▼
                                                          └──► Doppler model fit
 Temperature ──► speed of sound c ────────────────────────┘             │
                                                                        ▼
                                                               Confidence & gating
                                                                        │
                                                    ┌───────────────────┼────────────┐
                                                    ▼                   ▼            ▼
                                               Pass store         Live cue      Clip saver
                                              (ride log)         (optional)    (opt-in)
```

## 2. Signal model

Coordinates follow the road, positive forward. The bike moves at `v_b`, the vehicle at
`v_c`, the relative speed is `Δv = v_c − v_b`, the lateral offset is `d`, and the time of
closest approach is `t0`. The vehicle's position relative to the bike is

```
Δx(t) = Δv · (t − t0)
cosα(t) = −Δx / sqrt(Δx² + d²)        (+1 far behind, −1 far ahead)
```

The observed frequency of a source component `f0` is

```
f(t) = f0 · (c − v_b·cosα) / (c − v_c·cosα)
```

which approaches the forms given in PROBLEM.md when the vehicle is far behind or far
ahead. Taking logs:

```
ln f(t) = ln f0 + ln(c − v_b·cosα(t)) − ln(c − v_c·cosα(t))  [+ g(t)]
```

`g(t)` is a slow drift in the source frequency, for example a driver accelerating or an
engine changing revs. It is modelled as a low-order polynomial, linear by default, in
`(t − t0)`.

Model parameters: `θ = {ln f0, v_c, t0, τ, g1}`, where `τ = d/Δv` is the transition
timescale. The asymptotes fix `Δv`, the shape of the transition fixes `τ`, and the two
together give `d = τ·Δv`. `v_b` and `c` are inputs, not free parameters.

The emission-time delay is ignored. At 2 m and 343 m/s it is about 6 ms, which is
negligible next to analysis frames of about 250 ms. The simulator models it exactly so
that the error can be measured.

**Why fit the full curve rather than compare the start and end frequencies?** The
start/end ratio cannot tell Doppler apart from a pitch rise due to acceleration. The full
curve can, because Doppler has a characteristic sigmoid shape centred on `t0`, while
drift is smooth across the whole window.

## 3. Platform and capture

### 3.1 Audio session

- `AVAudioSession` category `.playAndRecord` with options `.mixWithOthers` and
  `.allowBluetoothA2DP`, so the user's music and the spoken cues can coexist. Mode
  `.measurement` turns off AGC, noise suppression and voice processing. Those stages
  would distort spectra and levels.
- Capture at 48 kHz via `AVAudioEngine` input tap, buffer ≈ 4096 frames.
- **Microphone configuration** (to be decided by Spike S1, §10). Candidates:
  - **Stereo capture** (`setPreferredInputOrientation`, data source with
    `.stereo` polar pattern, iPhone XS and later). This gives two channels for wind
    suppression (§4.2) and for judging whether sound comes from in front or behind.
  - **Single mic**: whichever data source (bottom / front / back) shows the least wind
    noise. When mounted, the back mic faces forward and the front mic faces the rider,
    and therefore traffic coming from behind.
- Optional accessory: a small foam windscreen clipped over the chosen mic port on the
  mount.

### 3.2 Background operation

- `UIBackgroundModes`: `audio`, `location`.
- `CLLocationManager` with `activityType = .fitness`, best accuracy, background
  updates on, `pausesLocationUpdatesAutomatically = false` while a ride is active.
- Handle `AVAudioSession.interruptionNotification` and route changes. On interruption,
  close the current analysis segment, record a gap in the ride log, and resume
  automatically.

### 3.3 Rider speed and speed of sound

- `v_b(t)` comes from `CLLocation.speed`, gated on `speedAccuracy`. Readings are
  smoothed with a 1-D Kalman filter, which takes optional `CMMotionManager` forward
  acceleration as a control input. Within a pass window, the filtered value is
  averaged over about ±3 s around `t0`.
- Optional: a BLE Cycling Speed and Cadence sensor (GATT service 0x1816) replaces GPS
  speed when it is paired.
- `c = 331.3 + 0.606·T` m/s. `T` comes from WeatherKit for the ride's start location,
  updated hourly, or from a manual setting. The default is 15 °C.

## 4. Processing pipeline

All DSP uses Accelerate/vDSP on a dedicated serial queue. The audio thread only copies
into a lock-free ring buffer.

### 4.1 Pre-processing

1. Decimate from 48 kHz to 16 kHz with a polyphase FIR. The useful band is about
   40 Hz – 4 kHz: engine and exhaust harmonics from roughly 30 to 600 Hz, and tyre-tread
   tones and road noise from about 300 Hz to 3 kHz.
2. Apply a 4th-order high-pass at 40 Hz to remove handling thumps and the lowest wind
   rumble.
3. Keep a 30 s ring buffer of decimated audio (about 1 MB) so that a whole pass window
   can be re-analysed once an event is confirmed.

### 4.2 Wind and self-noise suppression

- **Two-channel coherence mask** (stereo mode only). Wind is turbulence at each mic
  port, so it is largely incoherent between mics. Traffic noise is a far-field
  acoustic wave and is coherent. For each STFT bin, compute the short-term
  magnitude-squared coherence and attenuate bins whose coherence is below a threshold.
- **Spectral floor tracking.** A minimum-statistics noise estimate per bin over about
  2 s handles stationary self-noise, such as tyre hum and freewheel. The floor freezes
  while an event is in progress, so that a long overtake does not get absorbed into
  it.
- **Wind level indicator.** Energy below 150 Hz together with low coherence gives a
  per-frame wind score. This feeds the confidence estimate (§4.6) and the live
  "too windy" indicator.

### 4.3 Feature extraction

Two STFT resolutions run in parallel:

| Stream | Window | Hop | Use |
|---|---|---|---|
| Fast | 1024 (64 ms) | 16 ms | Loudness, front/back balance, event detection |
| Fine | 8192 (512 ms), Hann, zero-padded ×2 | 64 ms | Frequency tracking |

Per frame:

- Band loudness (A-weighted, 100 Hz – 3 kHz) in dB above the adaptive floor.
- Front/back balance in stereo mode: inter-channel level and phase difference in the
  300 Hz – 2 kHz band. This indicates whether the source is behind or ahead of the
  phone.
- Log-frequency spectrum: the fine STFT magnitude resampled onto a 1/96-octave grid
  (0.72 % per bin) from 40 Hz to 4 kHz (about 640 bins).

### 4.4 Event detector

A state machine runs on the fast stream.

1. **Idle → Candidate** when loudness rises ≥ 8 dB above the floor and keeps rising for
   ≥ 0.5 s. Approaching overtakers ramp up slowly because their closing speed is low.
2. **Candidate → Peak**: track the loudness maximum.
3. **Peak → Confirm** when loudness falls ≥ 6 dB from the peak, or after 6 s.
4. **Confirm** checks the overtake signature:
   - **Net negative frequency shift**: the log-spectrum cross-correlation between the
     2 s before the peak and the 2 s after it shows a downward shift.
   - **Direction**: in stereo mode, the front/back balance moves from *behind* to
     *ahead*. Oncoming vehicles go ahead→behind, and vehicles the rider passes move
     backwards.
   - **Plausibility**: the shift implies `Δv` between 2 and 40 m/s.
5. On confirmation, extract the analysis window `[t_peak − 8 s, t_peak + 6 s]` from
   the ring buffer and hand it to the frequency tracker. In single-mic mode, oncoming
   traffic is rejected by its much faster loudness rise and its implied `Δv`, which
   would put the vehicle's ground speed above the plausible upper limit. A learned
   classifier (§9) replaces these heuristics later.

If two loudness peaks fall within 3 s of each other, the event is flagged as
`multiVehicle`. In v1 the speed estimate is still attempted, but confidence is capped.

### 4.5 Frequency tracker

The goal is a track `s(t) = ln f(t) − ln f_ref` with a weight per frame. Two trackers
run, and their agreement is a key confidence signal.

**A. Harmonic tracker (primary for combustion vehicles).**
- For each fine frame, compute a harmonic-sum salience over candidate fundamentals from
  25 to 300 Hz, using 8 harmonics on the log-frequency grid with parabolic sub-bin
  interpolation.
- A Viterbi path through the salience matrix smooths the track. Transition costs
  penalise jumps larger than 3 % per hop. Paths through octave errors are allowed but
  penalised.
- Output: `ln f(t)` and the salience as the weight.

**B. Spectral-shift tracker (primary for EVs, a cross-check otherwise).**
- Doppler scales *every* frequency by the same factor, which is a pure translation on
  a log-frequency axis. Each frame's log spectrum, whitened and restricted to
  300 Hz – 3 kHz where tyre-tread tones and road noise dominate, is cross-correlated
  against two references: the highest-SNR frame before `t0`, and the highest-SNR frame
  after `t0`.
- Sub-bin peak interpolation of the correlation gives the shift, and the peak height
  gives the weight. The two half-tracks are joined using frames near `t0` where both
  are valid. Otherwise they are fitted with separate offsets: the absolute level is not
  needed, only the change.
- Risk: the broadband spectrum changes between approach and recession because the
  sources are directional (front tyres and engine ahead, exhaust behind). Tread tones,
  which are narrowband and come from all tyres, are the most reliable part. See §11.

**Disambiguating Doppler from rev changes.** An engine rev change scales the engine
harmonics but *not* the tyre and tread tones. Doppler scales both. A large disagreement
between trackers A and B therefore means the source changed pitch, so confidence is
reduced and the drift term is given more freedom.

### 4.6 Model fit and confidence

- Fit the model from §2 to the chosen track, or to both tracks jointly with
  separate `ln f0` values. Use Levenberg–Marquardt with a Huber loss and weights from
  the tracker.
- Initialisation: `t0` at the loudness peak; `v_c` from the asymptotic ratio formula in
  PROBLEM.md, computed from the medians of the first and last 1.5 s; `τ` = 0.3 s;
  drift = 0.
- Bounds: `v_c − v_b` ∈ [1, 45] m/s, `τ` ∈ [0.02, 3] s, `d` ∈ [0.3, 15] m.
- Uncertainty: the covariance of `v_c` from the Jacobian at the solution, combined with
  the GPS speed uncertainty and the uncertainty in `c`.

The **confidence score** is in [0, 1]. A logistic combination of the following features
is first hand-tuned, then calibrated on field data (§8):
- σ(v_c) from the fit
- weighted RMS residual
- number of valid frames on each side of `t0` (at least 1.5 s on each side is
  required)
- median wind score in the window
- tracker A/B agreement
- the direction check result and whether the event is multi-vehicle

Passes below 0.5 confidence are stored but marked "unreliable". They are hidden by
default and never announced live.

### 4.7 Compute and latency budget

- Fast STFT: about 63 FFTs/s of 1024 points. Fine STFT: about 16 FFTs/s of 16384 points.
  Using vDSP, this is well under 1 % CPU on an A-series chip.
- The per-event Viterbi and LM fit take tens of milliseconds.
- Result latency is about 6 s after the vehicle passes, because enough of the receding
  part of the track is needed.
- Battery target: under 8 % per hour. GPS and the always-on audio input dominate. DSP
  is negligible.

## 5. App structure

Swift and SwiftUI, iOS 17+, with SwiftData for persistence.

| Module | Responsibility |
|---|---|
| `AudioCapture` | Session config, engine tap, ring buffer, interruption handling |
| `DSP` | Decimation, filters, STFTs, log-freq resampling, coherence (vDSP) |
| `EventDetector` | State machine (§4.4) |
| `FrequencyTracker` | Harmonic + spectral-shift trackers (§4.5) |
| `DopplerEstimator` | Model, LM fit, uncertainty, confidence (§4.6) |
| `RiderState` | GPS/BLE speed, Kalman filter, temperature/`c` |
| `PassStore` | `Ride`, `Pass`, `Gap` models; export |
| `Announcer` | Spoken/haptic cues via `AVSpeechSynthesizer` |
| `Recorder` (dev) | Raw audio + sensor logging for dataset collection |
| `UI` | Ride, history, ride detail map, pass detail, settings |

### Data model

```
Ride   { id, start, end, distance, tempC, micConfig, appVersion, passes[], gaps[] }
Pass   { id, rideId, t0, lat, lon, riderSpeed, vehicleSpeed, relSpeed,
         speedSigma, passDistance?, confidence, flags[multiVehicle, ev, windy],
         trackerUsed, fitResidual, spectrogramPNG?, clipURL? }
Gap    { start, end, reason }   // interruption, low GPS, mic unavailable
```

### UI

- **Ride screen**: large Start/Stop button. While riding it shows the speed of the last
  pass, the number of passes, and indicators for wind and GPS quality. The screen can
  stay off.
- **History**: a list of rides with summary statistics (count, median speed, number
  above threshold).
- **Ride detail**: a MapKit map with passes as pins coloured by speed, plus a histogram.
- **Pass detail**: speed ± σ, distance, confidence, and a spectrogram with the fitted
  curve overlaid, so the user can judge whether to trust it. The audio clip is shown if
  one was kept.
- **Settings**: units, temperature source, mic mode, live cue threshold, audio clip
  retention (off by default), and debug recorder mode.

### Live cues

These are off by default. When enabled, the app announces a pass if confidence ≥ 0.7
and the speed is above the user's threshold, for example "Fifty-two". Speech uses
ducking so that it does not interrupt music for long.

### Export

All exports are started by the user from the share sheet:

- CSV of passes
- GPX of the track, with passes as waypoints
- JSON of the full ride

## 6. Privacy

- All processing happens on the device. Nothing is uploaded, and no account or server is
  needed.
- Audio is held only in a 30 s RAM ring buffer. Pass clips (±5 s) are saved only if the
  user opts in, and they can be deleted per ride or all at once.
- Clips are low-pass filtered at 4 kHz, which makes nearby speech hard to understand.
  The UI states that clips may contain voices.
- Location data stays in the local store and is shared only by user-initiated export.

## 7. Development tooling

- **Python research harness** (`research/`, using numpy/scipy). The algorithms are
  prototyped here first and run on recorded datasets. It is the reference
  implementation.
- **Pass simulator**: a moving point source with emission delay, 1/R amplitude, ground
  reflection, and a configurable mix of harmonics, tread tones and pink broadband
  noise. It supports acceleration and rev changes, and can be mixed with real wind and
  bike-noise recordings at a chosen SNR. It produces labelled test cases at any
  scale.
- **Golden test vectors**: the Swift DSP and estimator are checked against Python
  outputs on shared WAV + JSON fixtures in CI. Speed must match within 0.1 km/h.
- **Recorder mode** in the app logs multichannel audio, GPS, motion, and the
  pipeline's intermediate outputs. A desktop review tool shows the spectrogram so that
  passes can be labelled after the ride.

## 8. Validation plan

| Stage | Data | Ground truth | Gate |
|---|---|---|---|
| Simulated | 10k synthetic passes | Exact | Median error < 1 km/h at SNR ≥ 10 dB |
| Controlled | ≥ 100 passes, helper driver | Car GPS logger at steady speeds, 30–90 km/h | ±5 km/h on ≥ 80 % accepted |
| Reference | ≥ 500 passes on normal rides | Rear bike radar closing speed (where available) and/or roadside radar gun sessions | Same, plus detection ≥ 80 %, false positives < 1 per 10 min |
| Calibration | All labelled data | — | Confidence calibrated: of passes scored ≥ 0.7, ≥ 90 % within ±5 km/h |

Results are reported by vehicle type (combustion, EV), wind level and rider speed band.

## 9. Future work

- A small on-device classifier (Core ML) on log-mel features for overtake vs oncoming
  vs other, and for vehicle type, trained on the labelled dataset.
- Separating several vehicles by tracking multiple harmonic series at once.
- Integration with a rear bike radar, if its protocol is accessible, to fuse the two
  sources and to get continuous ground truth.
- Optional sharing of anonymised, aggregated pass statistics, for example for road
  safety campaigns. This requires a separate privacy design.

## 10. Spikes (do first)

| ID | Question | Method | Decides |
|---|---|---|---|
| S1 | Which mic config has the best traffic-to-wind ratio? Does a windscreen help? | Record every data source plus stereo mode at 15/25/35 km/h on a quiet road with test passes | §3.1 mic mode; whether a windscreen is recommended |
| S2 | Does `.measurement` mode in stereo deliver unprocessed, coherent channels? | Play a known sweep, then inspect inter-channel coherence and frequency response | Whether coherence-based wind suppression is viable |
| S3 | Are tread tones visible and stable enough for EVs? | Recordings of EV passes, inspected in the harness | Weighting of tracker B |
| S4 | Background audio + location stability over 3 h | Long ride on a test device, monitoring gaps and battery | Battery target, interruption handling |

## 11. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Wind noise swamps the signal above about 30 km/h | Few usable passes when riding fast | Coherence mask, mic choice, windscreen accessory, honest confidence and a "too windy" indicator |
| Source pitch changes during the pass | Biased speed | Drift term, A/B tracker agreement, full-curve fit |
| Directional source spectra differ before and after the pass | Tracker B bias | Split references, emphasis on tread tones, A/B cross-check |
| Oncoming or passed vehicles mistaken for overtakes | False passes | Stereo direction cue, loudness-ramp shape, plausibility bounds, later a classifier |
| GPS speed error | 1:1 error in vehicle speed | Kalman filter with motion data, optional BLE speed sensor, error included in σ |
| iOS stops background audio | Missed passes | Interruption handling, gaps logged visibly |
| Hard to get ground truth | Unvalidated accuracy | Controlled sessions, radar reference, simulator |

## 12. Milestones

1. **M0 – Harness**: simulator and Python reference estimator meet the simulated gate.
2. **M1 – Recorder app**: spikes S1–S4 done, and dataset collection under way (with
   controlled-pass sessions).
3. **M2 – Offline accuracy**: the reference pipeline meets the controlled and reference
   gates on recorded data.
4. **M3 – On-device v1**: Swift pipeline running in real time, matching golden vectors.
   Includes the ride log, map, pass detail and export.
5. **M4 – Polish**: live cues, passing distance, EV tuning, calibrated confidence.
6. **M5 – Beta**: TestFlight with a handful of riders and a feedback loop into the
   dataset.
