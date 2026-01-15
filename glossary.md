# Glossary

A comprehensive reference of rendering terms used throughout this knowledge base.

## A

**Albedo**: The base color of a surface, representing the intrinsic color without lighting. Also called diffuse color or base color.

**Alpha Testing**: Discarding pixels based on alpha (opacity) value, used for foliage and transparent cutouts.

**Ambient Occlusion (AO)**: Approximation of indirect shadowing caused by nearby geometry blocking ambient light.

**Anisotropic Filtering**: Texture filtering technique that improves quality for surfaces viewed at oblique angles.

**Anti-Aliasing (AA)**: Techniques to reduce jagged edges (aliasing) in rendered images.

**Attenuation**: The falloff or reduction of light intensity with distance.

## B

**BasePass**: The first major rendering pass that generates the GBuffer with material properties.

**Bilinear Filtering**: Simple texture filtering that samples 4 nearest texels and interpolates.

**Bloom**: Post-process effect that creates glow around bright areas.

**BRDF (Bidirectional Reflectance Distribution Function)**: Mathematical function describing how light reflects off a surface.

**BVH (Bounding Volume Hierarchy)**: Spatial acceleration structure for ray tracing and culling.

## C

**Cascaded Shadow Maps (CSM)**: Shadow technique for directional lights using multiple shadow maps at different distances.

**Clipmap**: Level-of-detail technique using concentric regions of increasing size and decreasing detail.

**Cluster**: Small group of triangles (~128) used in Nanite's geometry organization.

**Compute Shader**: Programmable shader stage for general GPU computation (not tied to graphics pipeline).

**CSM**: See Cascaded Shadow Maps.

## D

**DDC (Derived Data Cache)**: Cache storing compiled shaders, textures, and other derived assets to speed up iteration.

**Deferred Rendering**: Rendering technique that separates geometry and lighting into separate passes using a GBuffer.

**Depth Buffer (Z-Buffer)**: Stores per-pixel depth values for visibility determination.

**Depth of Field (DOF)**: Camera effect simulating focus, blurring objects based on distance from focal plane.

**Diffuse**: Non-shiny surface reflection, scattering light in all directions.

**Distance Field**: Volumetric representation storing distance to nearest surface at each point.

**DXR (DirectX Raytracing)**: Microsoft's API for hardware-accelerated ray tracing.

## E

**Early-Z**: Hardware optimization that discards hidden pixels before pixel shader execution.

**Emissive**: Self-illuminating material property, doesn't require lights.

**Environment Map**: Texture (often cubemap) capturing surrounding environment for reflections.

## F

**Forward Rendering**: Traditional rendering where lighting is computed for each object during geometry rendering.

**Fresnel Effect**: Increased reflectivity at grazing angles (e.g., water edges appear more reflective).

**Frustum**: View pyramid defining camera's visible region.

**Frustum Culling**: Optimization removing objects outside camera frustum.

## G

**GBuffer (Geometry Buffer)**: Set of textures storing material properties (albedo, normals, roughness, etc.) in deferred rendering.

**GGX**: Microfacet BRDF model used for physically-based specular reflections.

**Global Illumination (GI)**: Indirect lighting from light bouncing off surfaces.

**GPU (Graphics Processing Unit)**: Specialized processor for parallel graphics and compute operations.

## H

**HDR (High Dynamic Range)**: Image representation with luminance values beyond display range (0-1).

**HLSL (High-Level Shading Language)**: Shader programming language for DirectX.

**HZB (Hierarchical Z-Buffer)**: Mipmap chain of depth buffer used for efficient occlusion culling.

## I

**IBL (Image-Based Lighting)**: Lighting technique using environment maps for ambient illumination.

**Indirect Lighting**: Light that has bounced off surfaces (vs. direct from light sources).

**Instance**: Copy of a mesh with different transform (position, rotation, scale).

**Instancing**: Rendering many instances of same mesh efficiently.

## L

**LDR (Low Dynamic Range)**: Standard image representation with values in 0-1 range.

**Level of Detail (LOD)**: Simplified versions of mesh for distant rendering.

**Lightmap**: Baked texture storing precomputed lighting.

**LOD**: See Level of Detail.

**Lumen**: UE5's dynamic global illumination system.

**Luminance**: Brightness of a color, typically computed as weighted sum of RGB.

## M

**Material**: Asset defining surface appearance (shaders, textures, parameters).

**Mesh**: 3D model composed of vertices, edges, and faces.

**Metallic**: Material property (0-1) indicating whether surface is metallic or dielectric.

**Mipmap**: Pre-filtered texture at decreasing resolutions for efficient distant sampling.

**Motion Blur**: Effect simulating camera exposure time by blurring based on motion.

**Motion Vectors**: Per-pixel velocity information for temporal effects.

## N

**Nanite**: UE5's virtualized geometry system for rendering massive triangle counts.

**Normal Map**: Texture encoding per-pixel surface orientation for detail without geometry.

**Normal Vector**: Direction perpendicular to a surface.

## O

**Occlusion**: Blocking of visibility or light by geometry.

**Occlusion Culling**: Optimization removing objects hidden behind other geometry.

**Opacity**: Transparency value (0=transparent, 1=opaque).

**Overdraw**: Multiple pixels shaded at same screen location (wasted work).

## P

**PBR (Physically-Based Rendering)**: Rendering approach using real-world physical properties for materials and lighting.

**PCF (Percentage Closer Filtering)**: Shadow map filtering technique for soft shadow edges.

**Pixel Shader (Fragment Shader)**: Shader stage computing color of pixels.

**Point Light**: Omnidirectional light source at a position.

**Post-Processing**: Effects applied after scene rendering (bloom, tone mapping, etc.).

## R

**Radiance**: Light energy flowing in a specific direction.

**Radiance Cache**: Lumen's sparse 3D grid storing indirect lighting.

**Ray Tracing**: Rendering technique using ray-geometry intersection for accurate lighting/shadows/reflections.

**RDG (Render Dependency Graph)**: UE5's rendering framework for automatic resource management.

**Reflection**: Mirror-like light bounce off surfaces.

**Render Target**: Texture that can be rendered to (as opposed to sampled from).

**Roughness**: Material property (0-1) controlling surface microsurface detail (0=smooth/shiny, 1=rough/matte).

## S

**Shader**: Program running on GPU for rendering (vertex shader, pixel shader, compute shader, etc.).

**Shading Model**: Algorithm defining how materials interact with light (e.g., Default Lit, Subsurface, Cloth).

**Shadow Map**: Depth texture from light's perspective for shadow determination.

**Specular**: Shiny, mirror-like reflection (opposite of diffuse).

**Spotlight**: Cone-shaped directional light source.

**SSAO (Screen Space Ambient Occlusion)**: AO computed using screen-space depth and normals.

**SSR (Screen Space Reflections)**: Reflections computed using screen-space data.

**Surface Cache**: Lumen's system caching surface appearance for efficient GI.

## T

**TAA (Temporal Anti-Aliasing)**: Anti-aliasing using data from previous frames.

**Tessellation**: Dynamically subdividing geometry for additional detail.

**Texture**: Image data sampled in shaders (albedo textures, normal maps, etc.).

**Tone Mapping**: Process converting HDR to LDR for display.

**Translucency**: Partial transparency allowing light to pass through.

**TSR (Temporal Super Resolution)**: UE5's temporal upsampling and anti-aliasing technique.

## U

**UAV (Unordered Access View)**: Resource that can be randomly read/written in compute shaders.

**UV Coordinates**: 2D coordinates for texture mapping (typically 0-1 range).

## V

**Velocity Buffer**: Texture storing per-pixel motion vectors.

**Vertex**: Point in 3D space, basic building block of meshes.

**Vertex Factory**: System abstracting how vertex data is fetched and processed.

**Vertex Shader**: Shader stage transforming vertex positions and attributes.

**View**: Camera configuration (position, orientation, projection).

**Virtual Shadow Maps (VSM)**: UE5's page-based shadow system for high-resolution shadows.

**Visibility Buffer**: Rendering technique storing primitive IDs instead of shaded colors.

**Voxel**: Volumetric pixel, 3D equivalent of a 2D pixel.

## W

**World Position Offset (WPO)**: Material feature allowing vertex displacement in world space.

**World Space**: Coordinate system relative to scene origin (vs. object space or view space).

## Z

**Z-Buffer**: See Depth Buffer.

**Z-Fighting**: Visual artifact when two surfaces occupy same depth (flickering).

---

## Acronym Quick Reference

- **AA**: Anti-Aliasing
- **AO**: Ambient Occlusion
- **BRDF**: Bidirectional Reflectance Distribution Function
- **CSM**: Cascaded Shadow Maps
- **DDC**: Derived Data Cache
- **DOF**: Depth of Field
- **DXR**: DirectX Raytracing
- **GI**: Global Illumination
- **HLSL**: High-Level Shading Language
- **HZB**: Hierarchical Z-Buffer
- **IBL**: Image-Based Lighting
- **LOD**: Level of Detail
- **PBR**: Physically-Based Rendering
- **PCF**: Percentage Closer Filtering
- **RDG**: Render Dependency Graph
- **SSAO**: Screen Space Ambient Occlusion
- **SSR**: Screen Space Reflections
- **TAA**: Temporal Anti-Aliasing
- **TSR**: Temporal Super Resolution
- **UAV**: Unordered Access View
- **VSM**: Virtual Shadow Maps
- **WPO**: World Position Offset
