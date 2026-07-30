# Executable Assembly

This file is the top-level outline of the viewer executable. It assembles
feature-specific definitions and dependencies from the literate sections that
explain them; the only implementation logic kept here is the process boundary:
parse the command line, run the viewer, and translate exceptions into a
nonzero exit status.

# Viewer Pipeline :CARD:

After command-line validation, the viewer lifecycle owns mesh and window setup,
while surface analysis owns the geometric quantities derived from that mesh.
Reading the callouts from top to bottom gives the executable's operational
story. Each stage leaves the data needed by the stages below it in the shared
scope of `main()`.

The default manifold mode runs the complete analysis and flattening workflow.
The `--nonmanifold` option deliberately selects an inspection-only path:
Polyscope displays the loaded mesh, but algorithms whose connectivity
assumptions require a manifold are not constructed or exposed in the UI.

```cpp {name=pipeline-code}
@<initialize-viewer-state@>

@<load-mesh@>

@<inspect-mesh@>

@<initialize-polyscope@>
@<register-mesh@>

if (options.require_manifold) {
  @<compute-cross-field@>
  @<add-mesh-quantities@>

  @<report-frame-field@>

  @<configure-flattening@>
}

@<show-viewer@>
```

# Includes :CARD:

Each contributing section owns the headers required by its code. Repeated
headers are intentional: local dependency ownership is more useful here than
deduplicating a harmless preprocessor operation.

```cpp {name=main-includes}
@<cmdline-includes@>
@<surface-analysis-includes@>
@<viewer-includes@>

<cstdlib>
<exception>
<iostream>
```

# Namespace Aliases :CARD:

These aliases apply to the entire generated `main.cpp` translation unit. They
keep Geometry Central's mathematical and surface-mesh types readable throughout
the executable without importing individual library names into the global
namespace.

```cpp {name=main-namespace-aliases}
namespace gc = geometrycentral;
namespace gcs = geometrycentral::surface;
```

# File-local Definitions :CARD:

Each contributing section owns its private diagnostic helpers. This aggregate
keeps their details in the files that explain them while giving the executable
assembly a single definition callout.

```cpp {name=main-defs}
@<cmdline-defs@>
@<surface-analysis-defs@>
@<viewer-defs@>
```

# Main File :CARD:

Expected command-line mistakes are still reported locally by the parser.
Placing that work inside the outer `try` also ensures that any unexpected
parser failure remains inside the executable's single exception boundary.

The helper definitions assembled below are implementation details of the
executable. The anonymous namespace gives them internal linkage, preventing
their names from colliding with definitions in other translation units.
`main()` remains outside because C++ requires it in the global namespace.

```cpp {name=main tangle=src/main.cpp}
#include @<main-includes@>

@<main-namespace-aliases@>

namespace {
@<main-defs@>
}  // namespace

int main(int argc, char** argv) {
  try {
    @<handle-cmdline@>
    @<pipeline-code@>
  } catch (const std::exception& error) {
    std::cerr << "Error: " << error.what() << '\n';
    return EXIT_FAILURE;
  }

  return EXIT_SUCCESS;
}
```
