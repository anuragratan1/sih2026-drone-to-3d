# Performance & Hardware Engineering

To hit the 15-minute budget for a 10-minute video, we had to engineer performance at the hardware limit on Kaggle's dual-T4 GPUs.

## Decoding Bottleneck
**Initial State:** Full-resolution 4K frames were extracted, scored for sharpness using Laplacian variance, and cached.
**Finding:** The full-resolution scoring took massive CPU time and bottlenecked the pipeline.
**Fix:** We downscaled frames to 480px grayscale thumbnails strictly for sharpness scoring. The full 4K frame is only decoded if the frame "wins" its temporal window. We also switched to hardware-accelerated NVDEC decoding.
**Result:** Decoding speed reached 106 frames/second.

## Inference Overlap (Streaming)
**Initial State:** Decoding finished, then the 8GB model loaded into VRAM, then inference started.
**Fix:** We implemented a `STREAMING=True` threaded model. The decoder thread feeds keyframes into a growing queue, while the main thread simultaneously loads the PyTorch model and begins probing VRAM.
**Result:** Overlapped the 29-second model load time entirely with the decode phase.

## Multiprocessing
**Initial State:** Thread-based parallelism for dual GPUs caused PyTorch CUDA context deadlocks.
**Fix:** Switched to a separate OS process (`SecondGPUWorker`) with file-based IPC for the TSDF fusion and chunk handling.
**Result:** Both GPUs can run at maximum utilization without GIL contention or memory access violations.

## Dynamic VRAM Probing
**Finding:** Memory usage scales non-linearly with the number of views in a chunk.
**Fix:** Before processing, the pipeline runs a doubling-search VRAM probe with dummy tensors to measure the exact peak memory on the target hardware. This allows the system to run the absolute maximum chunk size on any GPU (T4, A100, RTX 4090) without ever crashing.
