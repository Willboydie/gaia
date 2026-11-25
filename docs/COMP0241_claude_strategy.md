# COMP0241 Final Project — Strategy Sketch

Two to four viable options per sub-task, a critique of each, and the one I'd pick. One thing shapes most of the choices: the brief says the AO may be swapped for an object that looks different. That favours methods based on shape (it's a sphere) and motion (it spins) over methods tuned to Gaia's colours.

---

## Before you start (applies to every task)

- **Calibrate first.** Get checkerboard intrinsics and undistortion for every camera you'll measure with. Tasks 2c, 4a and 4b all depend on it.
- **Plan one capture session.** The borrowed-sensor deadline is **17:00 on 14/12/2026**, and borrowed-sensor data has to be uploaded within an hour of capture. In that session, record:
  - a bottom view (camera on the floor looking up)
  - a side view with the camera levelled
  - a synchronised two-camera clip for 3f
  - a few static frames with a reference object
  - a LiDAR or depth scan
  - laser-rangefinder or tape readings as ground truth
- **Log your failure cases as you go.** Every question asks for examples of where methods succeed and fail. Keep a short list of hard frames (backlit, motion blur, the wire crossing the object, people in the background) so you don't have to hunt for them at write-up time.

---

## Task 1: Extracting the AO

### 1a. Candidate methods (you need at least two)

| Method | How it works | Good | Weak |
|---|---|---|---|
| **Colour thresholding (HSV or Lab)** | Keep pixels within a distance τ of the object's colour, or *far* from the background's colour | Fast, simple, gives a natural ROC parameter (τ) | Breaks when the AO changes; struggles with ocean/sky lookalikes, white clouds against bright ceilings, shadows. Modelling the **background** colour instead of the object helps with the appearance change |
| **Hough circle transform** | Edges vote for circle centres and radii; draw the best circle as a filled disc | Uses only shape, so it survives appearance changes; outputs a clean disc | Gets confused by other round things (lights, ducts). The disc is off a little under perspective or partial occlusion. Its parameters don't give a smooth ROC |
| **Canny edges + contour + ellipse fit** | Find contours, keep the most circular large one, fit an ellipse | Handles the slight ellipse you get off-axis; easy to filter by circularity | Edges break up on low-contrast limbs; the suspension wire can merge into the contour |
| **Otsu / k-means clustering** | Split pixels into two or three clusters automatically | No hand-picked colours | Assumes the AO and the background really do separate in colour or intensity; often fails on cluttered atrium backgrounds |
| **GrabCut / SAM (pretrained)** | GrabCut is an iterative graph-cut seeded with a box; SAM is a large pretrained segmentation model | Very accurate | GrabCut is slow, and SAM is heavy and needs a citation. Either fits better as a *refinement step* in 1b than as one of your two "methods" |

**Recommended pair:** colour thresholding (appearance-based) plus edge/Hough circle fitting (geometry-based). They fail in different ways, which is exactly what makes combining them worthwhile.

### 1c. The ROC parameter

- **Colour method:** sweep the colour-distance threshold τ. Raising τ lets in more pixels, so true positives and false positives both rise. That smooth, one-directional trade-off is what an ROC curve needs.
- **Geometry method:** the Hough accumulator threshold gives a jumpy ROC with only a few points, because it decides *whether* a circle is detected, not how big the mask is. Two better choices:
  1. **Radius scale** *s*, where the mask is a disc of radius *s·r̂* and *s* runs from 0.5 to 1.5. This maps directly onto under- versus over-segmentation.
  2. **Canny low threshold**, which controls how many edges survive.

  The radius scale is easier to justify, because it is exactly the false-positive/false-negative dial.
- **Pooling:** say whether you add up pixel counts across all images (micro-averaging) or average the per-image rates (macro-averaging). Micro lets large AOs dominate the result.

### 1b. Combining the methods

| Option | Critique |
|---|---|
| **Logical AND / OR** | AND cuts false positives but leaves holes; OR fills holes but adds clutter. Simple, but crude |
| **Cascade: geometry gives the region, colour works inside it** | Robust, and the background clutter disappears. If Hough misses, everything fails, so add a fallback |
| **Weighted soft vote:** score = α·colour_score + (1−α)·circle_score, then threshold at t | Gives you a natural two-parameter grid (α, t) for 1d. Needs both methods to output soft scores (e.g. colour distance, and signed distance from the fitted circle edge) |
| **Seed GrabCut with both masks, then morphological cleanup** | Best-looking masks; slower; harder to tune with a grid search |

**Recommended:** the cascade or the soft vote, then morphological opening and closing and keeping the largest connected component.

### 1d. Parameter search

- **Grid:** for the soft vote, use (α, t). For the cascade, use (colour τ, radius scale *s*).
- **Metric:** Youden's J (true-positive rate minus false-positive rate) is the obvious choice, but it has a flaw here. Background pixels vastly outnumber AO pixels, so the false-positive *rate* stays tiny even when the mask is visibly bloated. **IoU or F1 reflect how good the mask looks much better.** A strong answer plots both and explains why they disagree.
- **Picking the best point:** take the maximum, or a "plateau" criterion (the centre of the region within 1% of the maximum), which is less sensitive to small parameter changes. Argue for the plateau.
- **Avoid tuning and scoring on the same images.** Tune on part of the test set (e.g. Easy) and report on the rest, or cross-validate.
- **Expected discussion:** the grid optimum is fitted to the test set's lighting, so it may lose to your hand-picked values on your own captures. That's fine to show and explain.

---

## Task 2: Centre point and height

### 2a. Geometric centre

| Option | Critique |
|---|---|
| **Image moments of the mask** (centroid = m10/m00, m01/m00) | Trivial to compute, but holes, shadows or the wire in the mask pull it off-centre |
| **Least-squares or RANSAC circle fit to the contour** | Survives a partial or damaged mask, because the outline still lies on a circle. Recommended |
| **Rotation-axis point from motion** (bottom view) | The task title says "projection point of the rotation axis". In the bottom view this is the one point that doesn't move as the object spins. Fit a rotation to the optical flow and its fixed point is the axis. It can differ from the geometric centre if the object hangs tilted, which is good material for discussion |

Note that under perspective the image of a sphere's centre isn't exactly at the centre of the fitted ellipse. The error is small near the image centre, so mention it rather than correct it.

### 2b. Swing over time

- Track the centre in every frame (static camera), then plot x(t), y(t) and the 2D path.
- **Quantify it:**
  - amplitude in pixels, converted to mm using the known diameter or distance (needs calibration)
  - an FFT of x(t) to find the swing period
  - a sanity check against the pendulum formula T = 2π√(L/g), using the wire length L

  In plain terms, a hanging object swings at a rate set by how long its string is, so the measured period tells you whether the motion really is pendulum swing.
- **Bottom vs side view:** the bottom view shows the full 2D swing directly. A side view only sees one horizontal direction.
- **Pitfalls to control:**
  - Camera shake (footsteps): stabilise against background features.
  - Detection noise: film a still period to measure the noise floor, so "it swings by 0.4 px" can be judged against "my detector jitters by 0.5 px".
  - Smoothing with a Kalman filter or exponential moving average (EMA) is optional.

### 2c. Height above the ground

| Option | How it works | Critique |
|---|---|---|
| **A. Floor camera looking up** | A sphere of radius R whose outline spans half-angle α is at distance Z = R / sin α, where tan α = r_px / f. Then height = Z − R + lens height | Elegant and needs only one camera. But it needs R (circular with 4a unless a LiDAR or tape gives it), and the error grows with distance |
| **B. Levelled side camera plus measured horizontal distance d** | Angle above the horizon θ = atan((c_y − v)/f_y); height = h_cam + d·tan θ | Simple. One subtlety: the bottom of the outline is a *tangent* point, not the true lowest point. Use the centre row and subtract R to remove that bias |
| **C. Stereo / two-view triangulation** | Triangulate the centre from two calibrated views, then subtract R | Principled, but needs a good baseline and good calibration; the lowest point itself isn't a feature the two views can match |
| **D. Single-view metrology** | Vanishing lines plus a reference of known height (e.g. a pole near the AO) | Clever but fiddly; the reference has to stand at a similar depth |

**Secondary sensor for validation:**

- **Laser rangefinder:** the simplest and most defensible check.
- **Livox MID-360 / MID-70 LiDAR:** RANSAC-fit a plane to the floor and a sphere to the AO, then height = distance from the sphere's centre to the plane minus R. Gives R for free as well. The scan pattern is sparse, so build up several seconds of points, and remember that swing smears the fitted sphere.
- **Intel D455 / Femto Bolt:** depth error grows roughly with distance squared, so they may be poor at atrium heights; dark or shiny surfaces also hurt ToF.

**Recommended:** A or B as the main method, checked against a LiDAR sphere fit plus a laser reading.

---

## Task 3: Rotation cycle (the biggest share of marks)

The brief lets you build 3e first and reuse it for 3b–3d. Do that: one causal, per-frame ω estimator satisfies c, d and e, provided you also show the offline comparison.

### 3b. Manual estimate

- Pick a distinctive feature and log the frame numbers each time it crosses the centre line, over several full rotations.
- Period = Δframes / fps. Your timing error is about ±1 frame divided by the number of rotations, so measure as many rotations as you can.
- Report the mean and standard deviation per cycle; that answers "does the cycle time vary?".

### 3c. Automatic, offline (whole video available)

| Option | Critique |
|---|---|
| **Self-similarity / autocorrelation.** Crop the AO around its tracked centre (this cancels the swing), compare every frame with frame 0 (normalised cross-correlation), and find the peaks. Parabolic interpolation refines them to sub-frame precision | Works for any texture, so it survives an appearance change. Weaknesses: lighting drift lowers the peaks; a repeating pattern gives false peaks at half the period; a perfectly plain object gives nothing |
| **1D signal + FFT.** Mean intensity of a thin vertical strip through the centre, over time, then pick the dominant frequency | Very cheap. Frequency resolution is 1/(video length), so you need long clips; the strongest FFT peak can be a harmonic (2× or 3× the spin rate) rather than the actual spin rate |
| **Keyframe matching / loop closure.** Count ORB feature matches against the first frame and treat each match peak as one revolution | Intuitive and robust; produces discrete events (one per cycle) rather than a continuous rate |
| **Dense flow (RAFT etc.)** | Accurate but heavy, and excessive for this task |

### 3d. Faster than the frame rate

- Crop to the region around the AO, downscale, convert to greyscale, and run the strip or FFT approach, or the polar-unwrap approach below.
- Per-frame mask computation counts towards the time (pre-processing is included; initialisation isn't).
- Print `FPS: 30 / Processing time: 0.0xx s` for every frame, plus a histogram or percentile table, with visualisation both on and off.

### 3e. Real-time ω per frame (current and past frames only)

| Option | How it works | Critique |
|---|---|---|
| **Bottom view: in-plane rotation** | From below, spinning about the vertical axis looks like the image rotating about the axis point. Track features with Lucas–Kanade and fit a rotation (`estimateAffinePartial2D`) between frame *t−k* and frame *t*; ω = Δθ·fps/k | Clean geometry with little perspective distortion. The features near the pole are few and look foreshortened from below |
| **Polar unwrapping + phase correlation** | Unwrap the disc into polar coordinates around the axis point; rotation becomes a horizontal shift, which FFT phase correlation measures to sub-pixel precision | Very fast and precise; probably the best real-time option for the bottom view. It needs a good axis point, and an off-centre axis smears the shift |
| **Side view: central-meridian speed** | A point at latitude λ and longitude φ projects to x = R cos λ sin φ, so its horizontal image speed is ωR cos λ cos φ. Near the disc centre at the equator, u ≈ ω·R_px, so ω ≈ u/R_px | In plain terms, the middle of the visible face moves sideways fastest, and dividing its pixel speed by the radius in pixels gives the spin rate. Simple, but swing adds to u, so subtract the centre's velocity first |
| **Side view: lift to a 3D sphere** | Back-project tracked features onto a sphere model (centre and radius from Task 2), then solve for the rotation about the vertical axis (Kabsch with the axis constrained) | Uses every feature and is the most rigorous. Features near the edge of the disc are unreliable and should be down-weighted |
| **Sliding-window autocorrelation / sliding DFT** | Run the 3c signal methods over a moving window of past frames | Causal, but it can't give an answer until a full period has passed, and its latency equals one period. That's weaker for the "result per frame" requirement |

**Shared issues:**

- **Slow spin:** if a turn takes about 60 s, it rotates only ~0.2° per frame at 30 fps, so frame-to-frame estimates are noisy. Compare against frame *t−k* (still causal) and smooth ω with a Kalman filter or EMA.
- **Drift:** accumulating many small steps drifts over time. Correct it against a keyframe.

### 3f. Two views at once

**Synchronisation options:**

- **Hardware trigger:** check whether the Arducam global-shutter module has a trigger input. Best accuracy.
- **Both cameras on one PC, logging host timestamps per frame:** easy, accurate to milliseconds, and good enough.
- **A visible event both cameras see** (a flash, or a phone showing a millisecond clock): accurate to about one frame, and works across separate machines.
- **Post-hoc cross-correlation:** align on the **swing** signal, not on ω. Aligning on ω would be circular, because ω is what you're comparing.

**Differences to discuss:**

- The bottom view sees the pole and the rotation appears in-plane: clean geometry, less texture.
- The side view sees the equator: more image motion and texture, but foreshortening near the edges, and the swing gets mixed into the measured motion.
- The period should agree between views. Look at the per-frame noise in ω, any lag from smoothing, and frame-rate mismatches.

---

## Task 4: Landing speed in the Earth's frame

### 4a. Diameter and how uniform the radius is

| Option | Critique |
|---|---|
| **Calibrated camera plus known distance Z** (from the LiDAR or laser): D ≈ 2Z·tan(α) with the sphere correction | Simple. Errors in Z feed directly into D |
| **LiDAR sphere fit** | Direct and in metric units; noisy at mm level |
| **Reference object at the same depth** | Quick. Any depth mismatch gives a scale error |

**Checking uniformity:**

- Fit an ellipse and compare its two axes.
- Better: extract the radius as a function of angle around the centre, r(θ), and show it as a polar plot.
- Over many frames the AO rotates, so stacking r(θ) over time gives a heatmap of radius against angle and longitude.

**Key caveat:** perspective and lens distortion alone make a perfect sphere look elliptical off-axis. Undistort the image and keep the AO near the image centre (or model the effect) before claiming the object itself isn't round.

### 4b. Surface velocity as a function of latitude

- In the AO's own frame: **v(λ) = ω · R · cos λ**. Zero at the poles, maximum at the equator. In plain terms, points far from the axis travel a bigger circle in the same time, so they move faster.
- In the **Earth's frame**, which the task title asks for, add the swing velocity of the centre from 2b: **v = v_centre(t) + ω × r**. This is what the drone actually has to match, and it varies with time as the AO swings.
- If 4a shows the radius isn't uniform, use the measured R(λ).
- **Error propagation:** σ_v/v = √((σ_ω/ω)² + (σ_R/R)²). Quote it, because it shows the marker you understand which measurement dominates the error.
- **Visualise** with a latitude/longitude grid on the sphere and velocity arrows, plus a v-against-λ curve with error bands.

---

## Suggested priority order

1 → 2a/2b → 3e (reused for 3b–3d) → 2c → 3f → 4.
