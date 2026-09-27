# Forge — All-in-One Godot Asset Studio (Web-Based)

> Import raw 3D models and Godot Engine scenes. Light them, frame them, render them.
> All in the browser — no install, no server, no upload of your files anywhere.

---

## 🚀 Elevator Pitch (short version — for LinkedIn / social)

**Forge** is a browser-based 3D asset studio built specifically for game developers working with **Godot Engine**. Drop in a raw `.tscn` scene file — Forge parses it, reconstructs the visual geometry, reapplies the PBR materials, and gives you a full lighting/camera/environment studio to turn it into a polished icon, skin preview, or store thumbnail. Everything runs client-side in the browser: your project never leaves your machine.

No more exporting a throwaway screenshot from the editor and cropping it in Photoshop. Point Forge at your scene folder, pick a preset, hit **Render**, done.

---

## A. Product Description & Core Value

Godot developers constantly need clean, presentable renders of their in-game assets — for the Steam store page, item icons, skin previews, marketing screenshots, Discord announcements. The usual workflow means jumping back into the Godot editor, wrestling with a throwaway camera and lighting rig, and screenshotting the viewport. It's slow, and it's disconnected from the tools artists actually like using for compositing and lookdev.

**Forge closes that gap.** It's a purpose-built, all-in-one **3D viewer, lighting studio, and render pipeline** that runs entirely in a web browser — reachable from a link, with zero installation. It accepts standard 3D formats (**GLB, GLTF, OBJ, STL**) *and*, uniquely, **raw Godot `.tscn` scene files**, parsing them directly without ever touching Godot's own editor or export pipeline.

### Feature highlights

| Category | What it does |
|---|---|
| **Camera** | Full manual control (position, rotation, FOV, target) plus 10 one-click presets — Front, Back, Isometric, Showcase, Close-up, Portrait, and more |
| **Background** | Solid color, two-stop gradient, or a custom uploaded image — fully independent from lighting |
| **Environment** | Separate from background — controls ambient light color/intensity and scene "mood" (Studio, Sunset, Night, Dramatic, Outdoor, etc.) with rotation and intensity sliders |
| **Lighting** | Key light intensity & angle, shadow toggle, ground plane toggle, five ready-made lighting moods |
| **Templates** | One-click presets that combine camera + background + environment for common jobs: *Item Preview*, *Skin Preview*, *Showcase*, *Product* |
| **Transform** | Full position/rotation/scale control per model, with uniform-scale slider and one-click reset |
| **Material Editor** | Full PBR material editing per-mesh: base color, metalness, roughness, normal map, emissive (color + intensity), ambient occlusion, opacity, wireframe, double-sided — with live texture-preview thumbnails, and manual texture attachment for untextured models |
| **Render** | Choose exact output resolution (256² up to 1920×1080, or custom), transparent PNG support, JPG/WebP export, instant preview, one-click download |
| **Responsive** | Full mobile/tablet support — sidebars collapse into swipe-out drawers, safe-area aware for notches, touch-optimized controls, works in portrait and landscape |

Every one of these systems is genuinely functional — not a mockup. Adjusting any slider updates the viewport in real time, and the final render is captured from an actual full-resolution render target, not an upscaled screenshot of the UI.

---

## B. The Masterpiece: Native Godot `.tscn` Import

This is the feature that sets Forge apart from every generic "drag-and-drop a GLB" viewer online: **Forge reads Godot's actual scene file format directly** — the same human-readable text format Godot itself saves to disk — and reconstructs a renderable scene from it, entirely client-side.

### How the parser works

1. **Text-level parsing, not a game engine.** A `.tscn` file isn't JSON or XML — it's Godot's own resource format, structured as a series of `[section]` blocks (`[ext_resource]`, `[sub_resource]`, `[node]`). Forge's parser walks the file line-by-line, splitting it into these blocks and decoding each block's properties — including Godot's native value types (`Vector3(...)`, `Color(...)`, `Transform3D(...)`, and both `ExtResource(...)` / `SubResource(...)` references) — into a structured in-memory scene graph.

2. **Visual-only filtering, by design.** A Godot scene tree mixes visual meshes with collision shapes, physics bodies, trigger areas, cameras, and lights — none of which should appear in a rendered icon. Forge's importer filters the node tree at the parsing stage: only `MeshInstance3D` nodes are ever converted into renderable geometry. Everything else — `CollisionShape3D`, `StaticBody3D`, `Area3D`, `Camera3D`, `DirectionalLight3D`, and so on — is recognized and silently discarded before it ever reaches the 3D engine. The status log reports exactly how many nodes were imported versus skipped, so nothing is a mystery.

3. **Procedural primitive reconstruction.** Godot's built-in primitive shapes (`BoxMesh`, `SphereMesh`, `CylinderMesh`, `CapsuleMesh`, `PlaneMesh`) aren't stored as mesh data in the `.tscn` — they're stored as *parameters* (a size, a radius, a height). Forge reads those parameters straight out of the `[sub_resource]` block and **regenerates the actual 3D geometry procedurally, in the browser**, matching Godot's own primitive dimensions exactly. No placeholder boxes, no missing shapes.

4. **PBR material translation.** Every `StandardMaterial3D` / `ORMMaterial3D` resource attached to a mesh (whether as a node-level `surface_material_override` or embedded directly in a primitive mesh's own `material` property) is decoded and translated into an equivalent physically-based material in the render engine: `albedo_color → color`, `metallic → metalness`, `roughness → roughness`, and `albedo_texture → base color map` — including resolving and loading the actual texture image referenced by the material.

5. **Solving the `res://` problem — entirely in-browser.** This is the hardest part of importing Godot scenes on the web: every asset path in a `.tscn` is written as a project-relative `res://folder/file.ext` reference, which only means something *inside a real Godot project folder*. A bare `.tscn` file alone is just a text map — it can't be rendered without its original assets sitting alongside it.

   Forge solves this without a server, by accepting either:
   - **A full project folder upload** (via the browser's native folder-picker), or
   - **A `.zip` of the project** (parsed client-side with a bundled JSZip module)

   Every file that comes in is indexed into an in-memory map, keyed by its path relative to the project root. As the parser encounters each `res://...` reference, it strips the prefix and resolves it against that map — with a tiered fallback (exact path → folder-suffix match → filename-only match) so that even projects whose folder structure doesn't perfectly mirror the original are still resolved wherever possible. The matched file is read directly out of browser memory and handed to the renderer as a `Blob` — **nothing is ever uploaded to a server.**

---

## C. Technical Specification (Under the Hood)

**Rendering engine:** [Three.js](https://threejs.org/) (r128), running on WebGL, is the core 3D engine — handling the scene graph, PBR material system, camera, lighting, shadow mapping, and the full render pipeline. Model parsing is handled by Three.js's own `GLTFLoader`, `OBJLoader`/`MTLLoader`, and `STLLoader` modules; the Godot `.tscn` interpreter described above is fully custom-built on top of this, translating Godot's scene format into native Three.js scene-graph objects and materials.

**Architecture:** A single self-contained, dependency-light web application — HTML, CSS, and vanilla JavaScript, with Three.js and JSZip loaded as the only external libraries. No build step, no backend, no database. Everything — model parsing, `.tscn` interpretation, material editing, and final image rendering — happens **entirely client-side, in the user's own browser.**

**Memory management for large projects:** Because Forge deliberately avoids ever sending project files to a server, keeping memory usage under control in the browser is critical, especially for large Godot projects with many textures. Key strategies:

- **On-demand, not eager, asset loading.** Uploaded files are indexed by path into a lightweight `Map` immediately, but the actual texture/model *data* is only decoded and uploaded to the GPU when a `.tscn` node actually references it — never for unused project files sitting in the same folder/zip.
- **Object URL lifecycle discipline.** Every binary asset handed to the renderer is wrapped in a short-lived `Blob URL`, explicitly revoked (`URL.revokeObjectURL`) the moment it's no longer needed, preventing the classic browser memory leak of accumulating undisposed blob references across repeated imports.
- **Automatic texture downscaling.** Real-world assets — especially phone-camera photos used as manually-attached textures — are frequently far larger (often 3000–8000px) than a real-time viewport or icon render needs. Forge decodes every incoming image via `createImageBitmap` (with a classic `<img>`-based fallback for maximum format compatibility) and automatically downsamples anything above 2048px on its longest edge onto an offscreen canvas before it ever becomes a GPU texture — dramatically cutting GPU memory pressure with no visible quality loss for this use case.
- **Full resource disposal on scene changes.** Swapping or resetting a model explicitly disposes the old geometry, materials, *and* their attached textures (`.dispose()` calls across the board) rather than just dereferencing them, so garbage collection isn't left to guess what's safe to reclaim.
- **Resilient, non-blocking async loading.** Every node and every texture in a Godot scene is loaded independently and defensively (`Promise.allSettled` + per-node error handling + load timeouts), so one missing or oversized asset degrades gracefully — the rest of the scene still renders — instead of one bad file aborting the entire import.

**Platform support:** Fully responsive across desktop, tablet, and mobile (Android/iOS), with dedicated portrait/landscape layouts, safe-area-aware UI for notched devices, and touch-optimized controls throughout.

---

## Suggested Usage by Context

- **GitHub `README.md`:** Use sections B and C in full — they're written to double as technical documentation.
- **Portfolio description:** Use the Elevator Pitch + the Feature Highlights table + one paragraph from Section A.
- **LinkedIn / social post:** Use the Elevator Pitch section as-is, or trim to 2–3 sentences plus a link and a screenshot/GIF of the render flow in action.
