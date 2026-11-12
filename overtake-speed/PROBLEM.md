# Problem: Measuring the speed of vehicles overtaking a bicycle with a handlebar-mounted iPhone

## Summary

When cycling on roads, the rider is regularly overtaken by cars, vans, lorries and
motorbikes. The speed at which a vehicle passes (and how close it is) largely decides
how dangerous and intimidating the pass is. There is currently no cheap, convenient way
for a cyclist to record how fast each overtaking vehicle was going.

The goal is an iPhone app that, while mounted in a holder on the handlebars during a
normal ride, automatically detects each overtaking vehicle and reports its speed.

Because the phone faces forward/upward on the handlebars, it has no useful view of
traffic approaching from behind. The most promising signal is therefore **sound**: the
**Doppler shift** of the vehicle's noise (engine, exhaust, tyre/road noise) as it
approaches from behind and then recedes ahead of the rider.

## Users and context

- A single cyclist riding on public roads, typically at 15–35 km/h.
- Phone: a recent iPhone in a standard handlebar mount, screen facing the rider.
- Vehicles: cars, vans, buses, lorries, motorbikes; petrol, diesel, hybrid and
  electric; overtaking speeds roughly 30–110 km/h.
- Rides last from minutes to several hours; the phone may be locked with the app in
  the background.
- No extra hardware is required for the core solution (optional add-ons are allowed,
  see below).

## What we want to know (outputs)

For each overtaking event:

| Field | Description |
|---|---|
| Timestamp | When the vehicle passed the rider |
| Location | GPS position at the moment of passing |
| Rider speed | From GPS (or a paired speed sensor) |
| Vehicle speed (ground) | **Primary output** — absolute speed of the vehicle |
| Relative (closing) speed | Vehicle speed minus rider speed |
| Confidence | How trustworthy the estimate is (signal quality, fit error) |
| Audio snippet (optional) | Short clip around the pass, for review and tuning |
| Passing distance (stretch) | Estimated lateral distance of the pass |

Outputs should be available as:

1. A ride log / list of overtakes after the ride (with a map), exportable (CSV/GPX/JSON).
2. Optionally, a live indication or spoken/haptic cue during the ride.
3. Summary statistics: number of passes, speed distribution, fastest passes, and
   passes over a threshold (e.g. > 50 km/h relative speed, or in a 30 km/h zone).

## Physics of the approach

Let `c` be the speed of sound (~343 m/s at 20 °C), `v_b` the rider's speed and `v_c`
the vehicle's speed, both in the same direction. The vehicle emits a sound component of
frequency `f0`.

While the vehicle is **far behind and approaching** (source moving towards observer,
observer moving away from source):

```
f_approach = f0 · (c − v_b) / (c − v_c)
```

Once it is **far ahead and receding**:

```
f_recede = f0 · (c + v_b) / (c + v_c)
```

The unknown source frequency `f0` cancels in the ratio:

```
r = f_approach / f_recede = (c − v_b)(c + v_c) / ((c − v_c)(c + v_b))
```

Solving for the vehicle speed:

```
v_c = c · (r·(c + v_b) − (c − v_b)) / (r·(c + v_b) + (c − v_b))
```

So with `r` measured from audio, `v_b` from GPS and `c` from air temperature, the
vehicle's ground speed can be computed without knowing what frequency the vehicle
actually emits.

Additional information is in the *shape* of the frequency transition: because the
vehicle passes at a lateral offset `d`, the observed frequency follows the radial
component of velocity, `v·cosθ`. The transition is sharp for a close pass and gradual
for a wide one, so fitting a model of the full frequency-vs-time curve can estimate
both relative speed and **passing distance** (`d`), as well as the time of closest
approach.

### Size of the effect

- Closing speed 10 m/s (36 km/h) → each side shifts ~3 %, total ratio ~6 % (about
  one semitone).
- Closing speed 1 km/h ≈ 0.16 % change in `r`. To hit ±5 km/h accuracy we need to
  measure the frequency ratio to roughly ±0.8 %. At 200 Hz that is ±1.6 Hz, which
  requires analysis windows of the order of 0.5–1 s, or sub-bin frequency estimation /
  tracking of harmonics.
- `c` varies ~0.6 m/s per °C; a 10 °C error gives ~2 % speed error — small but worth
  correcting using weather data or a rough temperature input.

## Key difficulties

1. **Wind noise.** A microphone on handlebars at 20–35 km/h receives strong,
   low-frequency, turbulent wind noise, overlapping engine frequencies. This is
   probably the single biggest obstacle. iPhones have several microphones (bottom,
   front/top, rear); choosing/combining them, and possibly a small foam/fur windscreen
   on the mount, may matter.
2. **The bicycle's own noise.** Tyre hum, freewheel ticking, chain, rattles and road
   vibration transmitted through the mount.
3. **Source frequency is not constant.** Engine/exhaust tones change with throttle and
   gear changes, often *during* an overtake (drivers accelerate to pass). The method
   must distinguish Doppler changes from genuine pitch changes, e.g. by using the
   consistency of the shift across multiple harmonics, using tyre noise, or fitting the
   physically constrained `cosθ` curve rather than just comparing start and end.
4. **Electric and hybrid vehicles** emit little tonal engine noise; their sound is
   mostly broadband tyre/road noise. Doppler shift of broadband noise is harder to
   measure (spectral envelope shifts, cross-correlation / time-scaling of spectra).
   Some EVs emit an artificial low-speed sound (AVAS), but usually only below ~20 km/h.
5. **Multiple vehicles.** Platoons of cars, oncoming traffic, vehicles in the
   opposite lane and parked/idling vehicles all contribute sound. Overtakes must be
   segmented and attributed to individual vehicles; oncoming traffic must be rejected
   (it produces a much larger shift, and from the front).
6. **Event detection.** Reliably detecting that an overtake happened (loudness peak +
   downward frequency sweep) without the rider's input, and without false positives
   from the rider's own manoeuvres or environmental noise.
7. **Rider speed accuracy.** GPS speed is noisy, especially in towns and under trees.
   A Bluetooth wheel-speed sensor would help but should be optional.
8. **iOS constraints.** Continuous audio capture and location in the background,
   battery consumption over multi-hour rides, interruptions (phone calls, other audio
   apps, Siri), and microphone permission/privacy. Processing should be on-device; raw
   audio should not be stored or uploaded by default.
9. **Ground truth.** It is hard to know the true speed of a passing vehicle to validate
   the system.

## Requirements

### Must

- Run on an unmodified iPhone mounted on the handlebars, using only built-in sensors.
- Detect overtaking vehicles automatically during a ride.
- Estimate each overtaking vehicle's ground speed and record it with time and location.
- Work with the screen locked / app in background for a full ride.
- Process audio on-device; keep raw audio only if the user opts in.
- Report a confidence value and reject estimates that are clearly unreliable rather
  than reporting a wrong number.

### Should

- Achieve ±5 km/h error (vs. ground truth) on at least 80 % of accepted passes with
  combustion vehicles in moderate wind conditions.
- Detect at least 80 % of overtakes with combustion vehicles, with fewer than 1 false
  overtake per 10 minutes of riding.
- Provide useful (if less accurate) estimates for electric vehicles.
- Export ride data in standard formats.
- Use < 10 % battery per hour of riding.

### Could

- Estimate passing distance.
- Live alerts (audio/haptic) for fast or close passes.
- Pair with a Bluetooth speed/cadence sensor for more accurate rider speed.
- Pair with a rear bike radar (e.g. units that report approaching vehicles and their
  closing speed) to cross-check estimates and act as ground truth during development.
- Classify vehicle type (car, lorry, motorbike, bus) from its sound.
- Store a short audio clip / spectrogram for each pass to support later review.

## Out of scope (for now)

- Identifying individual vehicles or number plates.
- Video capture (the phone faces the wrong way; dedicated rear cameras exist).
- Producing evidence of evidential/legal standard. Results are informational.
- Android support.

## Validation approach

- **Synthetic tests:** simulate passes (moving source with harmonics + broadband
  noise, varying lateral distance, speed, acceleration) mixed with recorded wind and
  bike noise; measure accuracy of the estimator against known truth.
- **Controlled field tests:** a helper drives past at known, steady speeds (cruise
  control / GPS logger in the car) while the rider records; vary rider speed, vehicle
  type and pass distance.
- **Reference sensor:** compare against a rear bike radar's closing-speed readings, or
  a roadside radar gun, during normal rides.
- **Real-ride dataset:** collect labelled recordings (with consent and privacy
  safeguards) across weather, road types and traffic densities.

## Open questions

- Which iPhone microphone(s) give the best signal-to-wind-noise ratio when mounted on
  handlebars, and does a small windscreen make the difference between usable and not?
- Is tonal (engine harmonic) tracking or broadband (tyre noise) analysis more robust
  across the vehicle mix actually encountered?
- How much does driver acceleration during overtakes bias the estimate, and can the
  curve-fitting approach correct for it?
- Can passing distance be estimated well enough from the transition shape alone to be
  useful (e.g. to flag passes closer than 1.5 m)?
- What minimum audio sample rate and analysis window give the required accuracy within
  the battery budget?
