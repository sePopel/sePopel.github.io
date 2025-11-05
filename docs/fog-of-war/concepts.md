# Concepts

## Fog Areas & Profiles
A **Fog Areas** is one fog stack (fog, memory, explored) writing a mask into one destinct render target. Each render target can be used for many different fog variants and scratches over a certain area in space.

![Render Target for the Fog Mask](../images/RenderTarget.png)

The system allows the usage of multiple fog areas, each using their own profile. However, it is still recommended to use as little different profiles as possible and merge close by fog areas to use just one render target.  Profiles can either be added within the project settings or created as Data Assets and added during runtime. 

**Primary Layer** is the default Fog Area using the plugins internal render target and MPC. 

Each layer tracks its placement as a 2D oriented box in world space:

- **Center** (XYZ)
- **Extends** (full size XYZ; not half-extents)
- **YawDegrees** (rotation in the plane)

At runtime you can set or expand the area via the Subsystem (e.g., `SetLayerArea`, `ExpandLayerAreaToInclude`).

---

## Material Parameters (MPC & Dynamic MIDs)

For every active layer the subsystem pushes the following **parameter names** (used for both MPC and dynamic MIDs). For additional profiles these names are customizable.

- `FOW_Enabled`(If the Fog System is currently updating this layer)
- `FOW_FogCenter` (Origin World XYZ)
- `FOW_FogSize`   (XYZ full size)(while the XZ axis map the render target into the world, the Z axis is only needed for certain fog visualizations)
- `FOW_FogYawDegrees`
- `FOW_ActorVisCutoff` (Highest mask value the system treats as hidden)
- `FOW_RT` (Texture param for the layer’s render target, only on dynamic materials)

You can create or aquire the registered **MID** already bound to a layer for a certain material via the Subsystem.

---

## Runtime Controls (per Layer)

Core runtime flags can be set before or during runtime per Fog Area.

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

Global defaults live in **Project Settings → Plugins → Fog Of War**.

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

## Final Fog Mask & Memory
The visibility within a fog area is stored within an editor exposed render target. To get to the final fog mask, many different compute shaders are used.

Using a certain, user-set period the current fog state is recalculated. To keep transition smooth, each tick blends between the current fog state and the previous one, until it is updated again.
When a unit leaves an area, a *memory* texture marks it as already explored. These areas are represented by a darker value as currently visible areas within the final mask. Blending between visibility states happens smoothly (linear or exponential).
- **Relax Rate** controls how fast the memory value is reached.
- **Fade In Rate** controls how fast new areas are discovered.
- **Forget Rate** slowly reduces the memory value of areas that are no longer visible.

You can freeze/clear/hide memory per layer at runtime.

