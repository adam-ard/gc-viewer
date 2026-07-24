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

Format all generated C++ sources, headers, and tests after tangling:

```bash {name=format menu=true}
find src tests -type f \( -name '*.cpp' -o -name '*.h' -o -name '*.hpp' \) \
    -exec clang-format -i {} +
```

For development and tests, use the Debug preset:

```bash {name=build-debug menu=true}
cmake --preset debug
cmake --build --preset debug
ctest --test-dir build/debug --output-on-failure
```

The numerical solver is substantially faster with compiler optimization. Use
the Release preset for interactive flattening:

```bash {name=build menu=true}
cmake --preset release
cmake --build --preset release
ctest --test-dir build/release --output-on-failure
```

## Run

```bash {name=app menu=true}
./build/release/gc_surface_viewer assets/stanford_bunny.off
```

To exercise the frame-field-to-Ricci workflow, run the viewer with the included
triangular bunny that has an open boundary. The viewer computes a
boundary-aligned fourfold field, converts its nonzero indices to cone-curvature
targets, and uses those targets when **Flatten** is pressed:

```bash {name=ricci-app menu=true}
./build/release/gc_surface_viewer ricci_flow/data/stanford_bunny_with_hole.obj
```

The **Cone singularities** panel lists the frame-field candidates. Click a
colored marker or its row to focus it, use the checkbox to include or exclude
it, and edit its integer index directly before pressing **Flatten**. The panel
reports the active index sum and the boundary curvature required by
Gauss--Bonnet.
