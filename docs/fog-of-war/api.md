
# Fog of War (UE5) — API Reference

This page documents the **public API** of the Fog of War plugin as exposed by the **Subsystem**, **Actor Components**, and the **Manager**. It’s written for **Blueprint** and **C++** users.  

---

## Concepts & Data Types

- **Fog Instance**: A named fog layer (render target + runtime settings). Multiple instances are supported and updated in order.
- **Revealers**: Vision sources that write visibility (smooth circles) into the fog.
- **Blockers / Occluders**: Circular blockers and/or depth-capture–driven occlusion; optional height-aware checks and distance field.
- **Stealth**: Policy that hides, fades, or “freezes as ghost” when an actor is not visible in fog.

**Key structs / enums (Blueprint-exposed):**

- `FFOW_AreaSettings` – Center (`FVector`), FullExtents (`FVector2D`), YawDeg (`float`).
- `FFOW_RuntimeSettings` – Toggles (Enabled/Freeze/ShowFog/ShowMemory/FreezeMemory), Visual (ExploredValue, ActorVisibilityThreshold, SmoothFalloff, Blur settings), Blending (Transition, FadeInRate, RelaxRate, ForgetRate), Occlusion (RayMode, DFScale, UseHeightOcclusion, Capture options), Performance (LOD, UpdateRate, TileSize/MaxPerTile, UpdateOnlyOnChanges).
- `FVisionData` – WorldPos, RangeWS, EyeHeightOffsetWS.
- `EFOWActivationPolicy`, `EFOWVisibilityPolicy`, `EFOWHiddenMode`, `EFOWOcclRayMode`, `EFOW_TransitionBehaviour`.

---

## Subsystem (`UFOWSubsystem`)

Access via:

```cpp
auto* FOW = GetWorld()->GetSubsystem<UFOWSubsystem>();
```

The subsystem (client-side) owns all **Fog Instances** and dispatches GPU work each update.

### Lifecycle & Materials

- `void RestartFogSystem()`: Clears memory/registries, detaches delegates, and rebuilds all state.
- `void ClearAllMemory()`: Clears “last seen” memory for **all** instances.
- `bool ClearMemory(FName FogInstance = NAME_None)`: Per-instance memory clear (`NAME_None` = primary).
- `UMaterialInstanceDynamic* AcquireSharedMIDForFogInstance(FName FogInstance, UMaterialInterface* Base)`: Returns a **shared, auto-updating MID** with fog parameters & RT bound.

### Fog Instance Management

- `FName GetPrimaryFogInstanceName() const`
- `void GetFogInstancesNames(TArray<FName>& Out)`
- `bool AddOrReplaceFogInstance(const FFOW_FogConfig& Config)`
- `bool RemoveFogInstanceByName(FName Name)`
- `void AddProfile(const UFOWProfile* Profile)` / `void RemoveProfile(const UFOWProfile* Profile)`
- `bool GetFogInstanceRuntime(FName, FFOW_RuntimeSettings& Out) const`
- `bool GetFogInstanceArea(FName, FFOW_AreaSettings& Out) const`

### Runtime Settings

_All setters return `bool` and accept `NAME_None` to target the **Primary** instance._

**Toggles**
- `SetFogInstanceEnabled(FName, bool)`
- `SetFogInstanceFreezeState(FName, bool)`
- `SetFogInstanceShowFog(FName, bool)`
- `SetFogInstanceShowMemory(FName, bool)`
- `SetFogInstanceFreezeMemory(FName, bool)`

**Visuals**
- `SetFogInstanceExploredValue(FName, float /*0..1*/)`
- `SetFogInstanceActorVisibilityThreshold(FName, float /*0..1*/)`
- `SetFogInstanceSmoothFalloff(FName, float /*0..1*/)`
- `SetFogInstanceBlurIterations(FName, int32)`
- `SetFogInstanceBlurRadius(FName, int32)`
- `SetFogInstanceBlurSigma(FName, float)`

**Blending**
- `SetFogInstanceTransitionMode(FName, EFOW_TransitionBehaviour)`
- `SetFogInstanceFadeInRate(FName, float)`
- `SetFogInstanceRelaxRate(FName, float)`
- `SetFogInstanceForgetRate(FName, float)`

**Occlusion**
- `SetFogInstanceOcclusionRayMode(FName, EFOWOcclRayMode)`
- `SetFogInstanceDFScale(FName, int32 /*>=1*/)`
- `SetFogInstanceUseHeightOcclusion(FName, bool)`
- `SetFogInstanceUseDepthCaptureOcclusion(FName, bool)`
- `SetFogInstanceCaptureUsesShowOnly(FName, bool)`
- `SetFogInstanceUpdateOnlyIfTransformChanged(FName, bool)`

**Performance**
- `SetFogInstanceLOD(FName, int32 /*1..32*/)`
- `SetFogInstanceUpdateOnlyAtCrucialChanges(FName, bool)`
- `SetFogInstanceTileSize(FName, int32)`
- `SetFogInstanceMaxPerTile(FName, int32)`
- `SetFogInstanceUpdateRate(FName, float /*seconds, >= 0.01*/)`
- `void SetStealthActorUpdateRateSeconds(float Seconds)`

**Bulk**
- `bool SetFogInstanceRuntime(FName, const FFOW_RuntimeSettings&)`

### Area & Transform

- `bool SetFogInstanceArea(FName, const FVector& Center, const FVector2D& FullExtents, float YawDeg)`
- `bool ExpandFogInstanceAreaToInclude(FName, const FVector& OtherCenter, const FVector2D& OtherFullExtents, float OtherYawDeg /*keeps A’s yaw*/ )`

### Vision Sources (Revealers)

**Actor-based**
- `void RegisterVisionSource(AActor* Source, float RangeWS, float EyeHeightOffsetWS = 0.f)`
- `void UnRegisterVisionSource(AActor* Source)`
- `void ClearAllVisionSources()`

**Static (handles)**
- `int32 AddStaticRevealer(FVector WorldPos, float RangeWS)`
- `void RemoveStaticRevealer(int32 Handle)`
- `bool SetStaticRevealerRange(int32 Handle, float NewRangeWS)`
- `void ClearAllStaticRevealers()`

> The GPU path bins revealers into tiles and draws a smooth circle per source. Optional blur + memory/composition run afterward.

### Stealth Actors

- `void RegisterStealthActor(AActor* Actor, EFOWHiddenMode HideMode, EFOWVisibilityPolicy Policy)`
- `void SetStealthVisibilityPolicy(AActor*, EFOWVisibilityPolicy)`
- `void SetStealthHiddenMode(AActor*, EFOWHiddenMode)`
- `void SetStealthActorHideThresholdOverride(AActor*, float ThresholdOrNegative /*<0 = system default*/ )`

### Position Queries (GPU)

- **Handle types**: `FFOWPositionQueryHandleBP` (Blueprint) / `FFOWPositionQueryHandle` (C++)
- **What it does**: Samples the **composed fog** at N world positions and returns per-position fog values (0..1) via non-blocking GPU readback.

---

## Actor Components

### `UFOWVisionComponent`

Attach to reveal fog around the actor.

**Properties**
- `Activation` (Manual / AutoOnBeginPlay)
- `VisionRange` (cm)
- `EyeHeightOffset` (for height-aware occlusion)

**Blueprint API**
- `ActivateForFOW()`, `DeactivateForFOW()`
- `SetVisionRange(float)`, `GetVisionRange()`
- `SetEyeHeightOffset(float)`
- `IsRevealing()`

> On activation it registers with the subsystem; changing range/offset while active re-registers to apply changes.

---

### `UFOWBlockerComponent`

Attach to provide occlusion either as simple **circles** or via **depth capture** writes into one or many fog instances.

**Modes**
- `Circle` (radius)
- `CaptureWriter` (SceneCapture-driven occlusion input)

**Properties**
- `Activation`
- `BlockerRadius` (Circle)
- `bAllFogInstances`
- `TargetFogInstances` (CaptureWriter)

**Blueprint API**
- `ActivateForFOW()`, `DeactivateForFOW()`
- `SetMode(EFOWBlockerMode)`
- `SetBlockerRadius(float)`
- `AddToAllFogInstancesCapture()`, `RemoveFromAllFogInstancesCapture()`
- `AddToFogInstanceCapture(FName)`, `RemoveFromFogInstanceCapture(FName)`

---

### `UFOWStealthComponent`

Attach to actors that should hide/fade/freeze when not visible in fog. Handles **material fades** and optional **ghost** sync.

**Properties**
- `Activation`
- `HiddenMode` (`Hide`, `FreezeWithGhost`)
- `VisibilityPolicy` (`FogDriven`, `AlwaysVisible`, `AlwaysHidden`)
- `bUseStealthVisibilityThresholdOverride` + `StealthVisibilityThresholdOverride` (0..1)
- `FadeSeconds`, `FadeParamName`, `bHideActorAfterFadeOut`

**Blueprint API**
- `ActivateForFOW()`, `DeactivateForFOW()`
- `SetHiddenMode(EFOWHiddenMode)`
- `SetVisibilityPolicy(EFOWVisibilityPolicy)`
- `SetStealthVisibilityThresholdOverride(bool bEnable, float Threshold01)`
- `SetFadeSeconds(float)`
- `SetFadeParam(FName)`
- **Events**: `OnBecameVisible`, `OnBecameHidden`

---

## Manager (`AFOWManager`)

A convenience actor that:
- Spawns **visual variants** (Fogged Mesh slices, Post Process, Volumetric Box, Light Function).
- **Adopts** components/lights from itself and other actors.
- Can **push area updates** into the subsystem.
- Adds/removes **profiles** at runtime.

**Properties (high level)**
- `TArray<UFOWProfile*> Profiles` – added/removed on BeginPlay/changes.
- `TArray<FFogVariant> Variants` – each targets a Fog Instance, can union/overwrite area, build visuals.

**API (high level)**
- `void RebuildFogVisuals()`
- Adoption:
    - Components: `AdoptComponent(UActorComponent*)`, `UnadoptComponent(UActorComponent*)`, `ClearAdoptedComponents()`
    - Lights: `AdoptLight(ULightComponent*)`, `UnadoptLight(ULightComponent*)`, `ClearAdoptedLights()`

**Variant types (examples)**
- **Fogged Meshes**: vertical slice stacks per mesh/instance; binds fog MID.
- **Post Process**: PP blendable with weight/priority/unbounded.
- **Volumetric Box**: proxy mesh + material; configurable half extents.
- **Light Function**: applies fog light-function material to adopted lights.

---

## Project Settings & Profiles

### Project Settings → Plugins → Fog Of War (`UFOWSettings`)
- **DefaultFogInstances** — initial fog instances (one internal allowed; others external).
- **GlobalFogDefaultUpdateRate** — cadence fallback.
- **GlobalStealthActorUpdateRate** — stealth polling rate.
- Default materials for Manager variants (Fogged Mesh / Volumetric Box / Post Process / Light Function).

### Profiles (`UFOWProfile`)
- Data assets with **external-only** Fog Instances.
- Add/remove via Subsystem or Manager at runtime.
- Names are sanitized (unique, non-empty).

---
## Examples

### Blueprint quick starts

```text
// Reveal an actor on BeginPlay
Add Component: FOWVisionComponent
  Activation = AutoOnBeginPlay
  VisionRange = 2500
```

```text
// Hide an enemy when not visible
Add Component: FOWStealthComponent
  HiddenMode = Hide
  VisibilityPolicy = FogDriven
  FadeSeconds = 0.25
  FadeParamName = "FOW_Visibility"
```

```text
// Use a blocker that writes into depth captures of all instances
Add Component: FOWBlockerComponent
  Mode = CaptureWriter
  bAllFogInstances = true
```

### C++ snippets

```cpp
// Toggle memory on the primary instance
if (auto* FOW = GetWorld()->GetSubsystem<UFOWSubsystem>())
{
    FOW->SetFogInstanceShowMemory(NAME_None, true);
}
```

```cpp
// Bind a MID with fog params and RT to a mesh
if (auto* FOW = GetWorld()->GetSubsystem<UFOWSubsystem>())
{
    UMaterialInstanceDynamic* MID = FOW->AcquireSharedMIDForFogInstance(NAME_None, BaseMat);
    Mesh->SetMaterial(0, MID);
}
```

```cpp
// Register a static revealer
if (auto* FOW = GetWorld()->GetSubsystem<UFOWSubsystem>())
{
    int32 Handle = FOW->AddStaticRevealer(GetActorLocation(), 1800.f);
    // Store handle to update or remove later
}
```

---

## Troubleshooting & Tips

- **Nothing shows up?** Ensure at least one **Fog Instance** is enabled and your materials/post process bind the **shared MID**.
- **Stealth not reacting?** Check `ActorVisibilityThreshold` vs `ExploredValue`, and the **stealth update rate**.
- **Expensive frames?** Lower **LOD**, reduce **Blur**, increase **UpdateRate**, or disable occlusion if not needed.
- **Multiple instances**: Keep them minimal; each has its own full pipeline and render target.
