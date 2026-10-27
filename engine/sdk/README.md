# PBR Engine SDK

The engine as a static library, with the libraries it uses next to it and every header a program
needs. Made by `cmake --install` from the engine's source tree (see its README).

```
include/          the engine's headers, plus cglm/, cJSON.h, GLFW/, glad/, KHR/, cimgui.h
lib/Debug/        PBR_Engine.lib  glfw3.lib  glad.lib  cimgui.lib
lib/Release/      the same, built for Release
engine/shaders/   the engine's shaders, loaded at runtime
```

## Using it in a project (Visual Studio)

- **C/C++ > General > Additional Include Directories:** `<sdk>\include`
- **Linker > General > Additional Library Directories:** `<sdk>\lib\$(Configuration)`
- **Linker > Input > Additional Dependencies:**
  `PBR_Engine.lib;glfw3.lib;glad.lib;opengl32.lib`, and also
  - `cimgui.lib` if the program uses the ImGui layer or panels (`UI/`),
  - `ws2_32.lib` if it uses the command server (`Net/CommandServer.h`).
- **Platform:** x64. Debug builds link `lib\Debug`, Release builds `lib\Release`: the two use
  different C runtimes (`/MDd` and `/MD`), and mixing them fails with `LNK2038`.
- The engine is C11: compile the program's C files as C11 or later (`/std:c11`).

A program that calls cimgui's `ig*()` functions itself also defines
`CIMGUI_DEFINE_ENUMS_AND_STRUCTS=1`, `IMGUI_DISABLE_OBSOLETE_FUNCTIONS=1`, `CIMGUI_USE_GLFW=1`,
`CIMGUI_USE_OPENGL3=1` and `IMGUI_IMPL_API=extern "C" __declspec(dllexport)`, as the library was
built with them.

## At runtime

- **Shaders:** copy `engine\` (the folder holding `shaders\`) next to the program's `.exe`; the
  engine looks for `<exe folder>\engine\shaders` first. A post-build step does it:
  `xcopy /E /I /Y "<sdk>\engine" "$(OutDir)engine"`
- **The program's files** (models, HDRIs, scenes) live in a project folder: `models\`, `hdri\`,
  `scenes\`, `textures\`. Point the engine at it before `run()`:
  `paths_set_project_root("assets");` -- every path the program loads (`"models/thing.glb"`,
  `"hdri/sky.hdr"`) is relative to it.

## A minimal program

```c
#include "App.h"
#include "Assets/AssetCache.h"
#include "Core/Paths.h"
#include "Platform/Window.h"
#include "Renderer/Renderer.h"
#include "Renderer/SceneRenderer.h"
#include "Scene/Camera.h"
#include "World/Environment.h"
#include "World/World.h"

static asset_cache    assets;
static world          scene;
static scene_renderer scene_view;
static camera         cam;

static void on_init(App* app)
{
    window_size size = window_get_size(get_renderer()->drawing_window);
    scene_renderer_create(&scene_view, size.width, size.height, 4, 2048);

    assets_init(&assets);
    world_init(&scene, &assets);

    entity_id ball = world_create(&scene, "ball");
    world_add_mesh_renderer(&scene, ball, assets_builtin(&assets, BUILTIN_SPHERE));

    light sun;
    light_init_directional(&sun, (vec3){ -0.4f, -1.0f, -0.5f }, (vec3){ 1.0f, 1.0f, 1.0f }, 3.0f);
    world_add_light(&scene, world_create(&scene, "sun"), sun);

    camera_init(&cam, (vec3){ 0.0f, 1.0f, 3.0f }, (vec3){ 0.0f, 0.0f, 0.0f }, 45.0f,
                (f32)size.width / (f32)size.height, 0.1f, 100.0f);
}

static void on_frame(App* app, f32 dt)
{
    world_update(&scene);
    scene_render_settings settings = scene_render_settings_default();
    scene_environment     env      = scene_environment_default();
    scene_render_settings_apply_environment(&settings, &env, NULL);
    settings.delta_time = dt;
    scene_renderer_draw(&scene_view, &scene, &cam, &settings);
}

static void on_shutdown(App* app)
{
    world_shutdown(&scene);
    assets_shutdown(&assets);
    scene_renderer_destroy(&scene_view);
}

int main(void)
{
    paths_set_project_root("assets");
    App app = { .name = "My game", .width = 1280, .height = 720,
                .on_init = on_init, .on_frame = on_frame, .on_shutdown = on_shutdown };
    run(&app);
    terminate(&app);
    return 0;
}
```

The engine's source tree has a fuller example: `testbed/main.c` (an orbit camera, an HDRI, the
material grid, shadows and a `--check` mode that saves a picture).
