# Roulette Vision Pro

Roulette Vision Pro is an experimental, client-side wheel and ball tracking
prototype built with OpenCV.js. It processes a live camera feed in the browser,
detects a bright moving ball, estimates wheel motion, converts image movement
into perspective-corrected polar coordinates, and displays live diagnostics.

The application is contained in one HTML file:

```text
roulette-vision-pro-improved.html
```

## Features

- Browser camera capture with rear-camera preference
- OpenCV.js image processing
- Bright-ball detection using contours, circularity, area, and radial gating
- Automatic wheel-track detection using ellipse fitting
- Automatic recalibration when the camera moves or the broadcast view changes
- Three-point perspective calibration fallback
- Ellipse-normalized polar coordinates for oblique camera angles
- Two-dimensional constant-velocity Kalman filter for ball tracking
- Smoothed ball speed and angular velocity
- Optional Lucas–Kanade optical flow for wheel rotation
- Ball-relative-to-wheel angular velocity
- Tracking confidence and state machine
- Debug masks, candidate boxes, trails, and wheel overlays
- Compact HUD mode
- Experimental European-wheel sector display

## Requirements

- A modern browser with camera and WebAssembly support
- HTTPS or `localhost`
- Internet access when loading OpenCV.js from its CDN
- A stable view where most of the wheel track remains visible

Camera access generally will not work when opening the HTML directly with a
`file://` URL. Serve it through a local web server instead.

## Running Locally

From the directory containing the HTML file:

```bash
python3 -m http.server 8080
```

Open:

```text
http://localhost:8080/roulette-vision-pro-improved.html
```

Allow camera access when prompted, wait for `OpenCV ready`, and select
**Start Camera**.

## Calibration

### Automatic calibration

After the camera starts, the application:

1. Converts the frame to grayscale.
2. Blurs and edge-detects the image.
3. Searches for large central contours.
4. Fits ellipses to candidate wheel tracks.
5. Selects the strongest plausible wheel ellipse.
6. Uses its center, major radius, minor radius, and rotation angle as the wheel
   geometry.

This works with oblique camera views because a circular wheel projects as an
ellipse.

### Automatic recalibration

**Auto re-calib** is enabled by default. The application periodically checks
the current camera geometry.

- Small changes are blended into the existing calibration.
- A major camera change must be detected consistently twice before it is
  accepted.
- Confirmed view changes reset the ball tracker and optical-flow state.
- Dealer movement and temporary overlays are less likely to trigger a false
  recalibration.

Disable **Auto re-calib** if the ellipse repeatedly locks onto the wrong table
edge.

### Perspective calibration

Use **Perspective Calib** when automatic detection selects the wrong ellipse.
Click the canvas three times:

1. The center of the wheel.
2. The wheel-track rim along its widest visible axis, usually left or right.
3. The rim along its narrowest visible axis, usually top or bottom.

The two measured radii define the projected wheel ellipse. For a typical
broadcast view, the wide axis is nearly horizontal and the narrow axis is
nearly vertical.

## Perspective-Corrected Coordinates

Raw image-space angles are inaccurate when the wheel is viewed obliquely.
Roulette Vision Pro rotates each detected point into the calibrated ellipse
axes and normalizes it by the ellipse radii:

\[
u = \frac{x'}{r_x}, \qquad v = \frac{y'}{r_y}
\]

It then calculates:

\[
\theta = \operatorname{atan2}(v, u), \qquad
r = \sqrt{u^2 + v^2}
\]

This affine correction maps the projected ellipse back to an approximate unit
circle before angular motion is measured.

## Controls

| Control | Purpose |
|---|---|
| Start Camera | Requests camera permission and begins processing |
| Auto-Calibrate | Runs immediate wheel-track detection |
| Perspective Calib | Starts three-point ellipse calibration |
| Reset Track | Clears ball position, velocity, trail, and filters |
| Show mask | Displays the thresholded ball-detection mask |
| Trail | Displays the filtered ball trajectory |
| Candidates | Displays accepted contour candidates and scores |
| Wheel OF | Enables wheel optical-flow estimation |
| Auto re-calib | Detects and handles camera movement automatically |
| Minimize | Hides controls and secondary diagnostics |

## Detection Settings

| Setting | Description |
|---|---|
| Bright threshold | Minimum grayscale brightness used to isolate the ball |
| Minimum area | Rejects contours that are too small |
| Maximum area | Rejects large reflections and graphics |
| ROI inner | Inner boundary of the elliptical search band |
| ROI outer | Outer boundary of the elliptical search band |

Start by adjusting **Bright threshold** until the debug mask isolates the ball.
Then adjust the area limits to reject reflections and on-screen graphics.

## HUD Metrics

- **Status:** Current camera or processing condition
- **State:** Calibration and tracking state
- **FPS:** Smoothed processing frame rate
- **Confidence:** Current candidate confidence or coasting state
- **Ball r / θ:** Normalized radial position and corrected angle
- **Linear velocity:** Filtered image-space ball velocity
- **Ball ω:** Unwrapped ball angular velocity
- **Wheel ω:** Optical-flow wheel angular velocity
- **Relative ω:** Ball angular velocity minus wheel angular velocity
- **Sector estimate:** Experimental visual sector mapping
- **Wheel:** Calibrated center, ellipse axes, and orientation
- **Candidates:** Number of detected ball candidates

## Tracking States

| State | Meaning |
|---|---|
| NEED_CAMERA | Waiting for camera startup |
| CALIBRATING | Detecting or manually defining wheel geometry |
| SEARCHING | Looking for a valid ball candidate |
| TRACKING | A candidate is being accepted and filtered |
| COASTING | Temporarily predicting without a measurement |
| LOST | Tracking was unavailable for too many frames |

## Troubleshooting

### Bad size of input Mat

Use the latest HTML version. It synchronizes the decoded camera dimensions,
video element dimensions, canvas buffer, and OpenCV input matrix before reading
a frame.

### Memory access out of bounds

Use the latest HTML version. The elliptical masks are preallocated before
OpenCV drawing operations, and temporary contour matrices are released after
use.

If an error remains, note the pipeline stage displayed in parentheses, such as
`ball detection`, `wheel optical flow`, or `automatic perspective calibration`.

### Wrong ellipse is selected

1. Disable **Auto re-calib**.
2. Select **Perspective Calib**.
3. Click center, wide-axis rim, and narrow-axis rim.
4. Confirm that the colored overlay follows the desired ball track.

### Ball is not detected

1. Enable **Show mask** and **Candidates**.
2. Reduce the bright threshold.
3. Expand the ROI outer boundary.
4. Adjust minimum and maximum contour area.
5. Reduce reflections and keep the camera stable.

### Tracking locks onto graphics or reflections

- Increase the bright threshold.
- Reduce maximum area.
- Tighten the elliptical ROI.
- Recalibrate around the ball track rather than the table's outer rim.

### Wheel velocity is unstable

- Ensure the calibrated ellipse follows a textured rotating part of the wheel.
- Disable **Wheel OF** if the broadcast contains frequent overlays or cuts.
- Use relative angular velocity only when optical-flow confidence is visually
  credible.

## Architecture

The processing pipeline is:

```text
Camera frame
  -> frame-size validation
  -> wheel ellipse calibration
  -> elliptical annular ROI
  -> grayscale and brightness threshold
  -> morphological cleanup
  -> contour candidates
  -> candidate scoring and motion gating
  -> Kalman filtering
  -> normalized polar coordinates
  -> wheel optical flow
  -> HUD and debug overlays
```

All processing runs locally in the browser. The application does not upload or
store camera frames.

## Limitations

- Ellipse normalization approximates perspective correction; a full projective
  homography may be more accurate under extreme camera angles.
- Partial occlusion, dealer movement, central hardware, reflections, and
  broadcast overlays can interfere with calibration and tracking.
- Optical flow can follow stationary graphics instead of the rotating wheel.
- Sector labels require a trustworthy zero reference and known wheel
  orientation.
- Browser performance varies considerably across phones and computers.
- The application is a computer-vision experiment, not a validated measurement
  instrument.

## Responsible Use

The sector display is an experimental visual estimate. It does not reliably
predict roulette outcomes and should not be treated as a betting signal or a
guarantee of any result. Follow applicable laws, platform rules, and venue
policies.

