# The Georeferencing Challenge

Turning a locally consistent mesh into a globally accurate mapping product is the most critical hurdle for single-pass drone reconstruction. This document details the research findings that define our georeferencing approach.

## Finding 1: The DJI Telemetry Legacy Trap

**The Problem:**
While parsing DJI subtitle files (SRT) and GPX tracks, we discovered that standard parsing libraries often wildly misinterpret altitude.

**The Evidence:**
Our analysis of 405 records from a DJI Phantom 4 RTK dataset revealed a fundamental flaw in how legacy tuple structures are read. The legacy tuple `GPS(lon, lat, N)` or `RTK(lon, lat, N)` is often parsed under the assumption that `N` represents altitude.
Our correlation analysis between `N` and the true barometer height field (`H` ranging from 2.0 to 24.8m) showed a correlation of **-0.28**. Furthermore, `N` was always an integer between 24 and 32.
**Conclusion:** `N` is actually the *satellite count/status*, not altitude. Relying on it causes the georeferencing fit to collapse along the Z-axis.

**The Fix:**
Our pipeline now explicitly isolates `rel_alt`, `H`, `BAROMETER`, or `Hb` metadata fields to extract a reliable relative altitude, completely ignoring the deceptive `N` integer.

---

## Finding 2: The Straight-Pass Rotational Degeneracy

**The Problem:**
Traditional Structure-from-Motion (SfM) fits GPS coordinates using a full 7-Degree-of-Freedom (7-DoF) similarity transform (Translation X/Y/Z, Rotation Roll/Pitch/Yaw, and Scale).

**The Evidence:**
When a drone flies a single straight line (a "straight pass"), the camera centers form a 1-dimensional line. A 7-DoF similarity transform fitted to a 1D line is mathematically underconstrained—it can freely "spin" around the flight axis. 
On synthetic testing, this 7-DoF degeneracy caused the entire reconstructed scene to tilt severely, producing a median scene error of **~27 meters** (and up to 267 meters in extreme cases), despite the camera centers perfectly matching the GPS coordinates.

**The Solution:**
We abandoned the 7-DoF fit. Instead, we constrain the Roll and Pitch of the reconstruction *before* fitting to GPS:
1. We determine the **scene gravity vector** (either directly from drone gimbal/attitude telemetry, or by assuming the dominant ground plane is horizontal).
2. We align the local reconstruction so its Z-axis matches gravity (East-North-Up coordinate system).
3. We then perform a restricted **4-DoF fit** against the GPS data, solving only for Yaw, Scale, and Translation (X/Y/Z).

**The Result:**
By eliminating the roll/pitch degeneracy, the absolute scene error on straight passes dropped from ~27 meters to **< 1.1 meters**, unlocking reliable single-pass mapping.

---

## Finding 3: Accuracy Certification via Hold-Out Validation

**The Problem:**
Simply reporting the Root Mean Square Error (RMSE) of the GPS fit is dangerously misleading, as the model will naturally minimize error on the training points, masking systemic scaling or drift issues.

**The Solution:**
Our pipeline incorporates a strict 5-fold hold-out validation loop during georeferencing. We fit the 4-DoF transform on 80% of the trajectory and measure the error on the unseen 20%. 
The system writes an **Accuracy Certificate** (`cameras.json`) alongside every output mesh, explicitly reporting the hold-out error and the computed metric scale factor (which ideally should remain tightly bound near 1.0, proving the feed-forward model successfully predicted true metric depth).
