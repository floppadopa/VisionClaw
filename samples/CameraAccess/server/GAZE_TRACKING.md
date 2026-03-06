# Gaze Tracking Architecture

## Overview
Three-layer hybrid gaze tracking: environment anchors + screen content + optical flow.
Server is the single cursor authority (Kalman filter + 60fps spring interpolation).
iOS only displays a UI indicator; it does not send /move commands.

## Pipeline
```
Camera (iPhone) -> JPEG encode (~5ms) -> WiFi upload (~15ms)
  -> Server decode + optical flow (~5ms on flow-only frames)
  -> Server SuperPoint + matching (~120-300ms on anchor frames, every 5th frame)
  -> Kalman filter update -> set cursor target
  -> 60fps spring interpolation thread -> move_mouse()
```
Flow-only frames: ~20ms round-trip (fast path, 4 out of 5 frames).
Anchor frames: ~150-350ms (feature extraction + matching, every 5th frame or >2s gap).

## Layer 1: Environment Anchors (Calibrated)
- 9 calibration points (3 per display)
- SuperPoint 1024 keypoints per anchor, persisted to calibration.json
- FLANN KD-tree matcher (O(log n) vs BFMatcher O(n), ~3x faster) with 0.75 ratio test
- RANSAC homography (threshold 10.0), project camera center to anchor space
- Top-3 anchor weighted average by inlier count for stability
- Determinant sanity check (0.01 < det < 100.0)
- Weakness: ~300px variance due to rough camera-to-screen scale estimation

## Layer 2: Screen Content (Live)
- Background thread captures per-monitor screenshots every 2s
- SuperPoint features extracted, downscaled to max 1024px
- Retina scale handling for HiDPI displays
- FLANN KD-tree matcher with 0.85 ratio test (more permissive for screen detail)
- RANSAC homography (threshold 5.0, tighter for pixel-level accuracy)
- Median of inlier screen-space points (more robust than camera center projection)
- Preferred over anchors when confident (>= 15 inliers used exclusively)
- Weakness: fails on plain/dark/uniform backgrounds

## Layer 3: Optical Flow (Inter-frame)
- Sparse Lucas-Kanade between consecutive frames
- 200 corners, quality 0.01, min distance 10px, block size 7
- Pyramid: 3 levels, window 21x21
- Median displacement for robustness (rejects outliers from moving objects)
- 1.0px dead zone for noise rejection (camera sensor + JPEG artifacts)
- Full camera-to-screen scale: `(scr_w/cam_w + scr_h/cam_h) / 2 * sqrt(det)`
- 0.6x damping factor to reduce drift from noise accumulation
- Applied directly to Kalman state (not as measurement, as direct state offset)
- Kalman velocity decayed 0.85x per flow frame to resist drift
- Runs every frame for smooth inter-anchor tracking

## Fusion Strategy
- Screen content weighted 3x in hybrid blend with env anchors
- Screen content used exclusively when >= 15 inliers (bypasses anchors)
- Minimum 12 inliers to accept any measurement (outlier gate)
- Kalman filter for optimal smoothing (see below)

## Kalman Filter
State: [x, y, vx, vy] — position + velocity model.

- **Process noise Q**: diag([2, 2, 20, 20]) — low position noise (trusts velocity model), moderate velocity noise (allows direction changes)
- **Base measurement noise R**: 800 — high because raw measurements have ~300px variance
- **Adaptive R**: scales with confidence and match count
  - `noise_scale = max(0.3, 1.0 - confidence) * max(1.0, 50.0 / match_count)`
  - High confidence + many matches -> R = 240 (trusts measurement)
  - Low confidence + few matches -> R = 2000+ (ignores noisy measurement)
- **Predict**: velocity model advances state between frames
- **Flow integration**: optical flow deltas applied directly to state (dampened 0.6x, velocity decayed 0.85x)
- **Measurement update**: standard Kalman gain K = PH'(HPH'+R)^-1

## 60fps Cursor Interpolation (Critically Damped Spring)
The Kalman filter updates at ~6fps (anchor frames) with optical flow corrections between.
Even with Kalman smoothing, the cursor would jump discretely at each update.

A dedicated 60fps thread smoothly drives the actual cursor:

```
Target <- Kalman filter output (updated at ~6-30fps)
Current <- actual cursor position (updated at 60fps)

Each frame (dt = 1/60):
  dx = target.x - current.x
  dy = target.y - current.y
  ax = omega^2 * dx - 2*omega * vx    (spring force - damping force)
  ay = omega^2 * dy - 2*omega * vy
  vx += ax * dt
  vy += ay * dt
  current += v * dt
  move_mouse(current)
```

- **omega = 10**: natural frequency, controls response speed (~250ms to reach target)
- **Critical damping (zeta = 1.0)**: reaches target as fast as possible without oscillation
- **No velocity pulses**: unlike lerp which instantly changes velocity when target shifts, the spring smoothly accelerates/decelerates
- **Dead zone (1.0px)**: stops micro-movements when settled, with velocity damping (0.8x per frame)

### Why Spring > Lerp
Lerp (`next = current + (target - current) * 0.25`) causes visible pulses because:
1. When a new Kalman update shifts the target, the velocity instantly jumps from near-zero to `delta * 0.25 * 60fps`
2. This creates a sawtooth velocity pattern: spike on each update, decay between updates
3. The human eye is very sensitive to acceleration changes (jerk)

The critically damped spring eliminates this because:
1. Velocity changes are governed by a differential equation, not discrete jumps
2. The damping term `-2*omega*v` prevents overshoot
3. The spring term `omega^2*dx` provides smooth acceleration proportional to distance
4. Result: smooth, continuous velocity profile with no perceptible jerk

## Single Cursor Authority
Server is the only thing that calls move_mouse(). This eliminates:
- Double smoothing (server + iOS both smoothing = excessive lag)
- Cursor fighting (server and iOS moving cursor to different positions)
- Extra WiFi round-trip latency (no /move HTTP calls from iOS)

iOS still receives the Kalman position in /locate responses for its UI indicator,
and does its own 60fps lerp purely for display smoothness. But it never sends
cursor movement commands (except during drag mode).

## Smoothing Journey (What We Tried)
1. **Raw positions** -> 300px+ jitter, unusable
2. **400px hard damping** -> reduced range, cursor felt stuck
3. **EMA (alpha=0.35)** -> smooth but laggy, double-smoothed with iOS lerp
4. **Dead zones (80px)** -> helped when still, but masked real movement
5. **Kalman filter (R=200)** -> still 100-150px jitter, R too low
6. **Kalman tuned (R=800, Q=[2,2,20,20])** -> ~50px jitter, much better
7. **60fps lerp interpolation** -> eliminated discrete jumps, but velocity pulses visible
8. **60fps critically damped spring (omega=12)** -> smooth, fast, no pulses
9. **FLANN matcher + 480px resolution + predictive lead** -> ~3x faster matching, lower latency
10. **Corrected scale factor (scr/cam ratio)** -> subtle head movements now register (was ~4x under-scaled)
11. **Flow damping 0.6x + velocity decay 0.85x + omega=10** -> stable without losing responsiveness. Current solution.

## File Layout
- `cursor_server.py` — Main server: Flask endpoints, GazeTracker, GazeKalmanFilter, interpolation
- `calibration.json` — Persisted anchor data (auto-saved/loaded)
- `GAZE_TRACKING.md` — This file
