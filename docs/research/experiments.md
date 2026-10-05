# Research & Experiments

During the development of our single-pass reconstruction pipeline, we faced several physical and computational challenges. Below is a structured evidence trail of our core experiments, demonstrating the transition from initial assumptions to validated solutions.

---

## Experiment 1: The TSDF Mesh Texturing Problem

**Question:** Why did the fused TSDF `mesh.ply` look significantly blurrier than the layered point clouds (`scene.glb`), despite sharing the same underlying geometry?
**Hypothesis:** The TSDF volume averages pixel colors across all views. Because drone poses always contain minor estimation noise (even a few centimeters), fine textures get blurred out by projection mismatch.
**Setup:** We measured the visual contrast of synthetic textures across the mesh.
**Input/Data:** A 4K residential drone pass (Hangzhou dataset) with high-frequency roof tile textures.
**Observation:** The ideal synthetic contrast was 0.40. The TSDF output yielded a contrast of only 0.073. Over 80% of fine texture detail was erased.
**What changed afterward:** We implemented `recolor_mesh_from_views`, a post-processing step that skips TSDF color averaging. Instead, every vertex is recolored exclusively from its single best view (the closest and most frontal view passing a depth test).
**Final Conclusion:** Vertex recoloring from the best view restored the synthetic contrast from 0.073 to 0.235, drastically improving visual sharpness without needing computationally expensive texture baking or Gaussian Splatting.

---

## Experiment 2: Dynamic Voxel Sizing for TSDF

**Question:** How large should TSDF voxels be when reconstructing drone footage?
**Hypothesis:** Using a fixed voxel size (e.g., 2 cm up to 25 meters) works well for indoor scanning but fails for drone footage.
**Setup:** We ran TSDF fusion on the Hangzhou RTK 4K sequence using fixed 2cm voxels.
**Observation:** The fusion yielded an empty mesh. The drone was flying above 25 meters, meaning the ground plane entirely escaped the fixed volume. Furthermore, setting a very large, fixed volume quickly caused Out-Of-Memory (OOM) errors.
**What changed afterward:** We implemented a purely data-driven heuristic: the voxel size is derived from the image-pixel footprint on the nearest quarter of the scene. Specifically, `voxel = depth_q25 / focal_px`. 
*(Note: An earlier iteration used median depth / 500, but background sky/mountains heavily skewed the median, leading to 22.6 cm voxels and blobby fragments. The 25th percentile proved stable against sky).*
**Final Conclusion:** Voxel size must be adaptive to the scene's bounding box and camera height, ensuring dense areas get high resolution without running out of RAM.

---

## Experiment 3: Multi-View Consistency for Artifact Removal

**Question:** Feed-forward depth models sometimes predict floating artifacts in featureless areas (like clear skies or featureless water). How can we remove them efficiently?
**Hypothesis:** True surfaces will be observed consistently across multiple overlapping views; hallucinations will disagree geometrically.
**Setup:** We enabled a strict multi-view consistency check (`use_multiview_confidence=True`), dropping any pixel that was not supported by depth predictions in adjacent views during chunk alignment.
**Observation:** Injected synthetic floaters were removed at a 100% success rate.
**Final Conclusion:** Geometric consistency filtering during the Sim(3) chunk stitching phase reliably culls monocular artifacts without needing heavy semantic segmentation models.

---

## Experiment 4: GPU Multiprocessing vs. Multithreading

**Question:** Can we accelerate the pipeline by running TSDF fusion and inference concurrently on a second GPU using Python threads?
**Setup:** We tested concurrent execution on Kaggle's dual-T4 instances. GPU0 ran MapAnything inference, while GPU1 (via a `ThreadPoolExecutor`) fused the completed chunks into a TSDF volume.
**Observation:** The threaded approach caused non-deterministic deadlocks and CUDA illegal memory access violations (`GET was unable to find an engine`). Python's GIL and PyTorch's CUDA context management collided heavily during TSDF `integrate` calls.
**What changed afterward:** We abandoned thread-based parallelism. We moved the second GPU worker to an entirely separate OS process communicating via a file-based job protocol. 
**Final Conclusion:** For safe multi-GPU pipelining in PyTorch/Open3D, process-level isolation is mandatory. Any failure in the worker process now cleanly degrades to sequential processing on the main GPU.
