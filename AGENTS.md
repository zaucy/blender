# AGENTS.md — Viewport Render Resolution Maintenance Guide

This document is written for AI coding assistants and developers maintaining or extending the `viewport-render-resolution` branch of Blender.

---

## 1. Feature Overview

The **Render Resolution** preview pixel size mode (`SCE_PREVIEW_PIXEL_SIZE_RENDER`) allows the 3D viewport to render at a 1:1 pixel match with the scene's configured render resolution (`RenderSettings.resolution_x`, `resolution_y`, scaled by `resolution_percentage`).

### Primary Goals:
1. **1:1 Camera Matching**: When looking through the active camera in Rendered or Material Preview mode, the viewport render buffer dimensions exactly match the output render resolution. Low-res pixel art renders can be previewed live in the viewport without needing to render to an image editor.
2. **Camera-Synchronized Orthographic View**: When outside the camera in orthographic mode (e.g., isometric or orthographic projection), the viewport simulates the same world-space pixel density as the camera. Panning and zooming outside the camera does not alter, blur, or distort the pixel art look of 3D geometry.
3. **Engine Parity**: Functions identically across both **Cycles** and **EEVEE**.

---

## 2. Codebase Architecture & File Map

### DNA / RNA Settings
- [`source/blender/makesdna/DNA_scene_enums.h`](source/blender/makesdna/DNA_scene_enums.h):
  - Defines `SCE_PREVIEW_PIXEL_SIZE_RENDER = 0` in `eScenePreviewPixelSize`.
  - Shifts standard divider constants (`SCE_PREVIEW_PIXEL_SIZE_1 = 1`, `2`, `4`, `8`, `16`).
- [`source/blender/makesrna/intern/rna_scene.cc`](source/blender/makesrna/intern/rna_scene.cc) & [`rna_render.cc`](source/blender/makesrna/intern/rna_render.cc):
  - Adds `"RENDER"` / `"Render Resolution"` to the `preview_pixel_size` RNA enum items with description: *"Render viewport at 1:1 match with output render resolution"*.

### Core Calculations (`blenkernel`)
- [`source/blender/blenkernel/BKE_scene.hh`](source/blender/blenkernel/BKE_scene.hh) & [`intern/scene.cc`](source/blender/blenkernel/intern/scene.cc):
  - Helper functions: `BKE_render_is_preview_render_resolution(const RenderData *rd)` and `BKE_render_preview_pixel_size_factor(const RenderData *rd)`.
- [`source/blender/blenkernel/BKE_camera.h`](source/blender/blenkernel/BKE_camera.h) & [`intern/camera.cc`](source/blender/blenkernel/intern/camera.cc):
  - `BKE_camera_preview_render_target_calc(...)`:
    - For camera view: computes exact render resolution matching `(render_x, render_y)`.
    - For ortho view outside camera: computes `target_res = round(view_extent / pixel_world)`.
  - `BKE_camera_preview_render_subpixel_phase_calc(...)`:
    - Calculates world anchor point (active camera object position + camera shifts, or world origin).
    - Computes continuous subpixel phase offsets (`r_phase_x`, `r_phase_y`) and quad bounds (`quad_x`, `quad_y`, `quad_w`, `quad_h`) for display scaling.
    - Applies **Grid Parity Compensation**: `0.5f * float(target_x - render_x)` to eliminate half-pixel alternating jumps during zoom.

### EEVEE Engine Implementation
- [`source/blender/draw/engines/eevee/eevee_camera.cc`](source/blender/draw/engines/eevee/eevee_camera.cc):
  - `Camera::render_resolution_get(...)`: sets low-res framebuffer target dimensions.
  - `Camera::matrices_init(...)`: locks `data.winmat[0][0] = 2.0f / (target_res_x * pixel_world_x)` and `data.winmat[1][1] = 2.0f / (target_res_y * pixel_world_y)` in orthographic mode so 3D world texel density remains strictly constant.
- [`source/blender/draw/engines/eevee/eevee_film.cc`](source/blender/draw/engines/eevee/eevee_film.cc), [`eevee_film_shared.hh`](source/blender/draw/engines/eevee/eevee_film_shared.hh), [`shaders/eevee_film.bsl.hh`](source/blender/draw/engines/eevee/shaders/eevee_film.bsl.hh):
  - Passes `camera_border_frame` (`quad_x, quad_y, quad_w, quad_h`) to the film composite pass to smoothly display and center the low-res quad across the viewport window with continuous scaling.

### Cycles Engine Implementation
- [`intern/cycles/blender/camera.cpp`](intern/cycles/blender/camera.cpp):
  - `sync_camera_resolution(...)`: clamps viewport resolution buffer to `(target_w, target_h)`.
  - Ortho projection locking: sets `bcam.ortho_scale = locked_extent / bcam.zoom` using `blender_camera_horizontal_fit(&bcam, target_w, target_h)`.
  - Computes display quad parameters and stores them on `BlenderDisplayDriver`.
- [`intern/cycles/blender/display_driver.cpp`](intern/cycles/blender/display_driver.cpp) & [`intern/cycles/blender/session.cpp`](intern/cycles/blender/session.cpp):
  - Applies `display_params` directly during draw, enabling smooth subpixel quad panning and continuous zoom scaling without resetting path tracing samples.
- [`intern/cycles/integrator/path_trace*`](intern/cycles/integrator/) & [`intern/cycles/session/*`](intern/cycles/session/):
  - Supports resolution buffer resizing and display coordinate translation without memory reallocation thrashing.

### UI Scripts
- [`scripts/startup/bl_ui/properties_render.py`](scripts/startup/bl_ui/properties_render.py):
  - Removed `DEFAULT_CLOSED` from `RENDER_PT_eevee_performance_viewport` so Viewport -> Pixel Size is open by default under Performance in EEVEE.
- [`scripts/startup/bl_ui/space_view3d.py`](scripts/startup/bl_ui/space_view3d.py):
  - Added `VIEW3D_PT_shading_performance` to expose the Pixel Size dropdown in the 3D Viewport Shading Popover (header) in Material Preview and Rendered modes.

---

## 3. Key Mathematical Models & Invariants

### A. Locked World Pixel Size
For a camera view with viewplane width $W_{\text{cam}}$ and render resolution $R_x$:
$$\text{pixel\_world\_x} = \frac{W_{\text{cam}}}{R_x}$$
In orthographic mode outside the camera, the target resolution is:
$$T_x = \mathrm{round}\left(\frac{W_{\text{view}}}{\text{pixel\_world\_x}}\right)$$
The projection matrix is locked to:
$$W_{\text{locked}} = T_x \cdot \text{pixel\_world\_x}$$
This guarantees that 1 texel in the render buffer represents **exactly $\text{pixel\_world\_x}$ meters in 3D world space at all times**.

### B. Ray Grid Parity Compensation
- Ray centers through the camera frame: $u_{\text{cam}}(i) = (i - 0.5 \cdot R_x + 0.5) \cdot \text{pixel\_world}$.
- Ray centers through the viewport frame: $u_{\text{view}}(j) = (j - 0.5 \cdot T_x + 0.5) \cdot \text{pixel\_world}$.
- To eliminate the alternating $0.5 \cdot \text{pixel\_world}$ shift when $T_x$ changes parity (even vs. odd) during zoom, the phase offset formula adds:
  $$\Delta \text{parity} = 0.5 \cdot (T_x - R_x)$$
  $$\text{offset\_pix} = \frac{\text{anchor\_view} - \text{view\_center}}{\text{pixel\_world}} + 0.5 \cdot (T - R)$$
  $$\text{phase} = \text{offset\_pix} - \mathrm{round}(\text{offset\_pix})$$

### C. Display Quad Scaling Invariant
The display quad dimensions on screen:
$$\text{quad\_w} = T_x \cdot \left(\text{pixel\_world} \cdot \frac{\text{winx}}{W_{\text{view}}}\right)$$
On-screen size of an object of world dimension $X$:
$$\text{screen\_size} = \frac{X}{\text{pixel\_world}} \cdot \frac{\text{quad\_w}}{T_x} = X \cdot \frac{\text{winx}}{W_{\text{view}}}$$
The discrete integer steps $T_x$ cancel out entirely. Zooming smoothly scales the pixel art on screen with zero jitter or edge pop.

---

## 4. Tips & Gotchas for Next Agents

1. **Python Script Loading in Release Builds**:
   - `blender.exe` (on Windows located at `C:\Users\zekew\projects\build_windows_x64_vc18_Release\bin\Release\blender.exe`) loads startup scripts from `build_.../bin/Release/5.3/scripts/startup/bl_ui/`.
   - If you edit `.py` files in `scripts/startup/bl_ui/`, remember to copy them to the build directory or rebuild `blender` so runtime reflects the changes immediately.

2. **Git LFS Push to GitHub Forks**:
   - GitHub denies LFS uploads to public forks (`@zaucy can not upload new objects to public fork zaucy/blender`).
   - Because our commits contain only code diffs and no new LFS assets, **always push using `git push --no-verify`** to skip the local Git LFS pre-push hook:
     ```pwsh
     git push --no-verify -u fork <branch>
     ```

3. **Cycles Viewport Zoom Factor**:
   - In Cycles, `bcam.zoom = 2.0f` for 3D viewports. Therefore, when setting `bcam.ortho_scale`:
     ```cpp
     bcam.ortho_scale = locked_extent / bcam.zoom;
     ```
   - For horizontal fit checks in Cycles, always pass `(target_w, target_h)` instead of `(width, height)`:
     ```cpp
     const bool horizontal_fit = blender_camera_horizontal_fit(&bcam, target_w, target_h);
     ```

4. **Building with Ninja / MSVC on Windows**:
   - Compiler commands are located in `build_windows_x64_vc18_Release`.
   - Recompile after C++ changes with:
     ```pwsh
     ninja -C C:\Users\zekew\projects\build_windows_x64_vc18_Release blender
     ```

5. **Headless Python Verification Scripts**:
   - Test scripts can be run using:
     ```pwsh
     C:\Users\zekew\projects\build_windows_x64_vc18_Release\bin\Release\blender.exe -b --python-expr "..."
     ```
   - For windowed render verification, launch Blender with a python script that sets the viewport shading to `RENDERED`, positions the view/camera, and saves a viewport render via `gpu.state` or window screenshots.
