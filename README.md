# html-water-effect

A real-time WebGPU ocean effect with FFT waves, whitecaps, sun glitter, and an atmospheric sunset sky. Run it from source or use a self-contained HTML file. No rendering engine or runtime dependencies.

Based on [Noniv/web-ocean](https://github.com/Noniv/web-ocean) by Maksymilian Dendura. This version adds a mobile quality profile, adaptive resolution, touch movement and look controls, and a 60 fps submission target. The original desktop quality profile remains available.

![Looking toward the sun from 2.4 m above the water](docs/front.jpg)

| 75 degrees off the sun, 2.4 m up | Looking toward the sun from 30 m up |
|---|---|
| ![](docs/side.jpg) | ![](docs/high.jpg) |

## Run it

Public mobile demo: **https://web-ocean-mobile.pages.dev/**. Cloudflare Pages serves the self-contained build over HTTPS without a login or a running local server.

You need a browser and device with WebGPU support. Touch devices automatically use the mobile profile, which targets 60 fps with a smaller simulation and adaptive render resolution. Desktop devices retain the original full-detail profile. The settings button can override either profile.

Open `ocean.html` directly in a browser that supports WebGPU on `file://`. `ocean.min.html` is the same self-contained page, minified. For a phone, host either page over **HTTPS**. An HTTP LAN URL is not a secure context and cannot run WebGPU; `localhost` only applies to the device running the browser.

To run from source:

```
npm ci
npm start        # serves http://localhost:8080/
```

After editing anything under `js/`, run `npm run build` to regenerate both bundles. The minified build strips WGSL comments and whitespace and minifies the JavaScript with esbuild.

To publish your own Cloudflare Pages demo after rebuilding:

```sh
mkdir -p output/cloudflare-pages
cp ocean.min.html output/cloudflare-pages/index.html
cp LICENSE output/cloudflare-pages/LICENSE
npx wrangler pages deploy output/cloudflare-pages --project-name YOUR_PROJECT_NAME --branch main --commit-dirty=true
```

Replace `YOUR_PROJECT_NAME` with your Pages project name and sign Wrangler into the Cloudflare account that owns it. Only the generated page and license are uploaded. The live demo linked above uses direct uploads rather than automatic GitHub builds.

## Controls

| Input | Action |
|---|---|
| Click the canvas | Capture the mouse (Esc releases it). Holding the right mouse button also works. |
| Mouse | Look around |
| W A S D | Move |
| Space or E / Q or C | Up / down |
| Shift | 5x speed |
| Mouse wheel | Change base speed |
| H | Settings panel: wind, swell, choppiness, turbidity, foam, sun, haze, clouds, exposure, bloom |
| Drag the sea on a touchscreen | Look around |
| Left touch pad | Move forward/back and strafe; can be held while looking |
| Touch arrows | Move up/down; can be held while moving and looking |
| Settings button | Open/close the scrollable settings panel and choose quality |

The camera keeps itself above the displaced water surface, so waves never pass through it.

URL parameters are handy for testing. `?w=2560&h=1440` forces the render resolution, `cam=x,y,z,yawDeg,pitchDeg` sets the camera, `fov=deg` sets the field of view, `ui=0` hides the HUD, and any settings key works too, for example `windSpeed=12&swellHs=2&sunElev=8`.

`?quality=mobile` forces the mobile profile, and `?quality=desktop` forces full detail. Auto uses the primary coarse-pointer media query. `adaptive=0` disables automatic resolution changes; an explicit `w` or `h`, or screenshot mode, also disables adaptation. Mobile rendering is capped at 60 submissions per second on high-refresh displays. Render dimensions are always bounded by the GPU texture limit, and small viewports use a valid bloom mip count.

## Add a scene

Set `globalThis.oceanExtension` before importing `js/main.js`. The async factory receives the WebGPU device, HDR format, MSAA count, shared frame uniform buffer, environment cube view/mip count, mutable parameters, canvas, mobile-profile flag, and `invalidate()` callback.

```js
globalThis.oceanExtension = async (engine) => {
  // Create pipelines and buffers using engine.device.
  return {
    update(dt, camera, waterHeight) {
      // Set camera.x/y/z, yaw, pitch and fovY (radians).
      // Update scene simulation and GPU buffers.
    },
    draw(pass) {
      // Draw into the ocean's HDR pass, after water and before sky.
    },
    get suspended() { return false; }, // true stops GPU submission until resumed
  };
};
await import('./js/main.js');
```

Use `engine.format`, `engine.sampleCount`, and `depth32float` with reverse-Z (`depthCompare: 'greater'`). `engine.frameBuffer` uses the exported `FRAME_WGSL` layout from `js/wgsl/common.js`. Coordinates are metres; subtract the camera position before applying its view-projection matrix. The supplied environment cube provides the same sky lighting as the sea. Call `invalidate()` after changing parameters to rebuild the wave spectrum and atmosphere.

An extension owns the camera; the standalone free-flight camera update and pointer lock are skipped. Keep the shell's HUD/settings/touch element IDs, or provide hidden compatibility nodes as the Titanic integration does. The renderer still initializes those controls. No extension means the original standalone behavior.

## How it works

### Waves

The sea is a Tessendorf FFT simulation with six cascades of 256 by 256 samples. The patches range from 1103 m down to 0.26 m, so wavelengths run from about a kilometre to a few millimetres. The smallest cascade produces the sharp sun sparkles near the camera.

The spectrum is a fetch-limited JONSWAP wind sea with Donelan-Banner directional spreading, plus a narrow long-crested swell. Dispersion includes the capillary term, and choppy horizontal displacement sharpens the crests. The inverse FFTs are radix-2 compute shaders working in workgroup memory.

The surface mesh is a CDLOD quadtree with 17 levels. Vertices are 6.25 cm apart near the camera and the mesh reaches 160 km, with continuous vertex morphing between levels.

### Shading

- Mip-mapped first and second moments of the wave slopes (LEAN mapping) give the right anisotropic roughness at every distance. The sun glitter widens with distance and the far sea settles into a sheen without aliasing. Screen-space slope derivatives add specular anti-aliasing.
- Sun glitter uses an anisotropic Gaussian slope distribution with Smith masking and the size of the solar disc.
- The sky reflection comes from an environment cube map updated one face per frame, with roughness-filtered mips. At grazing angles the reflection direction is tilted to account for which wave facets the viewer can actually see.
- The water body is turbid blue-green, lit by the whole sky hemisphere. Backlit crests get a subsurface glow, and wind-gust patches modulate the short ripples.
- Whitecaps start where the total surface Jacobian folds. Foam is stored in two camera-centred world-space maps (128 m and 1 km wide), decays over time, drifts downwind, and breaks into streaks as it ages. A procedural bubble texture gives it its structure.

### Sky

The atmosphere follows Hillaire's approach, with transmittance, multiple-scattering, and sky-view lookup tables plus aerosol haze. There are two cloud layers, a broken low cumulus and a fibrous altostratus deck. Each is shaded with self-shadowing toward the sun, lit rims, silver linings when backlit, and direction-dependent multiple scattering.

Cloud noise octaves are filtered by pixel footprint, and silhouettes are anti-aliased analytically: the screen-space derivative of each layer's edge function sets the width of its opacity edge. Cloud transmittance along the sun direction dims the sunlight that reaches the water, and aerial perspective is applied over the sea.

### Post-processing

The scene renders in HDR with 4x MSAA. Post-processing adds an 8-level bloom, AgX tone mapping, a slight white balance shift, and dithering. The downsample and additive upsample filters run in one compute pass using separate mip pyramids. The filter arithmetic and level count match the original raster implementation, which remains available with `?bloomPass=raster`. This avoids repeated render-pass stores on tile-based GPUs.

## Performance

### Mobile profile

| Resource | Desktop | Mobile |
|---|---|---|
| FFT | 6 cascades × 256² | 4 cascades × 128² |
| World-space foam | Two 1024² maps, ping-pong | Two 256² maps, ping-pong |
| Surface grid | 32 × 32 quads per node | 16 × 16 quads per node |
| MSAA | 4× | 1× |
| Sky LUT / reflection face | 256 × 144 / 256² | 128 × 72 / 64² |
| Bloom levels | Up to 8 | Up to 5 |
| Resolution | Native device pixel ratio | At most 1.5 DPR and 650,000 pixels |

The mobile profile lowers render scale when submitted frame rate falls below 54 fps or GPU time exceeds 14.5 ms. It recovers resolution slowly when timestamp queries show sustained headroom. Browsers without timestamp queries still reduce resolution using frame cadence. The scale floor is 60% in each dimension. FFT and foam storage fall by about 90%; short capillary waves and some near-surface geometry detail are traded for speed. The atmosphere LUT is reused until settings or camera altitude change. Cloud reflections continue updating round-robin.

The atmosphere LUT retains full profile resolution and refreshes on parameter changes or camera altitude movement (25 cm desktop tolerance, 5 m mobile). Animated cloud reflections continue updating.

The renderer keeps at most two frames in flight and pauses GPU submission while the page is hidden. Touch release, cancellation, focus loss, and backgrounding clear movement state. Device loss and startup failures display a reload/error message.

`window.oceanStats` exposes submitted frame count, FPS, total/compute/render/post GPU times, dimensions, scale, profile, bloom path and camera for runtime measurements. Mobile emulation measures the host GPU; it does **not** prove 60 fps on a physical phone. Device capability, temperature, view direction and browser can change performance.

### Original desktop measurement

At 2560 by 1440 on an RTX 5070 Ti, the GPU takes about 6 to 8 ms per frame, measured while other programs were also using the GPU. The HUD shows FPS and GPU time when the browser supports `timestamp-query`.

The per-frame code reuses preallocated typed arrays and descriptors. Recurring allocations are primarily WebGPU handles, such as the swap-chain view and command buffers, and queue-completion promises used to bound frames in flight.

## Known limitations

- Far water is still somewhat darker than the bright horizon sky above it.
- The water is tuned toward turbid coastal water. The "Water turbidity" slider gives darker, bluer deep water when turned down.
- The standalone scene is open ocean only. Shorelines, surf, and floating objects require a scene extension.
- Clouds are two flat layers, not volumetric.

## Credits

Maksymilian Dendura created the original WebGPU renderer and screenshots in [Noniv/web-ocean](https://github.com/Noniv/web-ocean). This repository preserves its commit history and MIT license. The mobile changes cover quality profiles, touch controls, render resolution, frame pacing, resource reuse, and browser lifecycle handling.

## Repository layout

| Path | Contents |
|---|---|
| `index.html` | Page shell, HUD, and settings panel styles |
| `js/main.js` | Device setup, resources, camera, per-frame loop |
| `js/wgsl/` | Shaders as JavaScript modules: `sim.js` (waves and foam), `sky.js` (atmosphere and clouds), `ocean.js` (surface and sky pass), `post.js` (bloom and tone mapping), `common.js` (shared structs) |
| `server.mjs` | Tiny static server for development |
| `build.mjs` | Builds `ocean.html` and `ocean.min.html` |
| `docs/` | README images |

## License

MIT. Original copyright and permission notice are preserved in [LICENSE](LICENSE) and embedded in both self-contained HTML builds. Keep the notice when copying or redistributing the effect.
