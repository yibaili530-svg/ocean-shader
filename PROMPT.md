# One-shot prompt: ocean-shader (realistic golden-hour ocean, single HTML file, WebGL2)

This file is a complete prompt that reproduces this repository's `index.html` in a single pass. Every value comes from the current code.

---

Build one file, `index.html`: an infinite golden-hour ocean that reads deep, clear and blue-dominant, with a soft golden sun glitter path and a sky of altocumulus patches. Read the whole spec before writing code. Section 0 is the technical contract and takes priority. Keep every tunable value as a named `const` at the top of `COMMON` so the look can be adjusted in one place.

## 0. Technical contract
- One self-contained HTML file using raw WebGL2 + GLSL ES 3.00. All noise and colours are generated in code, and the file runs offline as-is.
- Pipeline, in this order:
  1. **Sky panorama cache** `panoramaRT`: 2048×512 RGBA16F, equirectangular, with elevation mapped by the power `PANORAMA_ELEVATION_POWER = 2.0` so rows concentrate near the horizon. Update 8 rows per frame in a rolling refresh, and mark it ready after the first full pass.
  2. **Scene pass** `sceneRT`: RGBA16F at canvas size × `renderScale`. A single full-screen fragment shader handles the ocean ray march, water shading and the underwater view; sky pixels sample the panorama cache directly.
  3. **Bloom**: 5 successive half-resolution downsample levels from the scene, then a 3×3 tent upsample (1-2-1 weights) added back level by level.
  4. **Composite**: `(scene + bloom × 0.055) × EXPOSURE(1.05)` → ACES fitted (Stephen Hill matrices) → vignette 0.25 → film grain 0.02 (scaled by the square root of luminance) → sRGB encode → triangular dither of 1/255.
- Float render targets use RGBA16F (`EXT_color_buffer_float`). When the extension is missing, switch to RGBA8 with RGBM encoding (`exposed = col/64`, multiplier stored in alpha) so the image looks the same.
- Units: metres, seconds, radians; +Y is up; sea level is y = 0. Sun direction `SUN_DIR = (0, 0.069756474, -0.99756405)`, i.e. 4° elevation (`SUN_ELEVATION = 0.06981317008`), with its azimuth along −Z, the camera's default view direction.
- Determinism: all animation reads one uniform, `uTime`. Support the URL parameters `?t=` (freezes time, for screenshots) and `?scale= ?x= ?y= ?z= ?yaw= ?pitch=`. Default camera: x=0, y=2.5, z=0, yaw=0, pitch=0.03.
- Sharpness:
  - `renderScale` defaults to 1.0 in manual mode; `devicePixelRatio` is taken as `min(dpr, 1)`.
  - `[` / `]` step the scale by 0.05 within 0.25–1.0 and switch to manual mode.
  - `R` toggles auto resolution: it samples every 0.5 s, lowers the scale by 0.05 when fps < 54 and raises it by 0.03 when fps > 61, within 0.35–1.0.
- Robustness:
  - Check every shader with `getShaderInfoLog` / `getProgramInfoLog`, and show any error in a red on-screen box plus `console.error`. During normal use the console stays clean.
  - Keep every value finite: apply `max(…, 1e-4)` before `pow`, `sqrt` and `normalize`, and route all normalisation through `safeNorm`.
  - Handle `webglcontextlost` / `webglcontextrestored`: after a restore, rebuild programs, textures and render targets, then resume rendering.
  - The render loop reuses preallocated arrays and `Float32Array`s; render targets are rebuilt only when the size changes.
- Expose `window.__ocean = { fps, altitude, renderScale, frames, hdr }` and a global `waterHeight(x, z, t)` for automated checks.

## 1. Waves (geometry): 10 Gerstner waves
- The dominant wavelength is 120 m, and each following component multiplies the wavelength by `WAVE_RATIO = 0.6344941742` (120 m down to about 1.7 m).
- Parameter tables:
  - `AMPLITUDES = (0.60, 0.35, 0.22, 0.16, 0.115, 0.078, 0.053, 0.035, 0.024, 0.016)` m
  - `ANGLES = (0.06, -0.32, 0.49, -0.68, 0.90, -0.16, 0.65, -1.02, 0.27, -0.52)` rad, with direction vector `dir = (sin a, cos a)`
  - `PHASES = (0.4, 2.1, 4.8, 1.3, 5.5, 3.2, 0.7, 4.2, 2.7, 5.9)`
  - `STEEPNESS = (0.16, 0.15, 0.14, 0.13, 0.12, 0.11, 0.10, 0.09, 0.08, 0.07)`
- Phase is `k·dot(dir, q) − sqrt(g·k)·t + phase` with g = 9.81 (deep-water dispersion).
- Eulerian height: invert the horizontal Gerstner displacement with 4 Newton iterations, each correction clamped to ±8 m.
- Fade by pixel footprint: `waveFade(λ, fp) = smoothstep(2fp, 4fp, λ)`. The slope variance faded out this way accumulates into `lost` and feeds the roughness.
- Normal detail: 2 layers of gradient-noise ripples (wavelength from 1.5 m, strengths 0.075 and 0.045), each translating over time and affecting the normals only.
- Surface intersection: up to 32 march steps (`WATER_MARCH_STEPS = 32`) followed by bisection refinement. Every downward ray shades as water, using the last position when the steps run out.
- On the CPU, `waterHeight(x, z, t)` uses exactly the same 10 waves and the same Newton inversion as the GLSL, for keeping the camera on the water and for the HUD altitude.

## 2. Water material (ported from ABYSSAL's ocean shading)
- **Roughness**:
  - Cox–Munk mean-square slope `mss = 0.003 + 0.00512 × 7.0` (wind speed 7 m/s).
  - Pixel footprint `fp = max(dist × pixelAngle, |dFdx(p.xz)|, |dFdy(p.xz)|)`. Split it with `share = (0.06, 0.30, 0.64)` across three wavelength bands of 2.0 m / 1.5 m / 0.3 m: `unresolved = dot(share, 1 − resolved²)`.
  - `mssUnresolved = mss × max(unresolved, 0.10) + 0.0009`; `alpha = clamp(sqrt(2·mssUnresolved), 0.012, 0.62)`; `roughness = clamp(sqrt(alpha), 0.02, 0.86)`.
  - Between 200 m and 1000 m, blend the normal smoothly 30% toward (0, 1, 0).
- **Fresnel** `fresnelWater(NoV, roughness)`: `f = 0.02 + 0.98·(1−NoV)^5`, returning `mix(f, clamp(f·0.72 + 0.06, 0, 1), clamp(roughness·1.5, 0, 1))`.
- **Sky reflection**:
  - Sample the panorama cache with a blur width matched to the GGX lobe (set by alpha), then multiply by (0.59, 0.64, 0.66).
  - Reflection vectors that point downward bend back up: `R = normalize(R.x, mix(0.02, 0.35, roughness) − R.y·0.15, R.z)`.
- **Sky irradiance**: a cosine-weighted sample of the panorama cache, zenith × 0.4142 plus two 45° directions × 0.2929, then multiplied by `BODY_SKY_BALANCE = (3.2, 2.20, 0.83)` to give `bodySky`.
- **Water body colour**:
  - `waterScatter = (0.010, 0.045, 0.092)`, `waterAbsorb = (0.003, 0.016, 0.036)`.
  - `sun = SUN_COLOR(1.0, 0.66, 0.34) × SUN_IRRADIANCE(3.0)`, `sunUp = max(SUN_DIR.y, 0)`.
  - `body = scatter·(sun·sunUp·(1 − fresnelWater(sunUp, 0))/π + bodySky·0.94) + absorb·bodySky·0.8`.
- **Backlit crest glow**:
  - `heightNorm = clamp(h·0.35 + 0.35, 0, 1.5)`.
  - `backlit = heightNorm·pow(sat(dot(L, −V)), 4)·pow(0.5 − 0.5·dot(L, N), 3)`.
  - `body += scatter·sun·backlit·3.4 / (1 + max(0, −L.y)·4)`.
  - Finally, `body *= BODY_GAIN (3.8)`.
- **Sun glitter**:
  - GGX with alpha widened by the solar disc: `aP = clamp(alpha + 0.00465/2, 0, 1)`, energy normalisation `alpha²/aP²`.
  - Smith-GGX correlated visibility, Schlick 0.02 on VoH, then times NoL.
  - `specHDR = SUN_IRRADIANCE·glint·0.5` with a soft cap `SPEC_MAX = 8.0`: `spec = SUN_COLOR²·(8·specHDR / (8 + specHDR))`, so the glitter path reads soft and golden.
- **Combine**: `color = mix(body, reflection, fresnelWater(NoV, roughness)) + spec`.
- **Whitecaps** (sparse and soft):
  - Seed: `breaker = (1 − smoothstep(FOAM_THRESHOLD − 0.13, FOAM_THRESHOLD, J))·smoothstep(0.12, 0.65, h)`, where `FOAM_THRESHOLD = 0.78` and J is the Jacobian.
  - Sample two noise layers in a wind-aligned frame (wind angle 0.06 rad) with coordinates stretched by (0.22, 1.0)×0.8 and drifting at 0.12 m/s, so foam forms long downwind streaks: `raft = sat(0.5 + n)`, `fine = sat(0.5 + n(×2.03 + 7))`.
  - `carved = breaker·(0.10 + (raft·0.65 + fine·0.35)·1.55)`, onset 0.80, `foam = smoothstep(0.80, 1.10, carved)·mix(0.35, 1, fine)`.
  - Distance window: fades in over 12–24 m and out over 140–260 m; it also fades with footprint via `waveFade(3.5, fp)`.
  - Foam colour `(0.93, 0.96, 0.985)·mix(0.62, 1, fine)`, lit by `0.28 + sun·wrapNoL·0.035 + bodySky·0.20`, where `wrapNoL = sat((N·L + 0.45)/1.45)`.
  - A second `foamThin` term (onset × 0.85) lays a translucent film at 20% strength.
- **Aerial perspective**: blend toward the horizon sky colour by `1 − exp(−dist·0.00012)`; clamp the output at 60.

## 3. Sky
- Analytic clear sky `clearSky`: horizon colour `mix(HORIZON_AWAY (0.48, 0.32, 0.47), HORIZON_SUN (1.0, 0.62, 0.32), toward)`, with `toward = pow(sat(dot(horizontal view direction, −Z)·0.5 + 0.5), 4)`. Blend it with the zenith `ZENITH = (0.030, 0.095, 0.420)` using the weight `exp(−y·10.5)`, then add a Mie halo `SUN_COLOR·0.045·HG(μ, 0.76)·exp(−y·1.7)`.
- Sun disc: angular radius `SUN_RADIUS = 0.00471238898`, edge `1 − smoothstep(0.96, 1.04, r)`, limb darkening `0.45 + 0.55·sqrt(1 − r²)`, radiance `SUN_RADIANCE = 60`. Draw the disc analytically in the main pass so it stays crisp.

## 4. Clouds (marched only inside the sky panorama cache)
- The layer spans `CLOUD_BOTTOM = 1500` m to `CLOUD_TOP = 3000` m and drifts along −Z at `CLOUD_WIND = 3` m/s.
- Density:
  - Noise coordinates `q = (p − wind·t)·(0.00115, 0.0018, 0.00115)`.
  - `shape = 0.56 + perlinWorleyFBM(q) − CLOUD_COVERAGE(0.71)`, density `smoothstep(0, 0.12, shape)`.
  - Vertical profile `smoothstep(0, 0.15, h)·(1 − smoothstep(0.48, 1, h))`.
- March: at most 22 km per ray. The primary step count grows from 14 to 32 with elevation, sampling each step at its midpoint with zero jitter, so the result is smooth and grain-free.
- Lighting:
  - 3 light samples toward the sun, 350 m apart; extinction `CLOUD_EXTINCTION = 0.0045`.
  - Brightness from Beer × powder `2(1 − e^{−2d})`; dual-lobe HG (0.8·HG(0.6) + 0.2·HG(−0.2)).
  - Ambient (0.35, 0.42, 0.55), rising from 0.65× at the cloud base to 1.15× at the top.
  - `lighting = ambient + SUN_COLOR·(beer·(0.55 + phase·5)·powder + beer³·phase·0.7)`.
- Horizon transition: between elevations `CLOUD_HAZE_START = 0.025` and `CLOUD_HAZE_END = 0.105` the clouds blend smoothly into the clear sky, and distant clouds dissolve into haze by `1 − exp(−start·0.000018)`.

## 5. Underwater (camera more than 0.15 m below the surface)
- Depth fog:
  - View depth `viewDepth = depth + 30·(1 − smoothstep(0.25, 0.80, rd.y))`.
  - Fog colour runs from (0.05, 0.22, 0.20) to `DEEP_WATER = (0.0, 0.02, 0.05)` at a viewDepth of 30 m.
  - Base brightness is the fog colour × `mix(0.11, 0.85, sat(rd.y·0.6 + 0.4))`.
- Looking up at the surface:
  - Snell's window with the critical angle from n = 1.333 and a ±0.015 smooth edge.
  - Inside the window, show the refracted panorama sky; outside it, total internal reflection: `fog·(0.25 + 0.6·tir.y) + (0.004, 0.012, 0.025)`.
  - Per-channel absorption with distance, `ABSORPTION = (0.45, 0.09, 0.05)`.
- Light shafts: 10 steps along the view ray (`GOD_RAY_STEPS = 10`) with hash jitter, up to 65 m, sampling two warped Voronoi caustic layers along the refracted sun direction with HG g = 0.8. They appear on rays with rd.y from 0.55–0.80 upward.
- Looking down, the colour tends to `DEEP_WATER`. Within ±0.15 m of the surface, the above-water and underwater views blend smoothly.

## 6. Camera, controls, UI
- First-person camera: drag the mouse to look (double-click locks the pointer, Esc releases it); WASD moves; Space / Q rise / dive; Shift gives ×5; the scroll wheel sets the speed within 0.5–50 m/s (default 4). Within 1 m of the surface the camera follows the waves smoothly.
- HUD: 12px white monospace at 70% opacity. Top left: FPS, altitude above the water, `Scale 1.00 manual` or `auto`, and the internal resolution. Bottom left: one or two lines of controls.

## 7. Done criteria
Render three images with headless Chrome (SwiftShader is fine) at 1280×720 with `?scale=1&t=12`: `above.png` (default view), `down.png` (`&pitch=-0.8`) and `under.png` (`&y=-4&pitch=0.7`). Confirm each item:
1. The console is clean, and shader compile and link logs are empty.
2. The lower 35% of `above.png`: B > G > R, G/B < 0.85, mean luminance 0.08–0.35 (target about sRGB 0.13 / 0.23 / 0.37).
3. The top 15% (zenith) is darker and bluer than the band just above the horizon, and that horizon band has R > B.
4. The brightest pixel lies within ±5% of the image width of the centre column (sun and glitter).
5. In `under.png`, the mean luminance of the bottom third is below 0.5× that of the top third.
6. Near the horizon, each horizontal band's variance stays within 3× of its neighbours'.
7. Near-white water pixels (luminance > 0.8 and saturation < 0.15) cover about 0.05%, the glitter reads soft and golden, and the mean colour of `down.png` is about sRGB 0.04 / 0.17 / 0.33.

## 8. Code layout inside the file
In order: `<style>`; the canvas and UI divs; shader strings (`COMMON`, `PANORAMA_FS`, `SCENE_FS`, `BLOOM_DOWN_FS`, `BLOOM_UP_FS`, `COMPOSITE_FS`); then the JS (setup, URL parameters, input, `waterHeight`, resources and context restore, render loop).
