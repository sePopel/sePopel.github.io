# Performance

## Cost hotspots
- **Update cadence**: how often each layer recomputes (sec). Start with `0.2`.
- **Blur**: radius and iterations. Start with 1 iteration, radius 3–5.
- **LOD compute**: compute can run at lower resolution than the output texture, then upsample.
- **Query readbacks**: always batch AI queries instead of many single reads.

## Recommended defaults
- **Desktop**: cadence `0.15–0.25`, blur (radius 4, iter 1), small LOD downscale for compute.
- **Mid‑range laptops**: cadence `0.25–0.33`, blur (radius 3, iter 1), stronger LOD.
- **Consoles/Low‑end**: cadence `0.3–0.5`, blur (radius 3, iter 1), aggressive LOD.

## Pooling & Memory
The subsystem uses pooled render targets and cleans up on world transitions. You can **ClearMemory** per layer or disable memory entirely for performance tests.

## Profiling
- Use **stat GPU** and **stat unitgraph** to validate cadence/blur choices.
- Toggle features at runtime to identify the biggest wins (Blocker capture vs Circle, blur, cadence).


the smoother the better, bcus many upsampling steps with smoothing possibilities..  allowin to do the compute steps on lower resoluition