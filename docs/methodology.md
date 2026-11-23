# Methodology

This document describes the math Motion Tracker Lab actually implements — how a click on a
video frame becomes a real-world coordinate, how velocity and acceleration are derived, and how
the curve fits are computed. It reflects the code in [`script.js`](../script.js), not just the
feature list in the [README](../README.md).

## 1. Frame and time model

Each tracked point stores both the browser video's playback timestamp and a corrected
**physics time**:

- **Frame index**: `frame = round(video.currentTime * fps)`
- **Playback time**: `videoTime = video.currentTime` (seconds from the HTML5 `<video>` element)
- **Physics time**: `time = videoTime / timeCorrection`

The student-facing **RECORDED FPS** selector determines both the internal playback frame-step rate and the correction from stretched playback time to real event time. The app uses a 30 FPS playback timeline for slow-motion presets and uses the recorded FPS as the capture rate. For real-time presets, the correction is `1` and frame stepping uses the selected real-time frame rate.

For slow-motion recordings, the correction factor is:

```
timeCorrection = recordedFPS / playbackFPS
physicsTime = videoTime / timeCorrection
```

The presets are 30 FPS real time, 60 FPS HD real time, 60 FPS slow motion, 120 FPS slow motion, and 240 FPS slow motion. The Custom (slow motion) option accepts the actual FPS reported in the phone's video information (for example, 217.35); it assumes a 30 FPS playback timeline. This custom value is a user-supplied average rate, so recordings with variable frame timing may still have small timing deviations.

The correction is applied before calculating velocity, acceleration, position-time graphs, and exported time data. Frame stepping and frame numbering use the playback timeline, not the capture rate.

The original `videoTime` is retained for every tracked point. Changing the slow-motion correction
recalculates the physics time and all derived values for existing points, so students do not need
to retrack the video.

In **sample data mode** there is no video; points are synthetic and indexed directly:
`frame = index`, `time = index / fps`.

## 2. Calibration: pixel → real-world coordinates

Calibration happens in three steps, each building on the last. All raw clicks are stored in
**canvas pixel space** (`px, py`); real-world coordinates are computed on demand by
`pixelToReal()`.

### 2.1 Scale

The user clicks two reference points spanning a known real-world distance. The scale factor is:

```
pixelDistance   = hypot(p2.x - p1.x, p2.y - p1.y)
realDistance    = distance converted to meters (cm/mm → m)
pixelsPerUnit   = pixelDistance / realDistance      // px per meter
```

This is a single **isotropic** scale — the tool assumes no perspective distortion and the same
px/m ratio in both x and y. Camera angle, lens distortion, or motion toward/away from the camera
will bias every downstream measurement.

### 2.2 Origin

The user clicks the point that defines `(0, 0)`. This is stored as a draggable marker in pixel
space and only takes effect once confirmed.

### 2.3 Axes (rotation)

The user clicks a point in the intended `+x` direction from the origin. The axis angle is:

```
angle = atan2(clickY - originY, clickX - originX)
```

This lets the coordinate system be rotated to match a tilted camera or an inclined ramp, so
motion along an incline can be measured directly along its own x-axis rather than the screen's.

### 2.4 Combined transform

For any tracked pixel `(px, py)`:

```
dx = px - originX
dy = py - originY

// Rotate by -angle so the chosen +x direction aligns with the axis
rx =  dx·cos(angle) + dy·sin(angle)
ry = -dx·sin(angle) + dy·cos(angle)

x = rx / pixelsPerUnit
y = ry / pixelsPerUnit

// Physics mode: flip y so it increases upward (screen y increases downward)
if coordMode === 'physics': y = -y
```

`coordMode` is either:
- **Physics** (default) — y increases upward, matching conventional kinematics graphs.
- **Screen** — y increases downward, matching raw pixel coordinates.

Because this transform is recomputed from stored pixel coordinates, editing calibration
(re-setting scale, origin, or axes) after points are tracked automatically recalculates every
point's real-world coordinates (`recalcAllRealCoords()`) without needing to re-track.

## 3. Velocity and acceleration — numerical differentiation

Velocity and acceleration are **not** fit from a smooth curve; they are computed frame-to-frame
by finite differences over the real-world coordinates already produced by calibration.

For a list of tracked points ordered by time, velocity at each point uses:

- **First point** — forward difference: `vx[0] = (x[1] - x[0]) / (t[1] - t[0])`
- **Last point** — backward difference: `vx[n-1] = (x[n-1] - x[n-2]) / (t[n-1] - t[n-2])`
- **Interior points** — central difference: `vx[i] = (x[i+1] - x[i-1]) / (t[i+1] - t[i-1])`

Acceleration is computed the same way, one derivative order up, from the velocity series:

- **First point**: `ax[0] = (vx[1] - vx[0]) / (t[1] - t[0])`
- **Last point**: `ax[n-1] = (vx[n-1] - vx[n-2]) / (t[n-1] - t[n-2])`
- **Interior points**: `ax[i] = (vx[i+1] - vx[i-1]) / (t[i+1] - t[i-1])`

Acceleration requires at least 3 points; velocity requires at least 2. Central differences are
second-order accurate and are used everywhere they're available, which is why interior points are
generally smoother than the first/last point in the velocity and acceleration columns — expect
more noise at the endpoints of a data series.

Because this is a numerical derivative of manually-clicked positions, small clicking errors are
amplified once by differentiation for velocity and twice for acceleration. Uneven frame spacing
(e.g. a frame accidentally skipped) is handled correctly by the difference formulas using actual
`Δt`, but still increases local noise.

## 4. Curve fitting — ordinary least squares

Two fit types are available, both plain OLS regression fit directly to whichever `(x, y)` series
is currently graphed (position, velocity, or acceleration vs. time, or the x–y trajectory):

### 4.1 Linear fit

Standard least-squares line `y = slope·x + intercept`, solved via the normal equations:

```
slope     = (nΣxy - ΣxΣy) / (nΣx² - (Σx)²)
intercept = (Σy - slope·Σx) / n
```

### 4.2 Quadratic fit

Least-squares parabola `y = a·x² + b·x + c`, solved by building the 3×3 normal-equations system
from the power sums (Σx, Σx², Σx³, Σx⁴, Σy, Σxy, Σx²y) and solving it with **Gaussian elimination
with partial pivoting** (`gaussElim3`).

### 4.3 Goodness of fit

Both fits report the coefficient of determination:

```
R² = 1 - SSres / SStot
SSres = Σ(y_i - ŷ_i)²          // residual sum of squares
SStot = Σ(y_i - ȳ)²            // total sum of squares about the mean
```

### 4.4 Physical interpretation shown to the user

The app surfaces the fit coefficients as physics quantities depending on which graph is active:

| Graph | Fit | Interpretation shown |
|---|---|---|
| x(t) | Linear | slope ≈ constant vx |
| vx(t) | Linear | slope ≈ constant ax |
| vy(t) | Linear | slope ≈ constant ay (≈ g if in free fall) |
| y(t) | Quadratic | `2a` ≈ constant ay (≈ g if in free fall) |
| y(x) | Quadratic | parabolic trajectory, consistent with projectile motion |

These interpretations are only physically meaningful if the underlying motion actually matches
the assumed model (e.g. constant acceleration for a quadratic y(t) fit) — the tool does not
validate the physical assumption, it only reports the best-fit curve and its R².

## 5. Data flow summary

```
click (px, py) on canvas
        │
        ▼
video.currentTime ──► divide by slow-mo correction ──► physics time
        │
        ▼
pixelToReal()  ──────────────►  (rx, ry) in meters, using scale + origin + axes
        │
        ▼
App.points[]  { frame, time, videoTime, px, py, rx, ry }
        │
        ▼
calcVelocities() / calcAccelerations()   (finite differences over rx, ry, time)
        │
        ▼
Data table  +  Chart.js scatter plot  +  fitLinear() / fitQuadratic() (OLS)
```

Everything above runs synchronously in the browser on the arrays already held in memory — there
is no server round-trip, which is also why the app can run fully offline once its files (index.html,
script.js, style.css, vendor/) are on disk.
