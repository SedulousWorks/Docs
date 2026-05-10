# Resources

Resources are engine assets -- meshes, textures, materials, animations, audio clips, and more. The resource system manages loading, caching, reference counting, and hot-reload.

## Resource System Overview

`ResourceSystem` is the central hub, owned by the application and shared across all subsystems. It coordinates:

- **Registries** -- map GUIDs to file paths with protocol prefixes (`builtin://`, `project://`)
- **Managers** -- type-specific loaders (`TextureResourceManager`, `StaticMeshResourceManager`, etc.)
- **Cache** -- prevents duplicate loads of the same file
- **Hot-reload** -- detects file changes and reloads resources

Subsystems register their resource managers during initialization. You don't create the resource system yourself -- `EngineApplication` sets it up.

## Resource Registries

Registries provide a mapping from resource GUIDs to file paths. Each registry has a name that becomes a protocol prefix:

```
builtin://skies/realistic_sky.texture    -> Assets/skies/realistic_sky.texture
project://models/hero.mesh               -> ProjectDir/models/hero.mesh
```

`EngineApplication` automatically loads a `builtin.registry` from the asset directory. The editor creates a `project` registry per project.

## ResourceRef

A `ResourceRef` is a serializable reference to a resource -- a GUID plus a protocol path. Components store these to reference assets.

```beef
// Create a resource ref by path
let meshRef = ResourceRef(.Empty, "builtin://primitives/cube.mesh");
defer meshRef.Dispose();  // ResourceRef owns a String, must be disposed

// Check if valid
if (meshRef.IsValid)
{
    // Has either a GUID or a path
}
```

> **Important**: `ResourceRef` is a struct that owns a heap-allocated `String`. Always dispose it when done, or use `defer`. When storing in a class field, use a field destructor: `private ResourceRef mRef ~ _.Dispose();`

### Setting Resource Refs on Components

Components that reference resources follow a setter pattern that deep-copies the ref:

```beef
let meshRef = ResourceRef(.Empty, "project://models/hero.mesh");
defer meshRef.Dispose();
meshComponent.SetMeshRef(meshRef);
```

The component makes its own copy, so the original can be disposed immediately.

## Loading Resources

Resources are typically loaded implicitly when components resolve their `ResourceRef` during the update loop. You can also load explicitly:

```beef
// Load by path
if (ResourceSystem.LoadResource<TextureResource>(path) case .Ok(let handle))
{
    let texture = handle.Resource;
    // Use texture...
    handle.Release();  // Release when done
}

// Load by ResourceRef
if (ResourceSystem.LoadByRef<StaticMeshResource>(meshRef) case .Ok(let handle))
{
    let mesh = handle.Resource;
    handle.Release();
}
```

### Adding Programmatic Resources

Resources created in code (procedural meshes, generated textures) are added directly:

```beef
let cubeRes = StaticMeshResource.CreateCube(1, 1, 1);
cubeRes.Name.Set("MyCube");
ResourceSystem.AddResource(cubeRes);
```

After adding, the resource can be referenced by name in `ResourceRef`:

```beef
let meshRef = ResourceRef(.Empty, cubeRes.Name);
```

## Resource Types

### Meshes

Static meshes (`.mesh`) and skinned meshes have their own resource types and managers.

**Procedural creation:**

```beef
let plane = StaticMeshResource.CreatePlane(10, 10, 1, 1);
let cube = StaticMeshResource.CreateCube(1, 1, 1);
let sphere = StaticMeshResource.CreateSphere(0.5f, 32, 16);
```

**Loading from model files:**

Model files (`.gltf`, `.glb`, `.fbx`) are imported by the editor or programmatically. The import pipeline converts them into `.mesh`, `.texture`, `.material`, `.skeleton`, and `.animation` files.

```beef
using Sedulous.Models;
using Sedulous.Models.GLTF;

GltfModels.Initialize();  // Register the GLTF loader

if (ModelLoaderFactory.Load(gltfPath) case .Ok(let model))
{
    defer delete model;
    let importer = scope ModelImporter();
    if (importer.Import(model, .Default()) case .Ok(let result))
    {
        // result contains meshes, textures, materials, skeletons, animations
        // Convert to resources and register with ResourceSystem
    }
}
```

### Textures

Textures (`.texture`) wrap an image with sampling parameters (filter, wrap, mipmaps).

**Import via TextureImporter:**

```beef
using Sedulous.Textures.Importer;

// Standard 2D texture
if (TextureImporter.Import2D(imagePath) case .Ok(let texRes))
{
    ResourceSystem.AddResource(texRes);
}

// Equirectangular HDR sky
if (TextureImporter.ImportEquirectangular(hdrPath) case .Ok(let texRes))
{
    ResourceSystem.AddResource(texRes);
}
```

**Texture shapes:**

The `TextureShape` enum describes how pixel data is interpreted:

| Shape | Description |
|-------|-------------|
| `Texture2D` | Standard 2D image |
| `Texture2DArray` | Multiple layers, same dimensions |
| `Texture3D` | Volume texture |
| `Cubemap` | 6 square faces |
| `CubemapArray` | Array of cubemaps |

**Setup presets:**

```beef
texRes.SetupFor3D();                     // Mipmaps, linear, repeat, aniso 16
texRes.SetupForSprite();                 // Nearest, clamp, no mipmaps
texRes.SetupForUI();                     // Linear, clamp, no mipmaps
texRes.SetupForEquirectangularSkybox();  // Linear, clamp, Texture2D
texRes.SetupForCubemapSkybox();          // Linear, clamp, Cubemap
```

### Materials

Materials (`.material`) define surface appearance. The engine uses a PBR metallic-roughness workflow.

```beef
using Sedulous.Materials;

let mat = Materials.CreatePBR("MyMaterial", "forward");
mat.SetColor("BaseColor", .(0.8f, 0.2f, 0.2f, 1));
mat.SetFloat("Roughness", 0.5f);
mat.SetFloat("Metallic", 0.0f);
```

Material instances can override properties from a base material:

```beef
let instance = new MaterialInstance(baseMaterial);
instance.SetColor("BaseColor", .(1, 0, 0, 1));
```

### Animations

- **AnimationClipResource** (`.animation`) -- skeletal animation clips (bone keyframes)
- **SkeletonResource** (`.skeleton`) -- bone hierarchy
- **AnimationGraphResource** (`.animgraph`) -- blend trees and state machines
- **PropertyAnimationClipResource** (`.propanim`) -- property keyframe tracks (position, color, etc.)

### Audio

- **AudioClipResource** (`.audioclip`) -- decoded audio data
- **SoundCueResource** (`.soundcue`) -- audio playback configuration

### Particles

- **ParticleEffectResource** (`.particle`) -- particle system configuration (emitters, behaviors, initializers)

## Serialization

Resources use the `Sedulous.Serialization` framework (OpenDDL format). Each resource file has a header with type, GUID, name, and source path, followed by resource-specific data.

```
ResourceHeader {
    _type = "texture"
    _id = "a1b2c3d4-..."
    _name = "wood_diffuse"
    _sourcePath = "textures/wood_diffuse.texture"
}
// ... resource-specific data
```

Textures use a binary sidecar file (`.texture.bin`) for pixel data.

## Async Loading

Resources can be loaded asynchronously to avoid stalling the main thread:

```beef
ResourceSystem.LoadResourceAsync<TextureResource>(path,
    onCompleted: new (result) => {
        if (result case .Ok(let handle))
        {
            let texture = handle.Resource;
            // Use texture on main thread
        }
    });
```

The job runs on a worker thread. The completion callback fires on the main thread during `ResourceSystem.Update()` (called automatically each frame by `EngineApplication`).

For bulk loading, fire multiple async loads and wait for all to complete:

```beef
for (let path in assetPaths)
    ResourceSystem.LoadResourceAsync<StaticMeshResource>(path);
```

The resource system processes completions each frame. Resources become available as they finish loading.

## Hot-Reload

When `ResourceSystem.EnableHotReload()` is called (done by default in `EngineApplication`), the system watches registered files for changes. Modified resources are automatically reloaded, and components using them update on the next frame.

## Next Steps

- [Rendering](03_Rendering.md) -- cameras, lights, materials, sky, post-processing
- [Audio](05_Audio.md) -- playing sounds and music
