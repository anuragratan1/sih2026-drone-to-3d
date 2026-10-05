# Failure Cases & Edge Behaviors

Our pipeline was designed to handle imperfect data. We document our known failure cases and our graceful degradation strategies here.

## 1. Malformed Telemetry Logs
**Failure:** Drone flight logs (DJI SRT, GPX, CSV) are notoriously poorly formatted. Drones often drop GPS lock, resulting in variable frame rates, missing timecodes, or zeroed coordinates.
**Mitigation:** The telemetry parser never throws a fatal error. It logs a "what matched" report. If GPS is completely unavailable, the pipeline falls back to an unscaled, unreferenced local coordinate system and outputs a visual mesh anyway.

## 2. Memory Exhaustion (OOM)
**Failure:** The MapAnything attention matrix grows quadratically. If the VRAM probe is slightly optimistic, a chunk can still trigger a PyTorch Out-Of-Memory (OOM) error.
**Mitigation:** The chunk loop catches the OOM and immediately retries the chunk with `minibatch_size=1`. If that fails, it drops `use_multiview_confidence` to free memory and retries again.

## 3. Kernel Kills During Fusion
**Failure:** TSDF fusion on the CPU can consume massive RAM. In early tests, Kaggle's OOM killer terminated the notebook without warning because we checked raw RAM, ignoring the reclaimable page cache.
**Mitigation:** We implemented cgroup-aware available memory calculations (`limit - (usage - inactive_file)`). The TSDF voxel size is dynamically coarsened if the predicted mesh size exceeds free RAM.

## 4. Single-Pass Occlusions
**Failure:** A single straight flight path cannot see the back of buildings or underneath dense tree canopies.
**Observation:** The system accurately reconstructs only what it sees. It does not hallucinate geometry behind structures. We leave gap-filling (via Shape Priors) as future work.
