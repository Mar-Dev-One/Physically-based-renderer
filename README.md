# PBR-Renderer

A physically based renderer and game engine in C11 and OpenGL 4.6, built as a static library,
with a testbed and a game runtime.

- **Rendering**: Cook-Torrance GGX metallic-roughness shading, image-based lighting from HDRIs
  (multi-scatter, specular occlusion), cascaded shadow maps with filtered soft shadows, HDR
  rendering with ACES tone mapping and exposure, directional / point / spot lights.
- **Scene**: an entity-component world with generational handles, reflected components, a
  reference-counted asset cache, and JSON scene files.
- **Commands**: every scene change goes through one JSON command API, with validation, clear
  errors, and undo/redo grouped into transactions.
- **Remote control**: a localhost WebSocket server speaking JSON-RPC 2.0, with change
  notifications.

The project is a work in progress; see [docs/ROADMAP.md](docs/ROADMAP.md) for what's done and
what's next.

## Building

Requirements: CMake 3.10+, a C11 and C++17 compiler (C++ is only for Dear ImGui), and a GPU with
OpenGL 4.6.

```sh
git clone --recursive https://github.com/Mar-Dev-One/PBR-Renderer.git
cd PBR-Renderer
cmake -S . -B build
cmake --build build --config Release
```

If you cloned without `--recursive`, run `git submodule update --init` first (GLFW, cglm and
cimgui are submodules; the other dependencies are vendored in `engine/deps/`).

This builds the engine as one static library, `PBR_Engine`, and two programs on it. Both find
their files at runtime (the engine's shaders, and a project folder), so they can be started from
any directory.

- **Testbed** (`build/testbed/Release/Testbed.exe` with Visual Studio): builds a scene in code
  with the World API -- a ground plane, a grid of spheres from smooth to rough (dielectric and
  metal), a polished Suzanne turning on a pedestal, a sun, a point light and a spot light under an
  HDRI sky -- and draws it. Left-drag orbits, the wheel zooms, Esc quits. `--check PNG` renders it
  once in a hidden window and saves the picture; `--view NAME` shows a debug view (albedo,
  normals, roughness, metallic, ao, shadow, lighting, wireframe, lights). Its
  [main.c](testbed/main.c) is the shortest example of a program using the engine.
- **Game** (`build/game/Release/Game.exe`): the game runtime, what an export ships. It plays a
  project's scene with no UI, console or command server:
  `Game [--width N] [--height N] [--fullscreen] [--project DIR] [--check PNG] [scene] [hdri]`, where
  `scene` is a name under the project's `scenes/` (default: the project's startup scene) or a path
  to a `.json` file, and `hdri` is a `.hdr` under the project that overrides the scene's own. The
  project is `--project`, else a `project.json` next to the program (an exported game), else
  `assets/`. The scene file carries its environment (HDRI, exposure, ambient strength, skybox). It
  views the scene through its first Camera entity; without one, the camera frames the scene's
  objects (not its ground plane), left-drag orbits and the wheel zooms. The window's title and
  size come from the project. The game runs the scene's behaviours: a `Spin` component turns its
  entity at `degrees_per_second`. Esc quits. A Debug build is a console program, so its log shows
  in the terminal; a Release build opens no console and writes `log.txt` next to itself, and if it
  has to close (no OpenGL 4.6, a startup scene that doesn't load) it says why in a message box
  first.

**Exporting a game.** `project_export()` ([engine/src/Project/Project.h](engine/src/Project/Project.h))
makes a folder a player can run: the Game renamed after the project, the C++ runtime DLLs it
needs, the engine's shaders, `project.json`, every saved scene, and only the models and HDRIs
those scenes use. Build the Release game first so the export runs on PCs without Visual Studio:

```sh
cmake --build build --config Release --target Game
```

## Using the engine

A program fills in an `App` ([engine/src/App.h](engine/src/App.h)) with `on_init`, `on_frame`,
`on_resize` and `on_shutdown` hooks and calls `run()`. In them it builds a `world`
([World/World.h](engine/src/World/World.h)) -- `world_create()`, then components such as
`world_add_mesh_renderer()` (a model from the asset cache, or `assets_builtin()` shapes) and
`world_add_light()` -- or loads one from a scene file (`deserialize_scene()`), and each frame calls
`world_update()` and `scene_renderer_draw()` ([Renderer/SceneRenderer.h](engine/src/Renderer/SceneRenderer.h)).
[testbed/main.c](testbed/main.c) does exactly that.

**Projects.** A project is a folder with a `project.json` (name, startup scene, window size,
input bindings) and `scenes/`, `models/`, `hdri/` and `textures/`; every path in a scene is
relative to it. [assets/](assets/) is the sample project, and [assets/scenes/](assets/scenes/)
holds two example scenes.

**Commands.** Every scene change can also go through one JSON command API
([Commands/Commands.h](engine/src/Commands/Commands.h)), with validation, clear errors and
undo/redo grouped into transactions:

```json
{"cmd": "place_model", "args": {"model": "builtin:sphere", "size": 0.3, "on_ground": true}}
```

Commands cover entities and components (`entity.create`, `component.set`, ...), high-level
placement (`place_model`, `entity.place` relative to other entities), scene files (`scene.save`,
`scene.load`, `scene.list`), undo/redo (`history.*`) and batches; `scene.describe` shows the
current world. A program can add commands of its own (`command_context.host_commands`).

**Remote control.** `server_start()` ([Net/CommandServer.h](engine/src/Net/CommandServer.h)) opens
a command server on `127.0.0.1` and writes how to reach it (port and token) to
`~/.pbr-renderer/server.json`. Any WebSocket client can connect to
`ws://127.0.0.1:<port>/?token=<token>` and send the same commands as JSON-RPC 2.0; every change is
pushed back to connected clients as a `scene.changed` notification. See
[tests/server_test.py](tests/server_test.py) for a stdlib-only Python client.

## Models

`model_load(path)` ([engine/src/Scene/Model.h](engine/src/Scene/Model.h)) picks a loader by file
extension:

| Format | Loader | Notes |
|---|---|---|
| `.gltf` / `.glb` | [cgltf](https://github.com/jkuhlmann/cgltf) | Scene hierarchy, multiple meshes and materials, embedded or external textures, PBR metallic-roughness (base color, metal/rough, normal, occlusion, emissive), alpha mask/blend, double-sided. |
| `.obj` (+ `.mtl`) | [fast_obj](https://github.com/thisistherk/fast_obj) | Vertices are welded; `Kd`/`Ke`/`Ns`/`d` and `map_Kd`/`map_Ke` become a PBR material. |

Both are converted to the same `model` / `material` / `mesh` types and drawn with
`engine/shaders/pbr.*`. Not supported yet: skinning and animation, morph targets, Draco/meshopt
compression, `KHR_texture_transform`, and second UV sets.

To use a model, put it in the project's `models/` and load it with `assets_load_model()` (or name
it in a scene file). HDRIs work the same way: an equirectangular `.hdr` in the project's `hdri/`. [Poly Haven](https://polyhaven.com/hdris)
is a good source.

## Project layout

```
engine/src/
  App, Platform/   window, main loop, input
  RHI/             rendering hardware interface (OpenGL backend)
  Renderer/        scene renderer, lighting, shadows, IBL baking, post-processing
  Scene/           camera, lights, materials, meshes, model loaders
  World/           entity-component world, reflection, JSON serialization
  Assets/          shared, reference-counted asset cache
  Commands/        command API, undo/redo history, JSON-RPC
  Net/             localhost WebSocket command server
  UI/              ImGui layer, entity panel, inspector, command console, log
engine/shaders/    the engine's GLSL shaders
testbed/           builds a scene in code and draws it -- the engine used as a library
game/              the game runtime (plays a project's scene; no UI) -- what an export ships
assets/            the default project: models, HDRIs, textures, scenes
tests/             headless C tests and end-to-end server tests
docs/              roadmap
```

## Tests

```sh
ctest --test-dir build -C Release --output-on-failure
ctest --test-dir build -C Release -LE gpu      # without the GPU test, e.g. on a build server
```

The C tests are headless (no window or GL context), so they run anywhere. They cover the world,
serialization, commands, undo history, placement, JSON-RPC and encoding. With Python 3 on the
path, ctest also runs an end-to-end test of the WebSocket server.

`GoldenTests` (label `gpu`) renders the reference scenes in `tests/golden/scenes/` with the game's
`--check` and compares them with the saved pictures in `tests/golden/`. A failure leaves the render
and a diff (changed pixels in red) in `build/tests/golden/`. When a rendering change is intended,
save new pictures, look at them, and commit them:

```sh
build/tests/Release/GoldenTests --update            # every case
build/tests/Release/GoldenTests --update avocado    # some
```

The saved pictures come from one GPU; on another, small rounding differences may need new ones.

## Dependencies

[GLFW](https://github.com/glfw/glfw), [glad](https://github.com/Dav1dde/glad),
[cglm](https://github.com/recp/cglm), [Dear ImGui](https://github.com/ocornut/imgui) via
[cimgui](https://github.com/cimgui/cimgui), [stb_image / stb_image_write](https://github.com/nothings/stb),
[cgltf](https://github.com/jkuhlmann/cgltf), [fast_obj](https://github.com/thisistherk/fast_obj),
[cJSON](https://github.com/DaveGamble/cJSON).

## License

[MIT](LICENSE)
