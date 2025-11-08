# Practical Notes & Performance (FOW)

## Practical Notes
- **Keep instances few:** Merge nearby areas; each Fog Instance has its own RT and full pipeline.
- **Start settings:** `UpdateRate ≈ 0.2 s`, **Hierarchical DDA**, **LOD** on 4, mild **Blur**.
- **Skip occlusion when you can:** If blockers aren’t needed or added, system automatically skips raymarches.
- **Captures:** If you use it, prefer **ShowOnly** + **bUpdateOnlyIfTransformChanged** so you don’t recapture and recalculate the occlusion textures every update.
- **Batch queries:** Try to always use the **batched GPU readback** API, not many single queries.
- **Edge sanity:** Keep **Actor Visibility Cutoff** slightly above **ExploredValue**.
- **If an actor also blocks:** add **EyeOffset** so it doesn’t self-occlude.
- **Use the right raymarch method for your use case** If occlusion is not updated very often, prefer **Distance Field**, otherwise try using **Hierarchical DDA**. If there is a lot of height below the vision sources, which does not occlude, try any of the other ones.

## Cost Hotspots
### GPU
- **Occlusion rebuilds**: Coverage/Height write, **max-mip pyramid**, optionally a **Distance Field** in multiple passes.
- **Visibility pass:** per-pixel loops over vision sources and ray marches if pixel is in its area (reduced by binning them into tiles first), then optional **blur**.

## Important Settings
- **Update cadence:** Try to recalcualte the fog value as rarely as possible, smooth masks allow to turn this value up higher without artifacts (High Blur and Falloff).
- **LOD compute:** reduces the resolution of the internal textures for the shader. Get later on upscaled to the final resolution. Also works best with smooth masks.
- **Blur:** Too high settings can quickly become very costly. However, use it to smooth the mask, primarly on occlusion edges, so that you can use a more coarse LOD level.
- **DFScale (if DF):** **2–4** to shrink DF cost without visible artifacts.

## Choosing a Ray Method

The plugin comes with 4 raymarching methods. The best one depends on your use case. If occlusion is recomputed rarely, prefer **Distance Field**.
If it is updated frequently, prefer **Hierarchical DDA**. However, as **Hierarchical DDA** only does the height test at the finest level, it can quickly become slow if there is a lot of height which does not actually occlude the ray.
If this is the case, try using any of the other methods.


<!--
---


# Profiling & Debugging (RDG Events, GPU Stats & CSV)

The FOW code already emits **RDG events** and **CSV stats**.  Here’s the minimal “how to use it”.

## Unreal Insights (RDG + GPU)
1. **Open**: `Window → Developer Tools → Unreal Insights` → **Launch`.
2. **Record**: press **Start**, move units / rebuild occlusion / run queries, then **Stop**.
3. **Inspect**:
    - Enable the **GPU** and **RDG** tracks.
    - Search `FOW` to jump to passes (e.g., `FOW_BuildTileBins`, `ComputeVisibility`, `Blur H/V`, `MergeMemory`, `ComposeFog`, `UpscaleFog`, `FOW::BuildOccluderMask`, `FOW::CovMip_*`, `FOW::DF *`, `PositionQuery`).
    - Open the **RDG Pass Graph** for pass dependencies & resource lifetimes.

## CSV Profiler (FOW category)
Console:
```txt
csvprofile start file=FOW_Profile.csv categories=FOW
# reproduce your scenario...
csvprofile stop
```
Open the CSV in Insights or a spreadsheet. Useful columns:
`VisionCount`, `OccluderCount`, `CalcPixels`, `FullPixels`, `QueryCount`.

> CLI alternative: `-CSVCategories=FOW -CSV`

## On‑screen quick checks
- GPU timings overlay: ``stat gpu``
- One‑frame deep dive: ``profilegpu``

## Optional legacy GPU stat scopes
If desired, enable your `DECLARE_GPU_STAT_NAMED(...)` and wrap blocks with:
```cpp
RDG_GPU_STAT_SCOPE(GraphBuilder, FOW_Pipeline);
```
(Shows under **stat gpu**.)

## Add your own labels
```cpp
AddPass(GraphBuilder, RDG_EVENT_NAME("FOW::MyPass"), ERDGPassFlags::Compute, …);
CSV_CUSTOM_STAT(FOW, MyCounter, Value, ECsvCustomStatOp::Set);
```
-->