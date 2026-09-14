# Morning Circuit

Single-file web app: a fixed seven-station home workout that counts reps from the
camera feed, plus streaks, history and a daily reminder. No build step, no
dependencies, no server, no network calls at runtime (fonts are the only external
fetch). All data lives in `localStorage`.

**Everything is in `index.html`.** Open it over http(s) — not `file://`, which
blocks camera access.

## The workout

| # | Station | Target | Min gap between reps |
|---|---------|--------|----------------------|
| 1 | Hops (like jumping rope) | 100 | 0.38 s |
| 2 | Arm swings (overhead and down) | 25 | 0.85 s |
| 3 | Body squats | 25 | 1.25 s |
| 4 | Pushups | 25 | 1.15 s |
| 5 | Mountain climbers | 50 | 0.24 s |
| 6 | Jumping jacks | 25 | 0.65 s |
| 7 | Lunges | 25 | 1.20 s |

275 reps total. Morning is the tracked session; evening is optional and logged
separately on the same day.

> **Cadence ceiling.** A station's gap caps its max rate: `hops: 0.38` allows at
> most ~158 reps/min. Jump-rope cadence runs 120-180/min, so a fast hopper gets
> under-counted. If hops read low while the trace ticks look right, lower this
> first. The other six gaps sit well below any sustainable human rate.

## STATUS — read this first

**The rep counting has never been observed working on the owner's devices.** The
camera has not successfully opened on their Mac (Safari/Chrome) or iPhone. Every
other feature works: manual tap counting, station advance, streaks, the 35-day
history grid, stats, reminders, persistence.

A diagnostics panel was added specifically to identify where it fails. It has not
yet been read on a failing device. **That panel's output is the next thing anyone
should look at — do not start by rewriting the detector.**

### Already ruled out — don't redo this work

- Mac has a working FaceTime HD camera; Photo Booth works; no app is holding it.
- The hosted page is HTTPS, top-level (not in an iframe), `isSecureContext === true`.
- `enumerateDevices()` reports 1 videoinput with an empty label, i.e. the browser
  sees the camera and the origin has no grant yet.
- The click handler fires, `getUserMedia` is reached, and its rejection is caught
  and surfaced. Verified by driving the page in a headless browser: it returned
  `NotAllowedError` there (that environment blocks capture by policy), which proves
  the call path is correct.
- On iOS an empty `enumerateDevices()` list before a grant is NORMAL and is not
  evidence of a missing camera. An earlier version of the diagnostic wrongly
  reported "no camera" on iOS and sent the owner chasing macOS settings.

### Bugs already found and fixed (don't reintroduce)

1. `facingMode: "user"` — a phone constraint that a Mac camera can reject outright.
   Removed; the request is now plain `{video: true}`. Frames are downscaled to
   64×48 anyway, so constraints bought nothing.
2. A second `getUserMedia` retry after an `await` — outside the user-gesture
   window, which iOS refuses. There is now exactly ONE `getUserMedia` call site
   (`getCam()`, ~line 429). Keep it that way.
3. `await video.play()` was called while the `<video>` still had `hidden`
   (`display: none`). **iOS Safari will not start playback on a display:none
   video**, so a successfully granted camera fell into the error branch. The
   element is now unhidden before `play()`, and a `play()` rejection is non-fatal.
4. Consequence of (3): a stream that yields no frames could show "Counting" with
   nothing happening. A 3-second watchdog now reports `readyState` and dimensions
   instead.

### Open hypotheses, in order

1. **Browser-level camera default set to Deny.** Safari → Settings → Websites →
   Camera → "When visiting other websites". A global Deny suppresses the prompt on
   every site while leaving Photo Booth working — matches the reported symptom
   exactly. Chrome equivalent: Site settings → Camera.
2. **Per-app TCC grant.** System Settings → Privacy & Security → Camera — Safari
   must be listed and enabled. TCC is per app, so Photo Booth working proves
   nothing about Safari. `tccutil reset Camera com.apple.Safari` forces a fresh
   prompt (the user must run this themselves).
3. **The detector, not the camera.** If `getUserMedia` says `granted` and frames
   climb but reps stay at 0, the thresholds are wrong and the fix is in
   `sampleFrame()`. Compare `energy` against `threshold` in the panel while
   actually exercising.

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

## Deploy

```bash
cd ~/code/morning-circuit && vercel deploy --prod --yes
```

Live: https://morning-circuit-site.vercel.app (Vercel project
`krashnakants-projects/morning-circuit-site`). Note the CDN caches the alias —
verify a deploy with a cache-buster, `curl -s "https://…/?cb=$RANDOM" | grep …`,
or a published change will appear missing.

There is also a private Claude Artifact of the same app at
https://claude.ai/code/artifact/f1d4ef81-2dd4-4e7b-804f-8ac2436cd721 — the same
source with the `<!doctype>`/`<head>`/`<body>` wrapper stripped, since the
Artifact platform supplies it.

## Constraints the owner set

- Everything on-device. No server storage, no cloud AI, no uploads. Camera frames
  are processed in-page and never transmitted.
- Home workouts, bodyweight only.
- Hands-free is the point. Tap-to-count is a fallback, not the feature.
