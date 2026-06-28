# AirSim Code Tour

This file is a quick orientation guide for this checkout of AirSim. It focuses on where the main subsystems live, how API calls move through the stack, and which files usually need to change for common work.

## High-Level Shape

AirSim is split into a platform-independent C++ core and simulator-specific front ends.

1. Client code calls the RPC API from Python, C++, C#, or another msgpack-rpc client.
2. `AirLib` receives and models API calls, vehicles, sensors, physics, settings, and data structures.
3. `Unreal/Plugins/AirSim` adapts the core to Unreal Engine actors, pawns, cameras, rendering, weather, and world objects.
4. Vehicle-specific implementations connect the shared API surface to multirotors, cars, Computer Vision mode, PX4, ArduPilot, or SimpleFlight.

The docs that already cover the official architecture are:

- `docs/code_structure.md`
- `docs/design.md`
- `docs/dev_workflow.md`
- `docs/adding_new_apis.md`

## Repository Map

| Path | Purpose |
| --- | --- |
| `AirLib/` | Platform-independent C++ library: APIs, physics, sensors, settings, safety, vehicles, RPC client/server code. |
| `Unreal/Plugins/AirSim/` | Unreal Engine plugin implementation: sim modes, pawns, cameras, rendering, sensors, recording, weather, object control. |
| `PythonClient/` | Python package and sample scripts for AirSim RPC APIs. Main client binding is `PythonClient/airsim/client.py`. |
| `MavLinkCom/` | Standalone MAVLink communication library used by PX4 and ArduPilot integrations. |
| `HelloDrone/`, `HelloCar/`, `DroneShell/`, `DroneServer/` | Native sample and utility programs. |
| `ros/`, `ros2/` | ROS integration packages and wrappers. |
| `Unity/` | Experimental Unity integration. |
| `docs/` | MkDocs documentation source. Navigation is configured by `mkdocs.yml`. |
| `AirLibUnitTests/` | Native unit test project for core AirLib behavior. |
| `azure/`, `docker/`, `pipelines/` | Cloud, container, and CI support. |

## Core AirLib Areas

`AirLib` is the C++ core that should remain independent of Unreal-specific classes.

- API base classes: `AirLib/include/api/`
- RPC server/client implementation: `AirLib/src/api/RpcLibServerBase.cpp`, `AirLib/src/api/RpcLibClientBase.cpp`
- RPC adaptors and shared structs: `AirLib/include/api/RpcLibAdaptorsBase.hpp`, `AirLib/include/common/CommonStructs.hpp`
- Settings parsing: `AirLib/include/common/AirSimSettings.hpp`
- Physics: `AirLib/include/physics/`
- Sensors: `AirLib/include/sensors/`
- Multirotor models and APIs: `AirLib/include/vehicles/multirotor/`, `AirLib/src/vehicles/multirotor/`
- Car APIs and firmware adapters: `AirLib/include/vehicles/car/`, `AirLib/src/vehicles/car/`
- Safety and geofence logic: `AirLib/include/safety/`, `AirLib/src/safety/`

When adding fields to `settings.json`, start at `AirLib/include/common/AirSimSettings.hpp`.

## Unreal Plugin Areas

`Unreal/Plugins/AirSim/Source` is the Unreal-facing layer.

- Plugin module and game mode: `AirSim.cpp`, `AirSim.h`, `AirSimGameMode.cpp`
- Base sim-mode lifecycle: `SimMode/SimModeBase.*`, `SimMode/SimModeWorldBase.*`
- Vehicle modes:
  - Multirotor: `Vehicles/Multirotor/`
  - Car: `Vehicles/Car/`
  - Computer Vision mode: `Vehicles/ComputerVision/`
- Environment and world APIs: `WorldSimApi.*`
- Camera and image capture: `PIPCamera.*`, `UnrealImageCapture.*`, `RenderRequest.*`
- Sensor adapters backed by Unreal scene data: `UnrealSensors/`
- Recording: `Recording/`
- Weather: `Weather/`
- Common Unreal helpers: `AirBlueprintLib.*`, `NedTransform.*`

Most environment-only APIs eventually land in `WorldSimApi.cpp`. Most camera behavior eventually lands in `PIPCamera.cpp`, `UnrealImageCapture.cpp`, or `RenderRequest.cpp`.

## RPC Call Flow

For a typical Python API call:

1. `PythonClient/airsim/client.py` exposes a method and calls `self.client.call(...)`.
2. `AirLib/src/api/RpcLibServerBase.cpp` binds that RPC method name to a C++ lambda.
3. The lambda delegates to a vehicle API or `WorldSimApiBase`.
4. Unreal implementations in `Unreal/Plugins/AirSim/Source` perform the simulator work.
5. Result structs are converted through RPC adaptor types, then returned to the client.

For C++ clients, the public methods are declared in `AirLib/include/api/RpcLibClientBase.hpp` and implemented in `AirLib/src/api/RpcLibClientBase.cpp`.

Vehicle-specific RPC surfaces extend this pattern:

- Multirotor RPC: `AirLib/include/vehicles/multirotor/api/`, `AirLib/src/vehicles/multirotor/api/`
- Car RPC: `AirLib/include/vehicles/car/api/`, `AirLib/src/vehicles/car/api/`

## Adding or Changing an API

Use `docs/adding_new_apis.md` for the full upstream checklist. The usual file set is:

1. Implement core behavior in the relevant AirLib or Unreal class.
2. Add or update an abstract method if the call belongs on `WorldSimApiBase` or a vehicle API base.
3. Bind the RPC method in `RpcLibServerBase.cpp` or the vehicle-specific RPC server.
4. Add the C++ client wrapper in `RpcLibClientBase.hpp` and `RpcLibClientBase.cpp`.
5. Add Python bindings in `PythonClient/airsim/client.py`.
6. Add or update msgpack adaptor structs if the API returns a new structured type.
7. Add docs and an example script when the API is user-facing.

If the API is vehicle-control-specific, check the multirotor or car API base classes first. If it changes the scene, cameras, weather, objects, textures, or levels, check `WorldSimApi.*` first.

## CinemAirSim Notes

This checkout includes CinemAirSim camera extensions marked with `//CinemAirSim`.

Important files:

- `Unreal/Plugins/AirSim/Source/PIPCamera.h`
- `Unreal/Plugins/AirSim/Source/PIPCamera.cpp`
- `Unreal/Plugins/AirSim/Source/WorldSimApi.h`
- `Unreal/Plugins/AirSim/Source/WorldSimApi.cpp`
- `AirLib/include/api/WorldSimApiBase.hpp`
- `AirLib/include/api/RpcLibClientBase.hpp`
- `AirLib/src/api/RpcLibClientBase.cpp`
- `AirLib/src/api/RpcLibServerBase.cpp`
- `PythonClient/airsim/client.py`

The main change is that `APIPCamera` derives from `ACineCameraActor` and uses `UCineCameraComponent` for lens, filmback, focal length, manual focus, focus distance, aperture, focus plane, and current field-of-view controls. If you touch these APIs, keep the declarations, RPC binding names, C++ client methods, Unreal implementation, and Python wrapper in sync.

## Build and Local Workflow

Windows is the most mature development path for this Unreal-based repo.

- Build AirSim plugin from the repo root: `build.cmd`
- Linux build path: `setup.sh`, then `build.sh`
- Clean/rebuild helpers: `clean.cmd`, `clean_rebuild.bat`, `clean.sh`, `clean_rebuild.sh`
- Documentation build helpers: `build_docs.bat`, `build_docs.sh`

For Unreal plugin development, the documented workflow is:

1. Build the plugin from the AirSim repo root.
2. Deploy `Unreal/Plugins` into an Unreal environment such as Blocks.
3. Regenerate project files for the Unreal project.
4. Build and run from Visual Studio.

See `docs/dev_workflow.md` and `docs/unreal_blocks.md` for details.

## Testing Touchpoints

- Native core tests: `AirLibUnitTests/`
- Python smoke examples:
  - `PythonClient/multirotor/hello_drone.py`
  - `PythonClient/car/hello_car.py`
  - `PythonClient/computer_vision/cv_mode.py`
- Camera/image API examples:
  - `PythonClient/computer_vision/external_camera.py`
  - `PythonClient/computer_vision/fov_change.py`
  - `PythonClient/multirotor/gimbal.py`

When testing Python changes from source, scripts usually import local package paths through `setup_path.py` so they use the checkout version instead of an installed PyPI package.

## Good Starting Points

- Understanding settings: `AirLib/include/common/AirSimSettings.hpp`
- Understanding world APIs: `AirLib/include/api/WorldSimApiBase.hpp`, `Unreal/Plugins/AirSim/Source/WorldSimApi.cpp`
- Understanding RPC exposure: `AirLib/src/api/RpcLibServerBase.cpp`
- Understanding client wrappers: `AirLib/src/api/RpcLibClientBase.cpp`, `PythonClient/airsim/client.py`
- Understanding cameras: `Unreal/Plugins/AirSim/Source/PIPCamera.cpp`, `Unreal/Plugins/AirSim/Source/UnrealImageCapture.cpp`
- Understanding simulation modes: `Unreal/Plugins/AirSim/Source/SimMode/`
- Understanding multirotor behavior: `AirLib/include/vehicles/multirotor/`, `Unreal/Plugins/AirSim/Source/Vehicles/Multirotor/`
- Understanding car behavior: `AirLib/include/vehicles/car/`, `Unreal/Plugins/AirSim/Source/Vehicles/Car/`
