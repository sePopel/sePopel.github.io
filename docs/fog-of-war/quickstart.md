# Quickstart

## 0) Install & Enable
- Install the plugin and enable **Fog Of War** in your UE5 project.
- Restart the editor.

## 1) Configure Project Settings 
- Open **Edit → Project Settings → Plugins → Fog Of War**.
- **Profiles:** Each profile creates a new fog area. Per default, the first profile uses the plugin internal render target and material parameter collection. Additional profiles require their seperate render target and material parameter collection.

> You can always override these at runtime via the **Subsystem** (e.g., `SetLayerUpdateRate`, `SetLayerShowMemory`, `SetLayerActorVisibilityThreshold`, etc.) and add additional profiles during runtime.

## 2) Add a Manager
- In your level, place **FOW Manager** (`AFOWManager`).
- Select the Fog Variant you want to use and setup its parameters.

> **Advanced:** You can skip the Manager entirely, create your fog setup manually and bind its materials yourself by asking the **Subsystem** for a shared MID and applying it where you need it (or by using the fog areaws controlled material parameter collection). 


## 3) Vision Source 
- Add **FOW Vision Component** to your _seeing_ bots blueprints.
- Set the desired vision range and a possible Eye Offset (if the bot is also an occluder, this is possibly needed to prevent self-occlusion).
- The **FOW Manager** will automatically add the Vision actor to the fog layer.
- At runtime, the subsystem reads each Vision actor’s transform and writes visibility into the fog render targets.

## 4) Occlusion 
- Add **FOW Blocker Component** to occluding actors.
- Two modes:
    - **Circle**: set **Radius**.
    - **Capture Writer**: uses a scene capture to write depth/coverage into the occluder mask for the selected fog areas.

## 5) Stealth / Ghosts
- Add **FOW Stealth Component** to actors that should react to the visibility within the fog.
- Two modes:
    - **Hidden Mode**: *FreezeWithGhost* for a frozen version of themselves when leaving vision.
    - **Fade Seconds**: Controls the fade-in/out time of the actor.
    - **Fade Material Param**: The parameter that gets controlled in the actors materials

> **Heads-up:** The **FOW Manager** and all **components** (Vision/Blocker/Stealth) are *optional convenience helpers*. Users can also control fog areas and profiles, vision sources, blockers, stealth actors and visibility queries **directly via the `Fog Of War Subsystem`** (Blueprint or C++). See the API page for those calls. 
> 
> The creation of the fog visualization and fog-affected materials can also be done manually. This can be achieved by using the fog areas controlled material parameters within the material and registering it with the subsystem or by using the created Material Parameter Collection.

## 6) Press Play
- You should see fog revealing around your unit, blockers occluding and stealth actors freeze or hide within the fog.

### Subsystem-only quick pointers
> **Driving everything from the Subsystem**
> - Get it in BP/C++: `GetWorld()->GetSubsystem<UFogOfWarSubsystem>()`
> - Per-layer controls: `SetLayerEnabled`, `SetLayerShowFog/ShowMemory/FreezeMemory`, `SetLayerUpdateRate`, `SetLayerActorVisibilityThreshold`, `ClearMemory`,  `AddLayer` etc.
> - Materials: `AcquireSharedMIDForLayer(Name, BaseMaterial)` (binds fog params for you)
> - AI: use the **batched visibility queries** instead of many single reads

**Next**: read [Performance](performance.md) to tune blur and update cadence per platform.
