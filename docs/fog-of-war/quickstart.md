# Quickstart

This guide will show you how to get started with the plugin. It will cover the most crucial steps to get the features of the plugin up and running. Consider reading the [API](api.md) for more details and insight.

## 0) Install & Enable
- Install the plugin and enable **Fog Of War** in your UE5 project.
- Restart the editor.

## 1) Configure Project Settings 
- Open **Edit → Project Settings → Plugins → Fog Of War**. Here you can setup the initial fog instances. Per default, the first profile uses the plugin internal render target and material parameter collection.
- **Profiles:** Profile Data Assets can be used to create additional, new fog instances. Additional profiles require their separate render target and material parameter collection.

> You can always override these at runtime via the **FOWSubsystem** (e.g., `SetFogInstanceUpdateRate`, `SetFogInstanceShowMemory`, `SetFogInstanceActorVisibilityThreshold`, etc.) and add additional profiles during runtime.

## Two Paths
**Heads-up:** The **FOW Manager** and all **Components** (Vision/Blocker/Stealth) are *optional convenience helpers*. Users can also control fog instances and profiles, vision sources, blockers, stealth actors and visibility queries **directly via the `FOW Subsystem`** (Blueprint or C++). See the API page for those calls.

The creation of the fog visualization and fog-affected materials can also be done manually. This can be achieved by using the fog instances controlled material parameters within the material.
This can either be done by using the fog instances created Material Parameter Collection or by asking the **Subsystem** for a shared MID and applying it where you need it.


---
## Use the convenient helpers

For each of the manual steps the plugin offers different helpers. 

### 2) Add a Fog Manager (helper for runtime fog instances and visualizations)
The **FOW Manager** sets up different, premade Fog Variants and interacts with the system to correctly work with them. He also allows you to add additional **Fog Profiles** to the system during runtime. One profile has to be added if you did not set up any fog instances in the project settings.

- In your level, place **FOW Manager** (`AFOWManager`).
- Select the Fog Variant you want to use and setup its parameters.


### 3) Vision Sources (what reveals the fog)
- Add **FOW Vision Component** to your _seeing_ bots blueprints.
- Set the desired vision range and a possible Eye Offset (if the bot is also an occluder, this is possibly needed to prevent self-occlusion).
- At runtime, the subsystem reads each Vision actor’s transform and vision settings and writes visibility into the fog render targets.

### 4) Occlusion (what blocks vision)
- Add **FOW Blocker Component** to occluding actors.
- Two modes:
    - **Circle**: set **Radius**.
    - **Capture Writer**: uses a scene capture to write depth/coverage into the occluder mask for the selected fog instances. Only has an effect if the scene capture is set to only work with specific actors.

### 5) Stealth / Ghosts (actors reacting to fog)
- Add **FOW Stealth Component** to actors that should react to the visibility within the fog.
- Two modes:
    - **Hidden Mode**: *FreezeWithGhost* for a frozen version of themselves when leaving vision.
    - **Fade Seconds**: Controls the fade-in/out time of the actor.
    - **Fade Material Param**: The parameter that gets controlled in the actors materials

If you want the actor to smoothly fade in/out, you need to control its opacity with an correctly named scalar material parameter.

---
## Manual Steps (no helpers)

If you prefer full control, you can drive the Fog of War **entirely via the Subsystem** and your own materials. All functions are available via Blueprint and C++.

### 2)Fog instance control and Visualization

#### 2.1) Add / Position Fog Instances (runtime)
-  **At runtime**, get the subsystem and add (or replace) a named fog instance. This is mandatory if you did not set up any instances in the project settings:
    - **Blueprint:** `AddOrReplaceFogInstance(Config)`
    - **C++:**
      ```cpp
      if (auto* FOW = GetWorld()->GetSubsystem<UFOWSubsystem>())
      {
          FFOW_FogConfig Cfg;
          Cfg.Name = "Main"; // your fog instance name
          // Optionally assign external RT/MPC assets in Cfg if not using plugin defaults
          FOW->AddOrReplaceFogInstance(Cfg);
      }
      ```
-  **Position & size** the fog instances area if necessary, this has to be done if the size is not yet known before runtime and has not been set in the project settings:
    - **Function:** `SetFogInstanceArea(Name, Center, ExtentsFull, YawDeg)`

#### 2.2) Create, Hook up and use Materials 
Ask the subsystem for a **shared MID** bound to the fog instance and apply it anywhere. The subsystem will try to update the defined parameters in the MID.

To make setting up your own fog materials easier, the plugin provides a material function containing the default parameters for the subsystem. It calculates multiple different values convenient for your custom materials:

- **Blueprint:** ` FOW->AcquireSharedMIDForFogInstance("Main", Base)`
- **C++:**
  ```cpp
  if (auto* FOW = GetWorld()->GetSubsystem<UFOWSubsystem>())
  {
      UMaterialInterface* Base = MyBaseMaterial; // post-process, mesh, decal, light func, etc.
      if (auto* MID = FOW->AcquireSharedMIDForFogInstance("Main", Base))
      {
          UseMaterial(MID); 
      }
  }
  ```
> Manual binding alternative: create your own **MPC** and **RT**, reference them in your profile/config, and wire parameters yourself.

### 3) Register Vision Sources (what reveals the fog)
- **Vision Actor:** `RegisterVisionSource(Actor, VisionRange, EyeHeightOffsetWS)` / `UnRegisterVisionSource`
- **Static (no actor):** `AddStaticRevealer(WorldPos, RangeWS)` → handle; later `RemoveStaticRevealer(Handle)`

### 4) Add Occluders (what blocks vision)
Occlusion can be done in two ways, either by a circle or by a scene capture. Circular occlusion can be added as follows:

- **Circular Occlusion around Actor:** `RegisterBlocker(Actor, Radius)` / `UnregisterBlocker`
- **Static:** `AddStaticBlocker(WorldPos, RangeWS)` → handle; later `RemoveStaticBlocker(Handle)`

If the scene capture is set to only work with specific actors, you need to register them:

- **Register Capture Writer:** `RegisterCaptureWriter(Actor, CaptureName)` / `UnregisterCaptureWriter`

### 5) Stealth / Ghosts (actors reacting to fog)
- **Register:** `RegisterStealthActor(Actor, HiddenMode, VisibilityPolicy)` / `UnregisterStealthActor(Actor)`

Contrary to the other helper components, using the stealth component comes with some features, which are not available without using it. This includes a callback when the actor is no longer visible, as well as a smooth fade-in/out.

---

## 6) Press Play
- You should see fog revealing around your unit, blockers occluding and stealth actors freeze or hide within the fog.

## Subsystem usage - additional quick pointers
 **The Subsystem is the main entry point for all fog-related functionality. It is accessible via Blueprint and C++.**

 - Get it in BP/C++: `GetWorld()->GetSubsystem<UFogOfWarSubsystem>()`

### Per-FogInstance Runtime Controls (visuals, blending, perf)
- **Toggles:** `SetFogInstanceEnabled`, `SetFogInstanceShowFog`, `SetFogInstanceShowMemory`, `SetFogInstanceFreezeState`, `SetFogInstanceFreezeMemory`
- **Visual:** `SetFogInstanceExploredValue`, `SetFogInstanceActorVisibilityThreshold`, `SetFogInstanceSmoothFalloff`, `SetFogInstanceBlurIterations/Radius/Sigma`
- **Blending:** `SetFogInstanceTransitionMode`, `SetFogInstanceFadeInRate`, `SetFogInstanceRelaxRate`, `SetFogInstanceForgetRate`
- **Performance:** `SetFogInstanceLOD`, `SetFogInstanceUpdateOnlyAtCrucialChanges`

### Queries (AI/gameplay)
Ask the GPU for **final visibility** at world positions:

- **Single (latent):** `CheckPointVisibility(WorldPos, bVisible, Value, Threshold)`
- **Batched (latent):** `CheckPointsVisibility(WorldPositions, OutVisible, OutValues, Threshold)`
- **Advanced handles:** `StartPositionQuery_Auto(...)` → `PollPositionQueryResults(...)`

 Use a **negative Threshold** to auto-use the fog instance’s cutoff. Prefer **batched** queries over many singles for performance.


**Next**: read [Performance](performance.md) to keep an eye on the most important performance wins.
