# MP3 Visualizer

A single-file, near-zero-build, fully-local browser app: drop MP3 files onto
the page, get a playlist plus seven audio-reactive WebGL effects driven by the
Web Audio API. Optionally point it at a folder of images and the **Photos**
effect runs them as a beat-aware slideshow. Audio decode and image rendering
all happen client-side; nothing is uploaded.

Just open `index.html`. The page loads [Howler.js](https://howlerjs.com/) and
[Three.js](https://threejs.org/) from a CDN; no build step otherwise.

```sh
open index.html              # macOS
xdg-open index.html          # Linux
start index.html             # Windows
```

## Audio

Drag MP3 files anywhere on the page (or click "Choose MP3 files" on the welcome
card / the "Drop .mp3 files anywhere" hint in the header).

Playback is handled by [Howler.js](https://howlerjs.com/) on the Web Audio
path (`html5: false`). Howler manages the AudioContext lifecycle, mobile / iOS
audio unlock, and recovery from cross-tab audio focus interruptions, so the
app doesn't need to maintain those workarounds itself.

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
- **ⓘ** — open an info modal with the file's stored MP3 metadata: MPEG
  frame header (version, layer, bitrate, sample rate, channels, encoder
  string, frame count, CBR vs. VBR), ID3v2 frames (title / artist / album /
  year / track / genre / BPM / comment …) with embedded APIC cover art if
  present, and the trailing ID3v1 tag. Parsed in-browser from the stored
  Blob — no extra dependencies.
- **✕** — remove from playlist.
- Click anywhere on the row to play.
- **⠿ grip** — drag the handle to reorder tracks.

### Playlist order

- **Sort A–Z** button in the playlist header sorts by name (case-insensitive,
  natural numeric ordering).
- Drag the **⠿** grip on any row to reorder manually.
- The order — whether changed by sort, drag, add, or remove — is persisted to
  IndexedDB (`ord` field per track) and restored on reload.

## Effects

Cycle with the labelled effect button or `E` (`Shift+E` reverses). The active
effect index is persisted in `localStorage`.

| Name          | Description                                                  |
|---------------|--------------------------------------------------------------|
| Nebula        | FBM nebula clouds with a radial spectrum ring                |
| Star Nest     | Port of Pablo Román Andrioli's classic Shadertoy fractal     |
| Synthwave     | Retro grid + sun + green/yellow/red VU bars; palette cycles  |
| Kali          | Kali-set IFS fractal with orbit-trap glow                    |
| Plasma Globe  | Ray-marched volumetric plasma globe (after nimitz)           |
| Trails Stream | Three.js scene: light streams down a tunnel under a Milky Way background |
| Photos        | Slideshow of user-loaded images with animated transitions    |

All shader effects react to bass / mids / highs and a beat detector (transient
detection on the bass band with a velocity-based pulse clock that surges on
each kick). A comb-filter tempo tracker runs over a rolling spectral-flux
buffer to lock onto the dominant beat period and predict beats between hits;
the Trails Stream effect also reacts to a sustained-melody envelope so violin
and other transient-poor content keeps the streams flowing.

### Photos effect

- **Photos…** button in the header opens a directory picker
  (`webkitdirectory`). All `jpg / png / webp / gif / bmp / avif` files in the
  chosen folder are decoded, uploaded as WebGL textures, and persisted to
  IndexedDB.
- Picking a new directory replaces the previous set. Shift-click the button to
  clear all loaded photos.
- Each slide is shown for ~8s with a smooth **Ken Burns auto-pan**: the cover
  window drifts (eased) along a randomized direction, revealing portions that
  were initially cropped off. Movement is bounded by the cover budget so the
  edges of the image never appear — wide images pan horizontally, tall ones
  vertically, and screen-aspect images barely move. Each slide gets a fresh
  pan direction.
- Slide change is a 1.6s transition picking one of six styles at random:
  crossfade, diagonal wipe, chunky pixel dissolve, radial sweep, dual zoom,
  ripple distortion. Beats add a subtle brightness flash.
- Header HUD shows the current image's relative path while Photos is active;
  the previously-loaded directory name is shown on the button label.

## Welcome dialog

On first visit a welcome dialog gives a brief overview. **Don't show this
again** stores an opt-out in `localStorage`; **Skip for now** dismisses it for
the session only. The **ⓘ** button next to the "Spectral" brand reopens it any
time as an about panel (the opt-out checkbox is hidden in that mode). To avoid
a flash before the IndexedDB playlist restore resolves, the dialog is
suppressed synchronously when a saved playlist exists.

## Idle UI

The cursor and UI panels fade out after ~2.5s of no input (mouse, keys, wheel,
touch). Any movement or keypress brings them back. The fade-out is suspended
while dragging a playlist row so the HUD stays visible mid-reorder.

## Persistence

- **Playlist** — Audio Blobs, names, durations, likes, and order (`ord`) live
  in IndexedDB (`mvDB.tracks`). Survives reloads. Metadata updates (like,
  duration, order) are written with a read-modify-write `patch` that preserves
  the stored blob, since rewriting an IDB-backed `Blob` corrupts it on Safari.
- **Photos** — image Blobs and relative paths live in IndexedDB
  (`mvDB.images`). Survives reloads. Last-picked directory name is in
  `localStorage` (`mv.imgDir`).
- **Toggles, effect selection, volume** — `localStorage` keys (`mv.shuf`,
  `mv.loop`, `mv.autofx`, `mv.fx`).
- **Welcome dialog opt-out, playlist hint** — `localStorage` keys
  (`mv.hideWelcome`, `mv.hasPlaylist`).
- DB schema is at version 2; the upgrade path adds the `images` store
  alongside the existing `tracks` store.

## Implementation notes

Everything (apart from the Howler.js and Three.js CDN loads) is in
`index.html` — vanilla JS modules, no build, no other deps.

- A 2D fragment shader pipeline driven by an FFT spectrum texture (256 bins,
  uploaded each frame from a Web Audio `AnalyserNode`). The Trails Stream
  effect is the one exception: it's a Three.js scene with its own
  EffectComposer pipeline (bloom + selective bottom blur) loaded lazily on
  first activation.
- The analyser is spliced between `Howler.masterGain` and the `AudioContext`
  destination once, so the visualizer reads frequency data from whatever
  Howler is playing without each track needing its own wiring.
- Per-frame shader uniforms include `uBass`, `uMid`, `uHi`, `uLevel` (smoothed
  bands), plus `uBeat` (transient envelope), `uBeatT` (seconds since last
  beat) and `uPulse` — a velocity-based clock that surges on beats, used by
  effects that want motion locked to the beat rather than wall-clock time.
- For the Photos effect the shader has additional uniforms `uTexA / uTexB`
  (current and next slide), `uMix / uTransType / uAspectA / uAspectB`, plus
  `uPhotoT / uSlideDur / uKenSeed` driving the auto-pan. The render loop
  binds image textures to `TEXTURE1 / TEXTURE2` and restores `TEXTURE0` for
  the spectrum sampler so the other shaders keep working unchanged.
- Track Blobs are re-fetched from IndexedDB at play time before each Howl is
  created, because Safari/WebKit invalidates IDB-backed `Blob` references
  across page loads.
- Loading the page with `?debug=1` enables verbose `[audio]` console logging
  (AudioContext state changes, recovery, rebuilds); it is silent otherwise.

## Credits

- [Howler.js](https://howlerjs.com/) — MIT.
- [Three.js](https://threejs.org/) — MIT.
- [Star Nest](https://www.shadertoy.com/view/XlfGRj) by Pablo Román Andrioli
  ("Kali") — MIT.
- [Plasma Globe](https://www.shadertoy.com/view/XsjXRm) by nimitz — CC BY-NC-SA.
- The Trails Stream's Milky Way background adapts a community Shadertoy
  (metaDiamond stars + domain-warped fbm nebula).

## License

MIT.
