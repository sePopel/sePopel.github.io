# FAQ

**Does it work in multiplayer?**  
Yes. The subsystem only runs in game worlds, resets cleanly on world transitions, and avoids editor/PIE leaks.

**How do I disable memory?**  
Per layer: `SetLayerShowMemory(false)` and/or `FreezeMemory(true)`. You can also clear memory during gameplay.

**How do I show a ghost when an enemy leaves vision?**  
Add **FOW Stealth Component**, set **Hidden Mode** to *FreezeWithGhost*, and set **Fade Seconds** / **Fade Param**.

**Post-process vs layered mesh vs light function?**
- *Post-process*: simplest global overlay.
- *Layered mesh*: good for world‑space fog sheets and distance‑based culling.
- *Light function*: project fog via a light; great for stylized looks.

**Why don’t I see any fog?**  
Check that a Manager is placed, materials are assigned (or left empty for defaults), and the layer is **Enabled**. If edges look too soft, reduce blur radius or iterations.
