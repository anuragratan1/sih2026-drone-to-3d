# System Architecture & Component Map

Our system builds upon the MapAnything foundation by introducing a robust end-to-end pipeline specialized for drone video georeferencing and memory-efficient chunked processing.

## Core Architecture

### 1. Hardware & Process Isolation
- **Dual GPU Strategy:** The system is designed to leverage dual GPUs (e.g., 2x T4 on Kaggle). The second GPU runs in a completely separate process.
- **Why Separate Processes?** Using per-GPU threads causes CUDA illegal-memory-access deadlocks in PyTorch. Our process-based architecture guarantees stability and seamlessly falls back to a single GPU on any failure.
- **Hardware-Adaptive:** Memory use, maximum view count, and TSDF voxel size are dynamically derived from free VRAM and RAM at runtime.

### 2. Component Layers

#### A. Video Processing (Input Layer)
- **Streaming Decode:** Utilizes NVDEC for hardware-accelerated streaming decode at >100 frames/s.
- **Decode-Inference Overlap:** CPU-bound decoding and filtering are heavily overlapped with GPU-bound neural network inference.

#### B. Inference Engine (Core Machine Learning Layer)
- **MapAnything Backbone:** A single-pass feed-forward transformer that predicts per-pixel metric depth and per-frame OpenCV camera poses.
- **VRAM Probing & Chunking:** Since attention memory scales quadratically with view count, we probe VRAM at runtime. The video is processed in manageable overlapping chunks.

#### C. Spatial & Georeferencing Engine (Alignment Layer)
- **Sim(3) Stitching:** Adjacent chunks are stitched together using a similarity transform fitted over dense shared geometry (points observed in overlapping frames).
- **Gravity-Aware Orientation:** Rather than relying on a blind 7-DoF point-cloud fit (which degenerates heavily on straight flight passes), the system estimates the gravity vector directly from the scene (or SRT gimbal attitude) to define the vertical axis.
- **Telemetry Parsing:** A robust parser aligns DJI subtitles (legacy tuples, Mavic 2 style, M300 precision logs) or standard GPX/CSV logs to the video via frame counts and timing lines. 
- **4-DoF GPS Fit:** Using the scene vertical, we fit only Yaw, Scale, and Translation against the GPS path.

#### D. Geometry Fusion (Output Layer)
- **TSDF Fusion:** Depth maps are fused into a single mesh using a Truncated Signed Distance Function (TSDF) implementation.
- **Best-View Recoloring:** The final vertex colors are baked using the sharpest, least-occluded original frame viewing that surface.
