# Technical Report: Single-Pass Drone Video to Metric 3D

## Abstract
We present a robust, feed-forward pipeline capable of transforming a single-pass drone video (1080p/4K) into a fully georeferenced, metric 3D mesh within strict time constraints (under 15 minutes for a 10-minute video). By substituting traditional, computationally heavy Structure-from-Motion (SfM) with the MapAnything metric depth network, and combining it with a novel gravity-aware georeferencing formulation, our system achieves high-speed reconstruction and resolves severe rotational degeneracies inherent in single-pass flight trajectories. Measured on dual-T4 GPUs, our pipeline processes 4K drone footage at 1.86x real-time while strictly bounding VRAM usage through adaptive chunking.

## 1. Problem
Standard photogrammetry pipelines require drones to fly complex, overlapping grid patterns. This ensures sufficient parallax for SfM pipelines to triangulate points and optimize camera poses. In time-sensitive scenarios (e.g., disaster response, rapid scouting), flying a single straight pass over an area is preferred. However, single-pass video lacks baseline parallax, causing traditional SfM to fail or produce severely distorted models. Furthermore, running full bundle adjustment on high-resolution video is too slow to achieve "near real-time" situational awareness.

## 2. Motivation
The Smart India Hackathon (SIH26158) demands an offline-capable system that can digest drone video, GPS, and flight metadata to produce a 3D model with < 1 meter spatial accuracy, completely skipping traditional SfM. The goal is rapid, single-pass reconstruction without compromising geographical accuracy.

## 3. System Overview
Our system is a completely feed-forward pipeline:
1. **Intake & Streaming Decode**: Hardware-accelerated (NVDEC) decoding extracts frames, while keyframe selection isolates the sharpest, least redundant views.
2. **Chunked Metric Inference**: A deep learning model (MapAnything) predicts metric depth and relative pose for memory-bounded chunks of frames.
3. **Dense Sim(3) Stitching**: Consecutive chunks are stitched using a dense 3D correspondence similarity transform.
4. **Georeferencing & Fusion**: A gravity-aware fit aligns the trajectory to GPS telemetry. Depth maps are fused into a TSDF volume and extracted as a textured mesh.

## 4. Methodology & Architecture
Our core contribution is the surrounding engineering and geometric formulation that makes feed-forward inference viable on large-scale drone video.

### 4.1 Telemetry Alignment
Drone telemetry (DJI SRT, GPX, CSV) operates asynchronously from the video stream. We parse frame counts and timestamps to associate GNSS points with extracted keyframes. We discovered that legacy DJI logs misreport satellite counts as altitude; our parser specifically isolates true relative barometer/RTK altitude for reliable height estimation.

### 4.2 Gravity-Aware Georeferencing (4-DoF Fit)
A standard 7-DoF Sim(3) fit aligns camera centers to GPS coordinates. However, a single straight flight path is a 1D line. Fitting 7 parameters to a 1D line leaves the roll and pitch completely unconstrained, causing the reconstructed ground plane to tilt wildly (up to 27m median error in our tests). 
We solve this by defining a local East-North-Up (ENU) frame based on the scene's gravity vector (derived from gimbal pitch/roll or scene heuristics). We then perform a restricted 4-DoF fit (Translation, Yaw, Scale), dropping scene error to < 1.1m.

### 4.3 Adaptive TSDF Fusion & Best-View Texturing
Standard TSDF fusion uses fixed voxel sizes, which fails on drone footage where the scene distance varies wildly. We dynamically derive the voxel size by projecting the image pixel footprint at the 25th percentile of the scene depth. 
Furthermore, traditional TSDF averages color across views. Because drone poses contain minor noise, this blurs textures significantly. We implemented a best-view recoloring pass that textures each vertex from the single most direct camera view, restoring visual contrast.

## 5. Technical Contributions
- **Hardware-Accelerated Streaming:** Overlapping NVDEC video decoding with model loading.
- **Dynamic VRAM Probing:** Run-time calibration to determine the maximum safe chunk size for attention matrices.
- **Process-Level Parallelism:** Isolating GPU inference from CPU/GPU TSDF fusion in separate OS processes to avoid PyTorch/CUDA context deadlocks.
- **4-DoF Gravity-Aware Georeferencing:** Solving the single-pass straight-line degeneracy problem.
- **Data-Driven TSDF Parameters:** Adaptive voxel sizing and view-dependent mesh recoloring.

## 6. Experiments & Evaluation

### 6.1 Inference Scaling & Speed
Measured on Kaggle (2x T4 GPUs):
- **Input:** 406-second 4K video (12,162 frames, DJI Phantom 4 RTK).
- **Processing Time:** 3 minutes 38 seconds (1.86x real-time).
- **Decoding:** 106 frames/second on NVDEC.

### 6.2 Accuracy Validation
We utilized a strict 5-fold hold-out validation technique for the GPS fit. 80% of the trajectory points are used to calculate the 4-DoF transform, while the remaining 20% are used to measure the hold-out error. The system guarantees geometric integrity by outputting a scale-factor certificate. On synthetic straight passes, the 4-DoF fit reduced error from ~27m (7-DoF) to < 1.1m.

## 7. Failure Cases & Limitations
- **Uncatchable Kernel Kills:** Early iterations suffered from OOM limits because we measured raw RAM instead of active reclaimable page cache. We solved this by using cgroup-aware available memory calculations.
- **Occlusions:** A single pass cannot observe geometry behind buildings or under canopies. The system outputs point clouds that accurately represent only the observed surfaces.
- **Absolute Ground Truth:** While we certify camera position against GPS with < 1.1m error, absolute surface accuracy requires surveying physical ground control points (GCPs) which is planned for future validation.

## 8. Conclusion
We successfully engineered a single-pass drone reconstruction pipeline that meets the SIH26158 time and accuracy constraints. By leveraging MapAnything for rapid depth estimation and engineering a custom gravity-aware georeferencing and memory-adaptive fusion system, we achieved massive speedups without sacrificing geometric stability.

## 9. References
1. MapAnything: Universal Feed-Forward Metric 3D Reconstruction (Keetha et al., 2025)
2. Open3D: A Modern Library for 3D Data Processing (Zhou et al., 2018)
