# Geometry Central Surface Viewer

A literate C++17 surface viewer that uses Geometry Central for mesh topology,
preflight analysis, and frame fields; Polyscope for interactive visualization;
and Ricci flow for curvature-driven flattening.

## Requirements

- CMake 3.20 or newer
- A C++17 compiler
- clang-format
- `omd`, the literate-programming tool used to generate this project
- OpenGL development libraries
- X11 or Wayland development libraries required by GLFW on Linux

Geometry Central and Polyscope are fetched automatically when installed packages
are not found. To use installed packages instead, configure with
`-DGeometryCentral_DIR=<path>` and `-Dpolyscope_DIR=<path>`.

## Ricci-flow dependency

The Ricci-flow solver is maintained in a separate repository and is not fetched
by CMake. After cloning this viewer, clone the solver into the `ricci_flow`
directory expected by `add_subdirectory()`:

```bash
git clone https://github.com/kendrickshepherd/ricci_flow.git ricci_flow
```

The resulting layout should contain `ricci_flow/CMakeLists.txt` alongside this
project's top-level `CMakeLists.txt`.

## Generate

The `.o.md` files are the source of truth. This task tangles every generated
C++ source, header, and test, then applies the repository's Chromium-based
format:

```omd {name=generate menu=true}
tangle
run format
```

Run it after changing a literate section:

```bash
omd run generate
```

## Build

The formatting operation used by `generate` is also available independently:

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

The default mode runs the complete manifold workflow, including preflight,
surface analysis, cone selection, and flatten controls:

```bash {name=app menu=true}
./build/release/gc_surface_viewer ricci_flow/data/duck_with_two_holes.obj
```

Passing `--nonmanifold` selects inspection-only mode. It uses Geometry
Central's general mesh reader and omits direction fields, curvature quantities,
cone targets, and flatten controls—even when the supplied mesh happens to be
manifold:

```bash {name=app-nonmanifold menu=true}
./build/release/gc_surface_viewer --nonmanifold ricci_flow/data/duck_with_two_holes.obj
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
