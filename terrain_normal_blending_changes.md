# Terrain Normal Map Blending — Fork Changes Documentation

This document describes all changes made in the custom OpenMW fork (`F:\OPENMWBUILD\openmw`) relative to the upstream OpenMW dev branch to enable proper terrain normal map blending and exposure to the post-processing pipeline.

The upstream OpenMW bug: terrain renders one normal map per chunk (whichever texture dominates), unable to blend normals across texture layers. As a workaround, the post-processing normal buffer falls back to smooth per-vertex normals for terrain. These changes fix that by making every terrain layer write and alpha-blend its normal map into the normal MRT.

---

## 1. `files/shaders/compatibility/terrain.frag`

**Upstream:** ~106 lines, no normal blending logic.
**Fork:** ~318 lines with a full multi-layer normal blending pipeline.

### 1a. All Layers Write Normals

**Upstream behavior:** Only the final (top-most) terrain texture layer writes its normal to the normal buffer. Every other layer's contribution is discarded.

**Fork behavior:** Controlled by the shader define `writeNormals`, which the C++ side now sets to `"1"` for **all** layers (see material.cpp change below). Every layer writes its sampled normal map value, and the MRT alpha-blends them together across passes.

### 1b. Dual Normal Buffer Output

The fork outputs normals to **two** G-buffer slots:

```glsl
// gl_FragData[1]: per-pixel mapped (texture) normal, view-space, alpha-blended across all layers
gl_FragData[1] = vec4(viewNormal, alpha);

// gl_FragData[4]: smooth interpolated vertex normal, for post-processing fallback/merge
gl_FragData[4] = vec4(smoothViewNormal, 1.0);
```

The post-processing shader is expected to normalize the result of the alpha-blended `gl_FragData[1]` normals, since accumulated alpha-blending of unit vectors does not preserve unit length.

### 1c. Height-Biased Layer Blending

When parallax is enabled, the blend alpha between terrain layers is not just the raw blendmap value — it is modified by the sampled height from the parallax/height map:

```glsl
float heightDev = texture2D(heightTex, uv).r - 0.5;
float jitter = <noise>;
float blendAlpha = smoothstep(0.4, 0.6, rawBlend + heightDev + jitter);
```

This creates organic, height-aware transitions at layer boundaries rather than flat blendmap-driven cutoffs, which also benefits the normal map transitions.

### 1d. PBR and Wetness Pipeline

Dynamic wetness (driven by `weatherID`, `nextWeatherID`, `weatherTransition`, `rainIntensity`) modifies the roughness/smoothness of terrain surfaces, which affects how the normal map contributes to specular response. This is additive to the normal blending but complements it visually.

---

## 2. `components/terrain/material.cpp`

This is the most critical C++ file. Three distinct changes here.

### 2a. Blend Functions — RGBA Instead of Additive

**Upstream:**
```cpp
// First pass (base layer):
BlendFuncFirst: (SRC_ALPHA, ZERO)

// Subsequent passes (additional layers):
BlendFunc: (SRC_ALPHA, ONE)   // ← ADDITIVE blending
```

**Fork:**
```cpp
// First pass:
BlendFuncFirst: (SRC_ALPHA, ZERO, ONE, ZERO)   // Separate RGB and alpha blend ops

// Subsequent passes:
BlendFunc: (SRC_ALPHA, ONE_MINUS_SRC_ALPHA, ONE, ONE_MINUS_SRC_ALPHA)  // Proper alpha-over for ALL 4 channels
```

**Why this matters:** Upstream's additive blend on subsequent passes means normal contributions from multiple layers accumulate incorrectly — values can exceed 1.0 and the result is not a proper weighted blend. The fork's `ONE_MINUS_SRC_ALPHA` destination factor produces a correct alpha-composited result across all four RGBA channels of the normal MRT.

### 2b. Normal MRT Blending Not Disabled

**Upstream:**
```cpp
if (!blendmaps.empty()) {
    if (sceneManager->getSupportsNormalsRT())
        stateset->setAttribute(new osg::Disablei(GL_BLEND, 1));  // Disables blending on RT index 1 (normals)
}
```

**Fork:**
```cpp
if (!blendmaps.empty()) {
    stateset->setMode(GL_BLEND, osg::StateAttribute::ON);
    // The Disablei calls for RT 1, 2, 3 are removed
}
```

**Why this matters:** The upstream explicitly disabled alpha-blending on the normal render target (`COLOR_ATTACHMENT1`). With blending disabled, only one layer's normal write survives — last-write-wins, not a blend. The fork removes this and lets all MRTs (normals, spec, etc.) participate in the same alpha-blending as the diffuse channel.

### 2c. `writeNormals` Define — All Layers vs. Last Only

**Upstream:**
```cpp
defineMap["writeNormals"] = (it == layers.end() - 1) ? "1" : "0";
```
Only the final layer gets `writeNormals = 1`, so all earlier layers short-circuit before writing to the normal buffer.

**Fork:**
```cpp
defineMap["writeNormals"] = "1";
```
Every layer writes normals. The alpha-blending state set up in 2a and 2b then composites them correctly.

### 2d. BlendmapTexMat Nudge — Shader Mode Only for Fixed Function

**Upstream:** Always applies a small UV nudge to the blendmap texture matrix to avoid seams (a fixed-function artifact correction).

**Fork:** The nudge is only applied in `!useShaders` (fixed-function) mode. In shader mode, the precision is sufficient and the nudge introduces a subtle UV offset that misaligns blendmap samples relative to the parallax-corrected normal UV, causing visible normal map drift at layer edges.

---

## 3. `components/terrain/material.hpp`

**Function signature change for `createPasses`:**

**Upstream:**
```cpp
std::vector<osg::ref_ptr<osg::StateSet>> createPasses(
    Resource::SceneManager* sceneManager,
    const std::vector<TextureLayer>& layers,
    const std::vector<osg::ref_ptr<osg::Texture2D>>& blendmaps,
    int blendmapScale, float layerTileSize,
    bool isComposite,
    bool esm4terrain = false);
```

**Fork:**
```cpp
std::vector<osg::ref_ptr<osg::StateSet>> createPasses(
    bool useShaders,               // New first parameter
    Resource::SceneManager* sceneManager,
    const std::vector<TextureLayer>& layers,
    const std::vector<osg::ref_ptr<osg::Texture2D>>& blendmaps,
    int blendmapScale, float layerTileSize,
    bool esm4terrain = false);
```

`useShaders` is threaded down from `chunkmanager.cpp` and controls the blendmap nudge behavior (see 2d above) as well as enables shader-specific blend function paths.

---

## 4. `components/terrain/chunkmanager.cpp`

### 4a. `createPasses` Call Site

Passes the new `useShaders` parameter, and removes the `forCompositeMap` argument (no longer needed since the fork consolidates composite map logic into the main terrain shader).

### 4b. Forced Shader Use with Unclamped Lighting

```cpp
bool useShaders = mSceneManager->getForceShaders();
if (!mSceneManager->getClampLighting())
    useShaders = true;
```

When lighting is unclamped (which is the typical modded/enhanced setup), shaders are forced on for terrain. This prevents a fallback to fixed-function rendering where normal map blending is impossible, and avoids visible normal map seams between chunks that would appear in mixed shader/fixed-function configurations.

---

## Summary of the Blending Pipeline

With all changes applied, the terrain normal blending works as follows:

1. **Each texture layer pass** is compiled with `writeNormals = 1`, so every pass samples and writes its normal map.
2. **Blend state** on all MRTs uses proper alpha-over compositing (`SRC_ALPHA, ONE_MINUS_SRC_ALPHA`), not additive accumulation.
3. **The normal MRT is not disabled** — it participates in the same blending as diffuse.
4. **Height-biased alpha** from the parallax texture creates organic transitions that simultaneously blend diffuse textures and their normal maps.
5. **Two outputs** — the blended per-pixel normal and the smooth vertex normal — give post-processing shaders access to both mapped and smooth terrain normals.
6. **Post-processing shaders** must normalize the result from `omw_SamplerNormals` for terrain pixels, since alpha-compositing unit vectors does not preserve magnitude.

---

## Files Not Changed (Relevant to Normals)

- `terrain.vert` — vertex shader unchanged between fork and upstream
- `esmterrain/storage.cpp/hpp` — normal map texture loading unchanged; the blending fix is entirely in the render pass setup and fragment shader
- Post-processing buffer infrastructure (`postprocessor.hpp/cpp`, `pingpongcanvas.hpp/cpp`) — already correct in upstream for the buffer setup; the problem was exclusively in how terrain wrote into those buffers
