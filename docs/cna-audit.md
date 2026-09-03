# Phase 1 — audit of CNA and sharp-runtime

Everything below was read out of the checkouts, not recalled from documentation.
It is the basis for every architectural decision in `docs/architecture.md`.

## Baselines

| Repository | Branch | HEAD | Notes |
| --- | --- | --- | --- |
| `openeggbert/cna` | `next` | `756096626cfb537e32ac8ccc6b0743a23c57ddfd` | 9 000 files, ~258 MB |
| `openeggbert/sharp-runtime` | `next` | `bd282d101640005454639b372f67e119ffa5642b` | 4 112 files, ~86 MB |
| `openeggbert/easy-gl` | `develop` | `deda7a426c3c166c0e03a4790f1ede610e2e46fb` | backs the GL renderer family |
| `openeggbert/meta-gl` | `develop` | `252017322b5e7c4711fcf13810e8b6da05fd0511` | easy-gl's own sibling |

`dependencies.lock` pins the same revisions.

## What CNA is

An XNA 4.0-shaped game framework re-implemented in C++23, on top of a C++
re-implementation of the .NET base class library (sharp-runtime). The public
surface is literally `Microsoft::Xna::Framework::...`, and everything CNA adds
beyond the XNA 4.0 contract is marked `CNAEXT` in the headers and gated behind
the `CNA_CNAEXT` compile option (`-DCNA_CNAEXT=ON`, **off by default**).

Two facts shape this project more than any other:

1. **`CNA_CNAEXT=OFF` hides the entire modern renderer.** Every
   `CNA/Graphics/*.hpp` header is wrapped in `#ifdef CNA_CNAEXT`, so without the
   option they compile to nothing at all. cna-street forces it ON.
2. **CNA exports no CMake package.** It is consumed with `add_subdirectory()`
   from a sibling checkout, and it resolves sharp-runtime, easy-gl and meta-gl
   relative to its own source root.

### Module inventory (`modules/`)

| Module | What cna-street takes from it |
| --- | --- |
| `core` | `CNA::Logger`, log levels/categories, renderer identity enums |
| `math` | `Vector2/3/4`, `Matrix`, `Quaternion`, `Color`, `Rectangle`, `Point`, `Plane`, `Ray`, `BoundingBox`, `BoundingSphere`, `BoundingFrustum`, `MathHelper`, `Curve` |
| `runtime` | `Game`, `GameTime`, `GameWindow`, `GraphicsDeviceManager`, `GameComponent`, `LaunchParameters` |
| `graphics` | `GraphicsDevice`, `Texture2D`, `TextureCube`, `RenderTarget2D`, `VertexBuffer`, `IndexBuffer`, `DynamicVertexBuffer`, `Model`/`ModelMesh`/`ModelMeshPart`, `BasicEffect`, `PbrEffect`, `ShaderEffect`, `SpriteBatch`, `SpriteFont`, all the render states |
| `graphics-ext` | the modern renderer — see below |
| `content` | `ContentManager`, the CNB format and codecs, the content pipeline (`GltfImporter`, `ContentCompiler`, `Texture2DContentPipeline`, …) |
| `input` | `Keyboard`, `Mouse` (incl. relative-mouse mode), `GamePad`, `Touch` |
| `platform` | `IPlatform`, `PlatformCapabilities` — OS/window/clock abstraction |
| `renderers` | 36 renderer back ends; cna-street targets the `easygl` family (`OPENGL33`/`OPENGLES3`) and can build against any 3D-capable one |
| `audio`, `media`, `storage`, `net`, `devices` | not central here; audio is used for ambience |

### The modern graphics layer (`modules/graphics-ext`, `CNA::Graphics`)

97 public headers. Grouped by what they do:

**Frame orchestration**
`RenderPipeline` (HDR scene target + the whole post chain, `begin`/`end` around
the game's own draws), `RenderPipelineSettings` (~50 knobs, plus
`toStringEXT()`/`applyFromStringEXT()` — a ready-made settings-file format),
`RenderQuality`, `PostProcessChain`, `PostProcessPass`, `PostProcessContext`,
`RenderTargetPool`, `ScopedRenderTarget`, `BlitPass`, `FullscreenPass`.

**Post-process passes** `BloomPass`, `TonemapPass` (`TonemappingMode`:
Reinhard/ACES/…), `FxaaPass`, `SsaoPass`, `SsrPass`, `DepthOfFieldPass`,
`ColorGradePass` + `CubeLut`, `ChromaticAberrationPass`, `FilmGrainPass`,
`LensFlarePass`, `MotionBlurPass`, `HeightFogPass`, `VolumetricFogPass`,
`LightShaftPass`, `AerialPerspectivePass`, `SpatialUpscalePass`,
`AutoExposureEXT`, `HdrDisplayOutput`, `CRTEffect`, `AsciiPass`.

**Shadows** `ShadowMap` (single), `CascadedShadowMap` (up to 4 cascades,
practical split scheme with `splitLambda`, texel-grid snapping, cascade blend
band, debug tint), `SpotShadowMap`, `CubeShadowMap`, `ContactShadowPass`,
`ShadowQuality`.

**Lighting** `DirectionalLightEXT`, `PointLightEXT`, `SpotLightEXT`,
`ClusteredLightEXT` + `ClusteredLightGrid`/`Buffer`/`Compute`/`Assignment` +
`ClusteredForwardEffect`, `LightProbeEXT`, `LightProbeVolumeEXT`,
`LightProbeBaker`, `EnvironmentProcessor` (irradiance / prefiltered specular /
BRDF LUT generation → IBL), `AreaLightShading`.

**Sky** `AtmosphericSky` (analytic sky radiance from sun direction + turbidity,
also callable on the CPU via the static `radiance()`), `Skybox` (cubemap with
yaw/intensity/tint).

**Geometry throughput** `InstancedRendererEXT` (matrix stream + optional
per-instance colour tint stream, with a non-instanced fallback),
`FrustumCullerEXT`, `LodGroupEXT` (distance or screen-space-error selection with
hysteresis), `GpuInstanceCuller`, `DepthNormalPrepass`, `TransparentDrawList`,
`WeightedBlendedTransparency`, `DecalPass`, `ParticleSystem`.

**Explicit GPU resources** `StorageBuffer`, `ComputeShader`, `GpuTimer`,
`ShaderEffectFactory`, `ShaderDiagnostics`, `MaterialBinding`, `DebugDraw`,
`DebugGizmos`, `EngineLayerVersion`.

**Materials** `PbrMaterial`, `PbrMaterialExtensions`, `GltfMaterialBridge`,
`ThinFilmIridescence`.

### `PbrEffect` — the material contract

A full glTF 2.0 metallic-roughness material, not a toy:

* base colour factor + texture (with an sRGB flag), alpha, `AlphaModeEXT`
  (Opaque/Mask/Blend) + alpha cutoff, double-sided flag;
* normal map + normal scale; metallic/roughness factors + combined map;
* emissive factor + map (+ sRGB flag); occlusion map + strength;
* `KHR_materials_specular` (specular factor/colour + maps) and
  `KHR_materials_ior`;
* per-slot **texture coordinate set** and per-slot **`TextureTransformEXT`**
  (offset / scale / rotation) — i.e. `KHR_texture_transform`, which is what makes
  correct per-surface tiling possible without duplicating textures;
* vertex colours (`VertexColorEnabledEXT`);
* shadow reception: `setShadowMapEXT`, `setLightViewProjectionEXT`,
  `setShadowCascadesEXT(ShadowCascadeStateEXT)`, filter radius, depth bias;
* `setImageBasedLightEXT(ImageBasedLightEXT)` — irradiance + prefiltered
  specular + BRDF LUT;
* `setPunctualLightEXT(PunctualLightEXT)`;
* `setEncodeOutputToSrgbEXTProperty` for gamma-correct output.

`BasicEffect` implements the same `IShadowReceiverEXT` shadow interface, so
cheap materials can still receive the same cascades.

### Capability model

`GraphicsDevice` reports what the selected renderer can actually do, and the
whole EXT layer degrades rather than throwing:

* `SupportsCapability(GraphicsCapability)` — `ThreeD`, `Instancing`,
  `ComputeShaders`, `FloatRenderTargets`, `MultipleRenderTargets`,
  `IndirectDraw`, `CustomEffects`, … (20 values);
* `SupportsRendererFeatureEXT(RendererFeature)` — a finer 30-value set that also
  covers `ShaderEffectSourceExecution`, `ShadowSampling`, `ImageBasedLighting`,
  `GpuTimers` and the shader dialects;
* `GetRendererLimitEXT(RendererLimit)` — max texture size, vertex streams,
  compute work-group sizes;
* `GetRendererCapabilityReportEXT()` — a printable report;
* `SupportsSurfaceFormatAsRenderTargetEXT(SurfaceFormat)`.

cna-street queries these once at start-up and disables the passes the renderer
cannot run, rather than assuming a back end.

### Content pipeline

Headless and deterministic — no GraphicsDevice, no display, no clock:

* **CNB** is CNA's own binary asset container (`CNA/Content/Cnb/*`): texture,
  model (v1 + v2), animation clip, sprite font, sound effect, curve codecs, CRC32C
  integrity, optional zstd chunk compression.
* `cna_tool_source_to_cnb` — `png|jpg|bmp|tga|hdr|wav|…` → `.cnb`.
* `cna_tool_gltf_to_cnb` — `.gltf`/`.glb` → `.cnb`, through the same
  glTF→CNJ orchestration the rest of the pipeline uses.
* `ContentManager::Load<T>(name)` resolves `.xnb` then `.cnb`.
* XNB (the original XNA container) is fully supported for reading, which is not
  needed here.

That gives a build-from-source asset path with no XNA Game Studio, no MonoGame
and no Visual Studio content tools — exactly what this project needs.

## What sharp-runtime is

A C++ implementation of the .NET base class library, 44 modules, consumed as
`SharpRuntime::<Component>` CMake targets. CNA declares its own component
closure in `cmake/SharpRuntimeConsumption.cmake`; a consumer that wants more has
to extend `SHARP_RUNTIME_COMPONENTS` **before** CNA is added.

cna-street uses, and `docs/architecture.md` records why:

| Component | Use |
| --- | --- |
| `System.Text.Json` (`Text.Json`) | the data-driven scene description and the settings file are parsed with `JsonDocument`/`JsonElement` |
| `System.Random` (`Core.Base`) | every procedural placement decision, so the city is reproducible from a seed |
| `System.Diagnostics.Stopwatch` | frame and phase timing behind the debug overlay |
| `System.IO` (`IO`) | asset path resolution and manifest reading |
| `System.Text.StringBuilder` | overlay text assembly |

## Renderers available on Linux

`SDL_RENDERER` (2D only), `SDL_GPU`, `OPENGLES2/3`, `OPENGL33` (easy-gl family),
`OPENGL1/2/4`, `OPENGLES1`, `VULKAN`, `WEBGPU`, `HEADLESS`, `SOFTWARE`, `STUB`,
`PORTABLEGL`, `BGFX`, `MAGNUM`, `SOKOL`, `DILIGENT`, `LLGL`, `FNA3D`, plus the
2D raster ones. cna-street defaults to **`OPENGL33`**: it is 3D-capable,
executes shader source (which the whole EXT layer needs), and runs on Mesa's
software rasteriser under Xvfb, which is how the screenshots in this repository
are produced.
