# MP3 Visualizer

A single-file, zero-build, fully-local browser app: drag MP3 files onto the page,
get a playlist plus 5 audio-reactive WebGL effects driven by the Web Audio API.
No uploads, no server. Just open `index.html`.

## Usage

```
open index.html              # macOS
xdg-open index.html          # Linux
start index.html             # Windows
```

Drop one or more `.mp3` files anywhere on the page (or click "Choose MP3 files"
on the welcome card / the "Drop .mp3 files anywhere" hint in the header).

### Controls

| Action            | Mouse / Click            | Keyboard          |
|-------------------|--------------------------|-------------------|
| Play / pause      | Play button              | `Space`           |
| Seek ±5s          | Scrub bar                | `Left` / `Right`  |
| Prev / next track | Prev / Next buttons      | `Shift+Left/Right`|
| Switch effect     | Effect button (header)   | `E` / `Shift+E`   |
| Fullscreen        | Fullscreen button        | `F`               |

### Toggles (header pills, persist in `localStorage`)

- **Shuffle** — random next track on Next / Prev / song end.
- **Loop** — wrap around at the end of the playlist.
- **Auto FX** — pick a random different effect when each new track starts.

### Per-track

- **♡ / ♥** — like a track (persisted with the track).
- **✕** — remove from playlist.
- Click anywhere on the row to play it.

The mouse cursor and UI panels fade out after ~2.5s of no activity; any
movement or keypress brings them back.

## Effects

| Name         | Description                                                   |
|--------------|---------------------------------------------------------------|
| Nebula       | FBM nebula clouds with a radial spectrum ring                 |
| Star Nest    | Port of Pablo Román Andrioli's iconic Shadertoy fractal       |
| Synthwave    | Retro grid + sun + classic green/yellow/red VU bars; palette cycles |
| Kali         | Kali-set IFS fractal with orbit-trap glow                     |
| Plasma Globe | Ray-marched volumetric plasma globe (after nimitz)            |

All effects react to bass / mids / highs and a beat detector on the bass band.
The current effect is persisted in `localStorage`.

## Persistence

- **Playlist** — Blobs are stored in IndexedDB (`mvDB.tracks`), so the
  playlist (with names, durations, and likes) survives page reloads. Removing a
  track also removes it from IDB.
- **Toggles**, **effect selection**, **liked state** — all in `localStorage` /
  IDB so they round-trip across sessions.

## Implementation

Everything is in `index.html`:

- A 2D fragment shader pipeline driven by an FFT spectrum texture (256 bins,
  uploaded each frame from `AnalyserNode`).
- Per-frame uniforms include `uBass`, `uMid`, `uHi`, `uLevel` (smoothed bands),
  plus `uBeat` (transient envelope), `uBeatT` (seconds since last beat) and
  `uPulse` (a velocity-based clock that surges on beats — used by effects that
  want motion locked to the beat rather than to wall-clock time).

## Credits

- [Star Nest](https://www.shadertoy.com/view/XlfGRj) by Pablo Román Andrioli
  ("Kali") — MIT.
- [Plasma Globe](https://www.shadertoy.com/view/XsjXRm) by nimitz — CC BY-NC-SA.

## License

MIT.
