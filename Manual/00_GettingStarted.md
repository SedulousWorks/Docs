# Getting Started

This guide walks through creating your first Sedulous application -- a window with a 3D scene, a camera, and a lit object.

## Project Setup

A Sedulous application is a Beef project that depends on the engine libraries. The minimum dependencies are:

```toml
[Dependencies]
corlib = "*"
"Sedulous.Engine.App" = "*"
"Sedulous.Engine.Core" = "*"
"Sedulous.Engine.Render" = "*"
"Sedulous.Runtime" = "*"
"Sedulous.Renderer" = "*"
"Sedulous.RHI" = "*"
"Sedulous.Core.Mathematics" = "*"
"Sedulous.Resources" = "*"
"Sedulous.Images.IO" = "*"
"Sedulous.Images.STB" = "*"
```

You also need one RHI backend:

```toml
"Sedulous.RHI.Vulkan" = "*"   # or
"Sedulous.RHI.DX12" = "*"
```

## Entry Point

The engine uses `EngineApplication` as the base class for all applications. Override its lifecycle methods to configure and run your game.

```beef
using Sedulous.Engine.App;

class MyGame : EngineApplication
{
    protected override void OnStartup()
    {
        // Set up your scene here
    }
}
```

The entry point creates and runs the application with settings:

```beef
class Program
{
    static int Main(String[] args)
    {
        let app = scope MyGame();
        return app.Run(.() {
            Title = "My Game",
            Width = 1280,
            Height = 720,
            Backend = .Vulkan
        });
    }
}
```

## Application Lifecycle

`EngineApplication` provides these virtual methods, called in order:

| Method | When | Purpose |
|--------|------|---------|
| `OnConfigure(Context)` | Before subsystem startup | Register custom subsystems |
| `OnStartup()` | After all subsystems initialized | Create scenes, load resources, set up entities |
| `OnUpdate(float deltaTime)` | Every frame | Game logic |
| `OnShutdown()` | Before cleanup | Release resources |

The engine handles device creation, swap chain, frame pacing, and presentation automatically. You don't need to manage the render loop.

The engine manages a set of subsystems (rendering, physics, audio, etc.) that you can access when needed:

```beef
let sceneSub = Context.GetSubsystem<SceneSubsystem>();
let audioSub = Context.GetSubsystem<AudioSubsystem>();
```

For details on the subsystem architecture and how to create your own, see [Engine Architecture](11_EngineArchitecture.md).

## Initializing Image Loaders

Before loading any textures or models, register at least one image loader. This is typically done at the start of `OnStartup()`:

```beef
using Sedulous.Images.STB;

protected override void OnStartup()
{
    STBImageLoader.Initialize();
    // ...
}
```

STB handles `.png`, `.jpg`, `.tga`, `.bmp`, and `.hdr` formats. For additional format support, add `Sedulous.Images.SDL`:

```beef
using Sedulous.Images.SDL;

SDLImageLoader.Initialize();
```

## Creating a Scene

Scenes are created through the `SceneSubsystem`. Each scene gets its own set of component managers, injected automatically by engine subsystems (rendering, physics, animation, etc.) via the `ISceneAware` interface.

```beef
using Sedulous.Engine.Core;
using Sedulous.Engine;

let sceneSub = Context.GetSubsystem<SceneSubsystem>();
let scene = sceneSub.CreateScene("MyScene");
```

## Creating Entities

Entities are lightweight handles in the scene. They have a transform (position, rotation, scale) and can hold components.

```beef
let entity = scene.CreateEntity("MyCube");

scene.SetLocalTransform(entity, .() {
    Position = .(0, 1, 0),
    Rotation = .Identity,
    Scale = .One
});
```

## Adding a Camera

Every scene needs at least one camera to render. Create an entity, then add a `CameraComponent` via the camera manager:

```beef
using Sedulous.Engine.Render;

let cameraEntity = scene.CreateEntity("Camera");
scene.SetLocalTransform(cameraEntity, Transform.CreateLookAt(
    .(0, 4, 8),     // position
    .(0, 0, 0)      // look-at target
));

let cameraMgr = scene.GetModule<CameraComponentManager>();
let camHandle = cameraMgr.CreateComponent(cameraEntity);
if (let cam = cameraMgr.Get(camHandle))
{
    cam.IsActive = true;
    cam.FieldOfView = 60.0f;
    cam.NearPlane = 0.1f;
    cam.FarPlane = 1000.0f;
}
```

## Adding a Light

Without a light, the scene renders black. Add a directional light:

```beef
let lightEntity = scene.CreateEntity("Sun");
scene.SetLocalTransform(lightEntity, .() {
    Position = .(0, 10, 0),
    Rotation = Quaternion.CreateFromYawPitchRoll(0.3f, -0.8f, 0),
    Scale = .One
});

let lightMgr = scene.GetModule<LightComponentManager>();
let lightHandle = lightMgr.CreateComponent(lightEntity);
if (let light = lightMgr.Get(lightHandle))
{
    light.Type = .Directional;
    light.Color = .(1, 1, 1);
    light.Intensity = 2.0f;
    light.CastShadows = true;
}
```

## Adding a Mesh

To display a 3D object, create a mesh resource and attach it to an entity via `MeshComponentManager`:

```beef
using Sedulous.Geometry.Resources;

// Create a procedural cube mesh
let cubeRes = StaticMeshResource.CreateCube(1, 1, 1);
ResourceSystem.AddResource(cubeRes);

let cubeEntity = scene.CreateEntity("Cube");
scene.SetLocalTransform(cubeEntity, .() {
    Position = .(0, 0.5f, 0),
    Rotation = .Identity,
    Scale = .One
});

let meshMgr = scene.GetModule<MeshComponentManager>();
let meshHandle = meshMgr.CreateComponent(cubeEntity);
if (let mesh = meshMgr.Get(meshHandle))
{
    let meshRef = ResourceRef(.Empty, cubeRes.Name);
    defer meshRef.Dispose();
    mesh.SetMeshRef(meshRef);
}
```

## Setting Up Render Settings

Scene-level render settings (sky, ambient light, exposure) are configured via `RenderSceneModule`:

```beef
if (let renderSettings = scene.GetModule<RenderSceneModule>())
{
    renderSettings.AmbientColor = .(0.1f, 0.1f, 0.15f);
    renderSettings.Exposure = 1.0f;
    renderSettings.SkyIntensity = 1.0f;
}
```

To set a sky texture from the builtin registry:

```beef
let skyRef = ResourceRef(.Empty, "builtin://skies/realistic_sky.texture");
defer skyRef.Dispose();
renderSettings.SetSkyTextureRef(skyRef);
```

## Complete Example

Putting it all together -- a minimal application with a lit cube on a ground plane:

```beef
using Sedulous.Engine.App;
using Sedulous.Engine.Core;
using Sedulous.Engine.Render;
using Sedulous.Engine;
using Sedulous.Runtime;
using Sedulous.Core.Mathematics;
using Sedulous.Geometry.Resources;
using Sedulous.Resources;
using Sedulous.Images.STB;

class MinimalApp : EngineApplication
{
    private Scene mScene;

    protected override void OnStartup()
    {
        STBImageLoader.Initialize();

        let sceneSub = Context.GetSubsystem<SceneSubsystem>();
        mScene = sceneSub.CreateScene("Main");

        // Camera
        let cam = mScene.CreateEntity("Camera");
        mScene.SetLocalTransform(cam, Transform.CreateLookAt(.(0, 4, 8), .(0, 0, 0)));
        let cameraMgr = mScene.GetModule<CameraComponentManager>();
        if (let c = cameraMgr.Get(cameraMgr.CreateComponent(cam)))
        {
            c.IsActive = true;
            c.FieldOfView = 60;
        }

        // Light
        let sun = mScene.CreateEntity("Sun");
        mScene.SetLocalTransform(sun, .() {
            Position = .(0, 10, 0),
            Rotation = Quaternion.CreateFromYawPitchRoll(0.3f, -0.8f, 0),
            Scale = .One
        });
        let lightMgr = mScene.GetModule<LightComponentManager>();
        if (let l = lightMgr.Get(lightMgr.CreateComponent(sun)))
        {
            l.Type = .Directional;
            l.Intensity = 2.0f;
            l.CastShadows = true;
        }

        // Ground plane
        let planeRes = StaticMeshResource.CreatePlane(10, 10, 1, 1);
        ResourceSystem.AddResource(planeRes);
        let ground = mScene.CreateEntity("Ground");
        let meshMgr = mScene.GetModule<MeshComponentManager>();
        if (let m = meshMgr.Get(meshMgr.CreateComponent(ground)))
        {
            let r = ResourceRef(.Empty, planeRes.Name);
            defer r.Dispose();
            m.SetMeshRef(r);
        }

        // Cube
        let cubeRes = StaticMeshResource.CreateCube(1, 1, 1);
        ResourceSystem.AddResource(cubeRes);
        let cube = mScene.CreateEntity("Cube");
        mScene.SetLocalTransform(cube, .() {
            Position = .(0, 0.5f, 0), Rotation = .Identity, Scale = .One
        });
        if (let m = meshMgr.Get(meshMgr.CreateComponent(cube)))
        {
            let r = ResourceRef(.Empty, cubeRes.Name);
            defer r.Dispose();
            m.SetMeshRef(r);
        }

        // Render settings
        if (let rs = mScene.GetModule<RenderSceneModule>())
        {
            rs.AmbientColor = .(0.1f, 0.1f, 0.15f);
            rs.Exposure = 1.0f;
        }
    }
}

class Program
{
    static int Main(String[] args)
    {
        let app = scope MinimalApp();
        return app.Run(.() { Title = "Minimal Sedulous App" });
    }
}
```

## Next Steps

- [Scenes and Entities](01_ScenesAndEntities.md) -- entity hierarchy, transforms, component lifecycle
- [Resources](02_Resources.md) -- loading meshes, textures, materials from files
- [Rendering](03_Rendering.md) -- cameras, lights, materials, post-processing
