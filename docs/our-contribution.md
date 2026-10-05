# Technical Contributions: What We Built

To evaluate our submission fairly, it is critical to separate the foundational machine learning backbone (which we adopted) from the extensive engineering, geometry, and pipeline logic (which we built).

Our goal was not to invent a new depth-estimation network from scratch, but rather to construct a robust, production-ready pipeline capable of deploying state-of-the-art AI into a constrained, time-sensitive drone mapping scenario.

---

## What We Adopted (Third-Party Foundations)

- **MapAnything Core Network:** We utilize the Apache-2.0 licensed *MapAnything* network developed by Meta Research and CMU. This provides the fundamental capability to predict feed-forward metric depth and relative camera poses from individual frames.
- **Base Primitives:** We rely on PyTorch for tensor operations, Open3D for the underlying TSDF data structures, and standard Umeyama algorithms for basic Sim(3) mathematical primitives.

## What We Engineered (Our Contributions)

We built the entire surrounding system required to turn a raw drone video into a georeferenced metric mesh within the 15-minute time budget.

### 1. The Video Intake & Keyframe Engine
MapAnything is designed for image sets, not continuous high-resolution video. 
- We built a streaming, hardware-accelerated (NVDEC) video decoder that operates concurrently with model loading.
- We engineered a fast keyframe selector that scores sharpness on down-scaled (480px) grayscale thumbnails (to avoid the I/O bottleneck of full 4K frame extraction), extracting only the sharpest frame per temporal window and rejecting near-duplicates.

### 2. Adaptive Memory & Chunking Architecture
Processing thousands of views simultaneously requires terabytes of VRAM due to attention mechanics.
- We implemented a **run-time VRAM probe** that actively measures the target GPU's free memory and mathematically sizes the inference chunk layout to prevent Out-Of-Memory (OOM) errors.
- We engineered a dense Sim(3) chunk stitching system that aligns adjacent video chunks based on dense 3D point correspondences in the overlapping regions, enabling infinite-length video processing.

### 3. Gravity-Aware Georeferencing
Traditional SfM fits tend to fail catastrophically on straight-line drone flights (producing 27m+ scene tilt errors).
- We built a custom telemetry parser capable of reading legacy and modern DJI subtitles (SRT), CSV, and GPX files, specifically identifying and rejecting documented altitude metadata bugs (e.g., satellite counts mislabeled as height).
- We formulated and implemented a **4-DoF Gravity-Aware Fit**. By determining the scene vertical (gravity) locally and restricting the GPS fit to Translation, Yaw, and Scale, we dropped georeferencing error from 27m down to < 1.1m.
- We implemented a 5-fold hold-out validation loop that automatically generates a metric accuracy certificate.

### 4. Advanced TSDF Fusion & Meshing
Standard TSDF fusion parameters fail on drone footage.
- We built a data-driven voxel sizing algorithm that dynamically derives the voxel resolution from the pixel footprint at the 25th-percentile scene depth.
- We solved severe mesh texture blurring by implementing a custom **Best-View Recoloring** pass, ensuring every mesh vertex is colored exclusively from the single most direct and unoccluded camera view, restoring extreme visual contrast.
- We implemented dual-GPU process-level isolation to allow inference and meshing to occur in parallel without PyTorch/CUDA context deadlocks.
