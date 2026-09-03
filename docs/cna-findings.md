# CNA findings

Everything this project ran into in CNA and sharp-runtime that cost time,
produced a wrong picture, or required a workaround. Each entry says what
happened, how to reproduce it, what was done here, and what a framework-level
fix would look like.

Nothing here is a complaint about CNA being incomplete. A framework whose
extension layer already contains cascaded shadow maps, clustered lights, an
analytic sky and a glTF-grade PBR material is a long way past "incomplete"; what
follows is what a real application finds when it uses all of that at once.

Baselines: CNA `next` `7560966`, sharp-runtime `next` `bd282d1`, easy-gl
`develop` `deda7a4`, meta-gl `develop` `2520173`. Renderer `OPENGL33` (EasyGL)
on Mesa 25.2 / llvmpipe.

---

## CNA-F1 — vendored single-header include paths resolve into the consumer

**Severity:** high (a consumer cannot build at all without a workaround)
**Affected:** `modules/content/CMakeLists.txt` (and anything else adding
`${CMAKE_SOURCE_DIR}/third_party/...`)

**What happens.** CNA's content module adds its vendored `cgltf` and `stb`
include directories as `${CMAKE_SOURCE_DIR}/third_party/...`. Under
`add_subdirectory()`, `CMAKE_SOURCE_DIR` is the *consumer's* root, so the paths
point at directories that do not exist and the build fails with
`fatal error: cgltf.h: No such file or directory`.

**Reproduction.** Any project that consumes CNA with `add_subdirectory()` and
builds `cna_content`.

**Workaround here.** `cmake/CnaStreetDependencies.cmake` sets the correct paths
with `include_directories()` before `add_subdirectory()`, relying on the fact
that the property is inherited by subdirectories.

**Proposed fix.** Use `${CMAKE_CURRENT_SOURCE_DIR}` (or a `CNA_ROOT`-style
variable captured at CNA's own top level) instead of `${CMAKE_SOURCE_DIR}`.
The same substitution fixes every occurrence.

**Fixed upstream during this work:** no — CNA is consumed read-only here.

---

## CNA-F2 — the module-layout validator tests the consumer's tree

**Severity:** medium
**Affected:** `modules/CMakeLists.txt`

**What happens.** CNA aborts the configure if `${CMAKE_SOURCE_DIR}/src` or
`/include` exists, with a message about a "legacy global `src/` tree
reappear[ing] at the repository root". Under `add_subdirectory()` that is the
consumer's root, so any project using the conventional `src/` + `include/`
layout is refused with a message about CNA's own history.

**Workaround here.** Application sources live under `street/`, and the reason is
stated at the top of `CMakeLists.txt` so nobody moves them back.

**Proposed fix.** Same substitution as CNA-F1.

---

## CNA-F3 — the vendored SDL build cache defaults inside the CNA checkout

**Severity:** low
**Affected:** `cmake/ThirdPartySDL.cmake` (`CNA_SDL_PREBUILT_ROOT`)

**What happens.** The persistent SDL build cache defaults to a directory inside
CNA's own source tree, which writes into a dependency a consumer may reasonably
treat as read-only (a submodule, a shared checkout, a read-only mount).

**Workaround here.** `CNA_SDL_PREBUILT_ROOT` is pointed at the consumer's build
directory, keyed by toolchain.

**Proposed fix.** Default it under `CMAKE_BINARY_DIR`.

---

## CNA-F4 — Linux prerequisites are larger than documented

**Severity:** low (documentation)

**What happens.** A clean Ubuntu 24.04 image needs `libxss-dev`,
`libxkbcommon-dev`, `wayland-protocols`, `libdecor-0-dev`, `libdbus-1-dev`,
`libudev-dev`, `libibus-1.0-dev` and the Mesa/X11 development packages before
the vendored SDL3 will configure; the failure is an SDL `SDL_missing_dependency`
error rather than anything naming CNA. `libzstd-dev` is separately required for
`.cnb` chunk compression and its absence is only a `STATUS` line.

**Proposed fix.** List the full set in the README's prerequisites, and promote
the zstd message to a warning since it silently changes what the content
pipeline can produce.

---

## CNA-F5 — the front-face winding convention is not documented

**Severity:** medium (silent wrong output)

**What happens.** Nothing states which winding CNA's default
`RasterizerState::CullCounterClockwise` keeps. The answer, determined
empirically against `OPENGL33`, is that a face is visible when its vertices are
**clockwise seen from the front** — i.e. `cross(b-a, c-a)` points *away* from
the viewer. That is XNA-faithful and the opposite of the glTF/OpenGL convention,
so geometry authored to the usual convention is invisible, and geometry authored
to it *by accident* is visible but lit from behind.

**Reproduction.** Draw one quad four ways (two windings × two cull modes) and
count the pixels. `docs/architecture.md` records the result.

**Workaround here.** `MeshBuilder::addTriangleIndices` is the single place the
readable counter-clockwise-from-front convention becomes CNA's order, and
`addQuadFacing` takes an intended normal where the corner order is derived from
something else. Both exist because the first version of the road markings and
the roof slopes came out facing into the ground: visible, and lit from below.

**Proposed fix.** State it in the `RasterizerState` documentation, with the
`cross(b-a, c-a)` form rather than "clockwise", which is ambiguous without a
stated handedness.

---

## CNA-F6 — the shadow caster program has no instanced variant

**Severity:** medium (performance)
**Affected:** `CNA::Graphics::ShadowMap`, `CNA::Graphics::CascadedShadowMap`

**What happens.** The caster vertex program takes its world matrix from the
`uWorld` uniform and does not declare the instance-transform attributes the
renderer binds at locations 12–15. So geometry drawn in one call by
`InstancedRendererEXT` cannot be cast in one call: every instance needs its own
`SetUniformMat4("uWorld", …)` and its own draw.

**Consequence here.** Street furniture — lamp columns, bollards, bins, signs,
trees, parked cars — is instanced for the main pass and drawn one at a time in
the shadow pass. That is bounded only by a distance limit (`propShadowDistance`,
default 74 m), which is honest but is a limit the main pass does not need.

**Proposed fix.** Compile the caster with the same
`CNA_GL_INSTANCE_TRANSFORM_DECL` block the stock programs use and multiply
through `cnaInstancePosition`. `uCnaInstanced` already gates it, so the
non-instanced path is unchanged.

---

## CNA-F7 — `AtmosphericSky` renders vertically mirrored

**Severity:** high (visibly wrong output)
**Affected:** `modules/graphics-ext/src/AtmosphericSky.cpp`

**What happens.** The sky's fragment program reconstructs a view ray with

```glsl
vec4 ray = uInverseViewProjection * vec4(TexCoord * 2.0 - 1.0, 1.0, 1.0);
```

`FullscreenPass` draws through `SpriteBatch`, whose texture-coordinate origin is
the **top** left of the destination rectangle, while clip space has +1 at the
top. The reconstructed ray is therefore mirrored in Y: the zenith is rendered
underfoot and the ground haze overhead. It is easy to miss because a clear sky
is nearly symmetric about the horizon — until the sun disc appears in the wrong
half of the frame, which is exactly how this was found.

**Reproduction.** Draw `AtmosphericSky` with the sun above the horizon and look
at where the sun ends up.

**Workaround here.** `SkySystem`'s own fragment program flips
`TexCoord.y` before the reconstruction.

**Proposed fix.** `vec2 screen = vec2(TexCoord.x, 1.0 - TexCoord.y);` in
`kFragmentBody`. One line, no API change.

---

## CNA-F8 — `PbrEffect` sRGB-encodes its output by default

**Severity:** high (silently wrong exposure in any HDR pipeline)
**Affected:** `PbrEffect::getEncodeOutputToSrgbEXTProperty` (defaults to `true`)

**What happens.** With the default, the shader's last act is
`FragColor.rgb = cnaLinearToSrgb(FragColor.rgb)`. That is right when the
fragment lands in the back buffer. It is wrong when it lands in
`RenderPipeline`'s float scene target, because `TonemapPass` then reads an
sRGB-encoded value as if it were linear radiance. Asphalt at 0.05 linear becomes
0.25, the tonemapper compresses what is left, and the whole frame flattens: in
this scene a 10:1 albedo ratio between asphalt and paving rendered as 1.6:1.

The failure is very hard to attribute, because the picture is not obviously
broken — it just looks washed out, which is what a hundred other things also
look like.

**Reproduction.** Wrap any `PbrEffect` draw in `RenderPipeline::begin`/`end`
with `setHDREnabled(true)` and compare a dark and a light material.

**Workaround here.** `SceneRenderer::applyMaterial` sets
`setEncodeOutputToSrgbEXTProperty(!pipeline.isUsingSceneTarget())`.

**Proposed fix.** Either default it to `false` (the encode is the *output
stage's* job and `RenderPipeline` already owns one), or have `RenderPipeline`
publish the expected encoding so effects can ask rather than guess. Documenting
it is the minimum: the property's own comment does not mention tonemapping.

---

## CNA-F9 — the shadow atlas is 8-bit and the default depth bias is below one step

**Severity:** medium
**Affected:** `ShadowMap`/`CascadedShadowMap` (atlas format), `PbrEffect`
(`shadowDepthBiasEXT_ = 0.0015f`)

**What happens.** The shadow map stores light-space distance in an ordinary
colour target — a documented and reasonable choice, since not every renderer can
sample a depth attachment. But that makes one quantisation step `1/255 =
0.0039`, and the default depth bias of `0.0015` is *smaller than one step*. Every
surface facing the sun therefore self-shadows.

**Workaround here.** `shadowDepthBias` defaults to `0.0060`.

**Proposed fix.** Default the bias above one quantisation step, or scale it from
the atlas format. A slope-scaled term would be better still: a constant bias
large enough for a grazing surface is large enough to detach the shadow of a
lamp column from its base.

---

## CNA-F10 — `Texture2D::SaveAsPng` cannot save a render target

**Severity:** low (diagnostics)

**What happens.** `SaveAsPng` on a `RenderTarget2D` throws
`no CPU-side pixel data available`. `GetData` works and reads the target back,
so the capability is there; only the convenience path is missing. Debugging a
shadow atlas is exactly the case where you want the convenience path.

**Workaround here.** `SceneRenderer::dumpShadowAtlas` reads back with `GetData`
and re-uploads through `Texture2D::CreateFromPixels` before saving.

**Proposed fix.** Have `SaveAsPng` fall back to `GetData` when there is no CPU
copy.

---

## CNA-F11 — `AtmosphericSky::getModelGlsl()` needs a header the caller must guess

**Severity:** low (documentation)

**What happens.** The static accessor returns the model body only — no
`#version`, no `precision` qualifier. A caller who pastes it into a
`ShaderEffect` gets a compile error about `in`/`out` qualifiers being invalid in
GLSL 1.10, which does not point anywhere near the cause. `AtmosphericSky`'s own
constructor prepends `"#version 300 es\nprecision highp float;\n"`.

**Proposed fix.** Say so in the doc comment, or expose the prologue as a second
accessor.

---

## CNA-F12 — the content pipeline could not produce a mip chain **(fixed in CNA)**

**Severity:** high — it made the content pipeline worse than not using it

**Affected API.** `cna_tool_source_to_cnb`,
`CNA::Content::Cnb::ImportImageAsCnbTexture2D`.

**What happens.** Every texture compiled by `cna_tool_source_to_cnb` has
exactly one mip level. The container is not the limitation: `TEXH` declares a
mip count, `TEXD` carries one payload per level, `CnbMaxTextureMipLevels` is 16
and the `Texture2D` loader uploads every level it finds. Nothing produced one.

**Reproduction.**

```
cna_tool_source_to_cnb road.png road.cnb --name road
cna_tool_cnb_info road.cnb        # -> Texture2D 512x512 Rgba8, 1 level
```

Then load it and look at a road surface at a grazing angle. The whole
carriageway shimmers, the paving speckles, and the façades crawl — worse than
the same textures generated at run time with a chain built by hand, which is
not a trade anyone should have to make. The screenshots that found this are in
the history of this project: the first content build was visibly worse than the
procedural path it replaced.

**Workaround here.** None was adopted. Generating the chain in the application
after loading would need a `GetData` readback per texture and a second upload,
which throws away most of what the pipeline is for.

**Fix.** Contributed to CNA rather than worked around, because the missing
piece belongs in the codec:
`CNA::Content::Cnb::GenerateRgba8MipChain(CnbTextureData&, CnbMipColorSpace)`
box-filters a single-level Rgba8 description into a complete chain, and
`--mipmaps` on the compiler asks for it. The colour space is an argument
rather than an assumption: averaging four sRGB-encoded texels treats an encoded
value as a quantity of light, so a colour map averaged that way dims as it
recedes, while normals, roughness and masks must be averaged exactly as stored.
`--mip-color-space` defaults to `linear`; this project's content build passes
`srgb` for albedo and emissive maps and leaves the rest alone.

**Where.** Commit `347139500` on branch `feat/cnb-source-mipmaps`, pushed to
`openeggbert/cna`, and kept here as
`docs/patches/0001-feat-cnb-generate-mip-chains-when-compiling-a-source.patch`
so this repository carries its own record of what it asked of the framework. It
carries five GTest cases in CNA's own suite; `tests/ContentPipelineTests.cpp`
here exercises the same function, because a regression in it would silently make
this project's content path worse than its procedural one.

---

## CNA-F13 — the light-shaft pass ghosts, and its sample count is not settable

**Severity:** low — visible, and tunable around

**Affected API.** `RenderPipelineSettings::setLightShaftIntensity` /
`setLightShaftThreshold` / `setLightShaftDecay`.

**What happens.** The radial blur that produces the shafts takes a fixed number
of samples, and `decay` decides how far across the screen those samples are
spread. At a decay near the default-ish 0.965 they are far enough apart that
each one lands as a *discrete* smeared copy of whatever seeded it rather than
blending into a ray.

With a threshold low enough to catch a sunlit façade's window frames — 0.82 in
this project's first attempt — the result is a chain of ghost windows climbing
diagonally across the sky from the sun's screen position. It looks exactly like
floating geometry, and it was diagnosed as floating geometry twice before the
FOV-dependence gave it away: a screen-space artefact moves when the field of
view changes and a building does not.

**Reproduction.** Point a camera up at a sunlit façade with the sun just off
screen, `setLightShaftThreshold(0.82f)`, `setLightShaftDecay(0.965f)`.

**Workaround here.** Threshold 0.995 so only the sun disc and the hottest
speculars seed a shaft, decay 0.86 so the samples overlap, intensity 0.32.

**Proposed fix.** Expose the sample count, or scale it with the decay so that a
long shaft is sampled more finely than a short one. Failing that, say in the
doc comment what decay does to the sampling, because the parameter reads like a
purely aesthetic falloff.

---

## Not defects — behaviour worth knowing

* **`CNA_CNAEXT` defaults to `OFF`.** Every `CNA/Graphics/*.hpp` header is
  wrapped in `#ifdef CNA_CNAEXT`, so with the default the entire modern renderer
  compiles to nothing, with no diagnostic. Worth a `STATUS` line at configure
  time.
* **The capability model is good and should be used.**
  `SupportsCapability`, `SupportsRendererFeatureEXT`, `GetRendererLimitEXT` and
  `GetRendererCapabilityReportEXT` between them answer every question this
  project needed to ask before allocating anything.
* **`RenderPipelineSettings::toStringEXT`/`applyFromStringEXT`** is a ready-made
  settings-file format that deserves to be better advertised.
