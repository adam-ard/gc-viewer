# Geometry Central Surface Viewer

A small C++17 surface mesh viewer using Geometry Central for mesh loading and
topology, and Polyscope for visualization.

## Requirements

- CMake 3.20 or newer
- A C++17 compiler
- clang-format
- OpenGL development libraries
- X11 or Wayland development libraries required by GLFW on Linux

Geometry Central and Polyscope are fetched automatically when installed packages
are not found. To use installed packages instead, configure with
`-DGeometryCentral_DIR=<path>` and `-Dpolyscope_DIR=<path>`.

## Build

Format all generated C++ sources and headers after tangling:

```bash {name=format menu=true}
find src -type f \( -name '*.cpp' -o -name '*.h' -o -name '*.hpp' \) \
    -exec clang-format -i {} +
```

```bash {name=build menu=true}
cmake --preset debug
cmake --build --preset debug
```

## Run

```bash {name=app menu=true}
./build/debug/gc_surface_viewer assets/stanford_bunny.off
```

To validate the Ricci-flow bridge with the known-convergent four-cone target
set, run the viewer with the included triangular bunny that has an open
boundary:

```bash {name=ricci-app menu=true}
./build/debug/gc_surface_viewer ricci_flow/data/stanford_bunny_with_hole.obj
```
