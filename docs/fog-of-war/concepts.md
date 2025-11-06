# Concepts

## Subsystem

All the relevant data is managed within and through the **FOW Subsystem**, which is a **World Subsystem**. One instance exists per game world (client or server). 
Even though the subsystem has mostly only visual use-cases, it is still created on the server, as it can be used to drive game mechanics. It is created automatically when the world initializes and is destroyed with the world.

---

## Fog Areas, Textures & Profiles
The system allows the creation of multiple, self-contained **Fog Areas**, each covering a certain area in space and consisting out of their own resources and runtime-setting.
For each of the existing areas, the plugin writes a mask into one distinct render target. 

### Render Target Texture
By mapping these textures over their corresponding areas, shaders can use it to control different parameters, like translucency or colour. 
Like this, each render target texture can be used for many different effects, like the plugins premade fog variants.

Moreover, by reading back the textures value at the pixel which maps to a certain position, the CPU can also query the mask to control game mechanics. The plugin uses this for example to blend actors in or out. 

Logically, the higher the resolution and the smaller the covered area, the greater the quality.
Additionally, the system allows to downscale the actual resolution used when computing the mask, which generally works best when transitions are kept smooth.

To make these textures usable in shaders, the system also forwards additional data to them via material parameter collections or registered dynamic materials.

![Render Target for the Fog Mask](../images/RenderTarget.png)

As already mentioned, the system allows the usage of multiple fog areas, each using their own profile. However, it is still recommended to keep the count of the different fog areas within the system as little as possible, as the render pipeline is executed for each one individually. 
Merging close by fog areas together, might result in the corresponding Render Target covering wide ares
to use just one render target.  Profiles can either be added within the project settings or created as Data Assets and added during runtime.

### Material Parameters (MPC & Dynamic MIDs)

For every active fog area, the subsystem pushes data to its MPC and MIDs via the selected **parameter names**. The default names are as follows. However, for additional profiles these names are customizable.

- `FOW_Enabled`(If the Fog System is currently updating this layer)
- `FOW_FogCenter` (Origin World XYZ)
- `FOW_FogSize`   (XYZ full size)(while the XZ axis map the render target into the world, the Z axis is only needed for certain fog visualizations)
- `FOW_FogYawDegrees`
- `FOW_ActorVisCutoff` (Highest mask value the system treats as hidden)
- `FOW_RT` (Texture param for the layer’s render target, only on dynamic materials)

You can create or aquire the registered **MID** already bound to a layer for a certain material via the Subsystem.

### Covered Area

Each layer tracks its placement as a 2D oriented box in world space:

- **Center** (XYZ)
- **Extends** (full size XYZ; not half-extents)
- **YawDegrees** (rotation in the plane)

At runtime you can set or expand the area via the Subsystem (e.g., `SetLayerArea`, `ExpandLayerAreaToInclude`).

###  Profiles & Runtime Controls 

Each entry in the **Fog Areas** has its own name, assets, state and settings. The initial settings and assets can either be setup before runtime through the project settings, findable in **Project Settings → Plugins → Fog Of War**, or
during runtime. At runtime, new entries can be added either purely through the subsystems functions, or by applying a **FOW Profile** data asset, which can be used to save certain setups.

Per default the plugin only contains one render target asset and one MPC. Additional areas require their own, unique render target and asset, which have to be manually created and hooked up.
**Primary Layer** is the default Fog Area, using the plugins internal render target and MPC. 

Core runtime flags and values can be set before or changed during runtime per Fog Area. The most important ones are as follows:

- **Enable/Disable** layer updates and presentation
- **Show Fog** (final visibility overlay)
- **Freeze State** (stop recomputing current state; blending still happens)
- **Show Memory** (ever-seen contribution)
- **Freeze Memory** (stop accumulating new exploration)
- **ExploredValue** (brightness of explored areas, also used by queries if desired)
- **SmoothFallOff** (edge softness of vision)
- **Blur**: `BlurIterations`, `BlurRadius`, `BlurSigma`
- **Actor Visibility Cutoff** (0..1) - used by GPU queries and stealth logic
- **Update Rate (seconds)** -cadence for recomputation (blending runs every tick)

---

## Vision & Blockers
- **Vision Actors or Static Vision Sources** write a circular, smooth mask around the actor’s location, optionally emitting reveal rays across multiple occlusion textures, if the feature is used.
- **Blocker Actors or Static Blockers** contributes to an **occluder mask**, either with a:
    - **Circle**: radial occluder.
    - **Capture Writer**: writes the object height into the occluder masks for selected fog areas, great for irregular geometry.
- It is also possible to include every visible Primitive within the Capture by toggling the ShowOnly mode of the profile.

### Occlusion Modes & Ray Methods
To figure out occlusion, the system allows the usage of many different Ray Methods, each shining within a different use case. Dependent on the method, the occlusion textures consist out of a height texture, a max mip chain of coverage masks and a distance field texture.

- **Step Scan**  
  March 1 px per step over the coverage/height textures.  

- **DDA (Grid DDA)**  
  Traverses the base coverage/height grid cell-by-cell.  

- **Hierarchical DDA (recommended if occlusion changes often)**  
  Marches over the **coverage mip chain**, skipping empty parents; (confirms at mip0 optionally with height).  
  *Pros:* fast in sparse scenes or with pure coverage occlusion, recalculation of textures is cheap. *Cons:* many unnecessary texture reads and steps if height is not blocking.

- **Distance Field (recommended if occlusion changes rarely)**  
  Trace using a precomputed distance field (with `DFScale`) and optionally confirm with height.  
  *Pros:* very fast in open areas and when blockers change infrequently. *Cons:* requires DF generation.

> **Tips**
> - Start with **Hierarchical DDA**.
> - Switch to **Distance Field** if occlusion rarely changes (tune `DFScale`).
> - If height occlusion is necessary and many occluders below the vision sources height exist, consider switching to either **DDA** or **Step Scan**.
> - Use **Capture Writer** for complex occluders; **Circle** for cheap bulk obstacles.

---

## Stealth, Visibility & Ghosts
- **Visibility Threshold** decides per fog area when an actor or query counts as “seen”.
- **Stealth Actors** respond to these visibility changes.
  - They can either freeze their current state by spawning a so called **Ghost Actor** or, disappear completely.
  - The usage of the **Stealth Component** and adding a material parameter to the actors material allows for smooth fading between states and is generally recommended.

---

## Computing the Mask & Memory
The visibility within a fog area is stored within an editor exposed render target. To get to the final fog mask, many different compute shaders are used.

Using a certain, user-set period the current fog state is recalculated. To keep transition smooth, each tick blends between the current fog state and the previous one, until it is updated again.
When a unit leaves an area, a *memory* texture marks it as already explored. These areas are represented by a darker value as currently visible areas within the final mask. Blending between visibility states happens smoothly (linear or exponential).
- **Relax Rate** controls how fast the memory value is reached.
- **Fade In Rate** controls how fast new areas are discovered.
- **Forget Rate** slowly reduces the memory value of areas that are no longer visible.

You can freeze/clear/hide memory per layer at runtime.

---

## Visibility & Value Queries