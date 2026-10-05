# High-Level Pipeline

This document describes the step-by-step data flow from a raw drone video to a metric 3D mesh.

## 1. Intake and Screening
When a drone video (1080p or 4K) is provided along with flight metadata (SRT/CSV/GPX):
- The telemetry is parsed and time-aligned with the video frames. Malformed telemetry is handled gracefully without stopping the pipeline.
- **Screening:** The video is analyzed for sky/horizon fraction, shot cuts, overlaid UI elements, and low-light conditions. Sky pixels are masked out to prevent junk depth fusion.

## 2. Keyframe Selection
We do not run inference on every frame (which would waste compute since adjacent frames are nearly identical).
- **Sharpness Filter:** We compute the variance of the Laplacian to pick the sharpest frame within a time window.
- **Near-Duplicate Filter:** We drop redundant frames.
- *Result:* For a 12,000-frame video, we might extract ~120 high-quality, well-separated keyframes.

## 3. Metric Depth and Pose Inference
- The keyframes are passed to the **MapAnything** network.
- Because memory is bounded, keyframes are processed in overlapping chunks (sized automatically by a VRAM probe).
- The network outputs metric depth maps and relative camera poses for each chunk.

## 4. Chunk Stitching
- We extract dense 3D points from the overlapping frames between chunks.
- A **Sim(3) similarity transform** is computed to align the chunks into a single, continuous, metric coordinate system.

## 5. Gravity-Aware Georeferencing
- A typical drone flight is a straight line. Standard 7-DoF alignments to a straight line are unstable (the model can "spin" around the flight path).
- **Vertical Constraint:** We fix the vertical (Z) axis using the ground plane derived from the scene itself (or gimbal pitch/roll if available).
- **4-DoF Fit:** We fit only the remaining parameters (yaw, scale, and XYZ translation) to the GPS path. We use a robust loss and K-fold hold-out to validate accuracy.

## 6. Mesh Fusion
- The aligned depth maps are fused into a global coordinate space using **TSDF (Truncated Signed Distance Function) Fusion**.
- The voxel size is calculated dynamically based on the pixel footprint.
- A confidence mask removes uncertain geometry, and best-view recoloring textures the mesh.

## 7. Export
- The system generates an explicit set of outputs: `mesh.ply`, GLB, colored point cloud, and COLMAP-compatible camera files.
