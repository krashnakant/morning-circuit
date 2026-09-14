# Morning Circuit

Single-file web app: a fixed seven-station home workout that counts reps from the
camera feed, plus streaks, history and a daily reminder. No build step, no
dependencies, no server, no network calls at runtime (fonts are the only external
fetch). All data lives in `localStorage`.

> **🔒 100% On-Device & Private — Nothing Stored Anywhere**
> - **Zero server uploads:** Video frames are analyzed live in-memory via HTML5 canvas and immediately discarded. No video, images, audio, or telemetry ever leave your device.
> - **No database / no tracking:** There are no user accounts, no analytics, and no external servers.
> - **Local persistence only:** All streak counters, exercise tallies, and history are stored strictly in your browser's private `localStorage`.

**Everything is in `index.html`.** Open it over http(s) — not `file://`, which
blocks camera access.

## The workout

| # | Station | Target | Strokes / rep | Min gap per stroke |
|---|---------|--------|---------------|-------------------|
| 1 | Hops (like jumping rope) | 100 | 1 (continuous) | 0.32 s |
| 2 | Arm swings (overhead and down) | 25 | 2 (up + down) | 0.42 s |
| 3 | Body squats | 25 | 2 (down + up) | 0.48 s |
| 4 | Pushups | 25 | 2 (down + up) | 0.44 s |
| 5 | Mountain climbers | 50 | 1 (per knee drive) | 0.22 s |
| 6 | Jumping jacks | 25 | 2 (out + in) | 0.34 s |
| 7 | Lunges | 25 | 2 (step + return) | 0.48 s |

275 reps total. Morning is the tracked session; evening is optional and logged
separately on the same day.

> **Cadence ceiling.** A station's gap caps its max rate: `hops: 0.38` allows at
> most ~158 reps/min. Jump-rope cadence runs 120-180/min, so a fast hopper gets
> under-counted. If hops read low while the trace ticks look right, lower this
> first. The other six gaps sit well below any sustainable human rate.

## Current status & features

- **Camera feed & live tracking:** Fully operational. Supports Safari (macOS & iOS) and Chrome/Firefox with unconstrained `{video: true}` stream capture and WebKit `playsinline`.
- **Accurate two-stroke rep detector:** Exercises with two distinct phases (e.g. squat descent + ascent, jumping jack outward + inward) require 2 motion stroke cycles per tallied rep, with an on-screen half-rep indicator (`strokeDot`) and tick trace. Single-stroke exercises (hops, mountain climbers) trigger immediately per cycle.
- **Athletic UI & Theme toggle:** Clean matte dark mode and high-contrast studio light mode, toggleable with the `🌓` header button and automatically synced to system preference.
- **Gym-mode Fullscreen (`F` or `⛶`):** Enlarges the viewfinder to 100vw/100vh with high-visibility tally numbers (`clamp(80px, 14vw, 140px)`), current station badge, and tap-anywhere counting support when propped against a wall.
- **Haptic feedback:** Tactile vibrations on rep completion and station completion on supported mobile devices (`navigator.vibrate`).
- **100% on-device privacy:** Absolutely no frames, images, audio, or telemetry leave the device; all processing occurs in volatile memory and is instantly discarded. All stats and streaks live in local storage.
- **Installable PWA & 100% Offline:** Can be installed locally on phones (iOS & Android) and PCs/Macs as a standalone native app via Service Worker and Web App Manifest (`manifest.webmanifest`).

## Installing locally as a PWA (Phone or PC)

- **iPhone / iPad (Safari):** Tap the **Share** button (📤) → scroll down and tap **Add to Home Screen** (➕) → tap **Add**. It launches full-screen without browser bars.
- **Android (Chrome):** Tap the **⬇ Install** button in the app header, or tap Chrome menu (⋮) → **Install app**.
- **Mac / PC (Chrome / Edge):** Click the **⬇ Install** button in the header, or the install icon in the address bar to install it to your Applications folder / Start menu.
- **Mac (Safari on macOS Sonoma+):** Click **File** → **Add to Dock...** to install it as a standalone macOS desktop app with its own icon in your Dock and Launchpad.

## Reading the diagnostics panel

Press **Diagnostics**, then **Start counting**:

```
getUserMedia  granted | rejected | not called
error         <DOMException name>: <message>
video         readyState=N WxH paused=bool hidden=bool
frames        N  (N/s)
motion        energy=0.00000  band=0.00000
threshold     0.00000  sens=1.3
reps counted  N   station=Hops
page          top-level|embedded  secure=bool
```

- `rejected` → permission problem; the error name identifies the layer. Nothing
  below that line matters.
- `granted` + `frames 0` + `readyState 0` → stream opened, no frames delivered.
- `granted` + frames climbing + `reps counted 0` → detector problem. `energy`
  should visibly swing while moving; if `energy` never exceeds `threshold`, lower
  the sensitivity multiplier or the `0.0018` motion floor.

## How the counting works

No ML model, and one is not possible here: the Artifact CSP blocks the runtime
weight fetches that MediaPipe/TF.js need, and the whole point was on-device
processing with nothing leaving the machine.

1. Each animation frame, the video is drawn to a 64×48 offscreen canvas.
2. Pixels are converted to grayscale and diffed against the previous frame;
   per-pixel differences above 14 are summed into a normalized **motion energy**.
3. Energy is band-passed: `fast` EMA (α 0.34) minus `slow` EMA (α 0.035).
4. An adaptive threshold tracks the noise floor: `max(0.0012, noise × sensitivity)`.
5. One rep is counted per motion cycle — the band signal must dip below
   `-0.35 × threshold` to arm, then rise above `+threshold`, with the station's
   minimum gap enforced as a refractory period.
6. A motion floor (`slow > 0.0018`) stops it counting in a still room.

It reads rhythm, not joint angles. It does not judge form. The trace strip under
the viewfinder draws the band signal with a tick per counted rep — the intended
way to tune sensitivity is to watch the ticks line up with real reps.

Tuning knobs: `STATIONS[].gap` (per-exercise refractory), the sensitivity slider
(0.5–2.6, persisted), and the constants in `sampleFrame()`.

## Data model

`localStorage["morning-circuit.v1"]`:

```jsonc
{
  "days": {
    "2026-09-14": {
      "am": { "counts": {"hops": 100, "squats": 25}, "sec": 412, "done": true },
      "pm": { "counts": {}, "sec": 0, "done": false }
    }
  },
  "prefs": { "sens": 1.3, "sound": true, "reminder": "06:45", "reminderOn": false }
}
```

A day counts toward the streak when `am.done || pm.done`; `done` means all seven
stations reached target. Storage is per-origin, so the artifact copy, localhost
and the Vercel URL each keep separate histories. "Copy backup" copies the JSON to
the clipboard — **there is no restore path yet**; that is a known gap and an easy
first task.

## Deployments

- **GitHub Pages:** https://krashnakant.github.io/morning-circuit/
- **Vercel Production:** https://morning-circuit.vercel.app (and https://morning-circuit-site.vercel.app)
- **GitHub Repository:** https://github.com/krashnakant/morning-circuit

Deploying to Vercel:
```bash
cd ~/code/morning-circuit && vercel deploy --prod --yes
```

Deploying to GitHub Pages:
```bash
git push origin main
```

## Constraints the owner set

- Everything on-device. No server storage, no cloud AI, no uploads. Camera frames
  are processed in-page and never transmitted.
- Home workouts, bodyweight only.
- Hands-free is the point. Tap-to-count is a fallback, not the feature.
