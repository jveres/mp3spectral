# MP3 Visualizer

A single-file, zero-build, fully-local browser app: drop MP3 files onto the
page, get a playlist plus six audio-reactive WebGL effects driven by the Web
Audio API. Optionally point it at a folder of images and the **Photos** effect
runs them as a beat-aware slideshow. Audio decode and image rendering all
happen client-side; nothing is uploaded.

Just open `index.html`.

```sh
open index.html              # macOS
xdg-open index.html          # Linux
start index.html             # Windows
```

## Audio

Drag MP3 files anywhere on the page (or click "Choose MP3 files" on the welcome
card / the "Drop .mp3 files anywhere" hint in the header).

Playback runs through `decodeAudioData` → `AudioBufferSourceNode` rather than a
`<audio>` element. Some VBR MP3s wrapped in `blob:` URLs cause the MediaElement
path to mis-report duration and truncate playback after a few seconds; decoding
to an `AudioBuffer` makes the duration the actual sample count, so playback
can't be cut short.

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
- Click anywhere on the row to play.

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
| Photos        | Slideshow of user-loaded images with animated transitions    |

All shader effects react to bass / mids / highs and a beat detector (transient
detection on the bass band with a velocity-based pulse clock that surges on
each kick).

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

## Idle UI

The cursor and UI panels fade out after ~2.5s of no input (mouse, keys, wheel,
touch). Any movement or keypress brings them back.

## Persistence

- **Playlist** — Audio Blobs, names, durations, and likes live in IndexedDB
  (`mvDB.tracks`). Survives reloads.
- **Photos** — image Blobs and relative paths live in IndexedDB
  (`mvDB.images`). Survives reloads. Last-picked directory name is in
  `localStorage` (`mv.imgDir`).
- **Toggles, effect selection, volume** — `localStorage` keys (`mv.shuf`,
  `mv.loop`, `mv.autofx`, `mv.fx`).
- DB schema is at version 2; the upgrade path adds the `images` store
  alongside the existing `tracks` store.

## Implementation notes

Everything is in `index.html` — vanilla JS modules, no build, no deps.

- A 2D fragment shader pipeline driven by an FFT spectrum texture (256 bins,
  uploaded each frame from `AnalyserNode`).
- Per-frame uniforms include `uBass`, `uMid`, `uHi`, `uLevel` (smoothed
  bands), plus `uBeat` (transient envelope), `uBeatT` (seconds since last
  beat) and `uPulse` — a velocity-based clock that surges on beats, used by
  effects that want motion locked to the beat rather than wall-clock time.
- For the Photos effect the shader has additional uniforms `uTexA / uTexB`
  (current and next slide), `uMix / uTransType / uAspectA / uAspectB`, plus
  `uPhotoT / uSlideDur / uKenSeed` driving the auto-pan. The render loop
  binds image textures to `TEXTURE1 / TEXTURE2` and restores `TEXTURE0` for
  the spectrum sampler so the other shaders keep working unchanged.
- Track Blobs are re-fetched from IndexedDB at play time before
  `decodeAudioData`, because Safari/WebKit invalidates IDB-backed `Blob`
  references across page loads.
- If another browser tab takes audio focus, the AudioContext can stick on
  `'interrupted'` / `'closed'` and `resume()` no longer lifts it. On the next
  user click the player synchronously closes the wedged context and creates
  a fresh one inside the same gesture stack, so the follow-up `resume()` can
  bring it to `'running'` before the next `BufferSource` starts.

## Credits

- [Star Nest](https://www.shadertoy.com/view/XlfGRj) by Pablo Román Andrioli
  ("Kali") — MIT.
- [Plasma Globe](https://www.shadertoy.com/view/XsjXRm) by nimitz — CC BY-NC-SA.

## License

MIT.
