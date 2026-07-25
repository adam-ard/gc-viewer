# Viewer Pipeline

This file is the readable outline of the viewer's work after command-line
validation. It deliberately contains orchestration rather than implementation:
the viewer lifecycle owns mesh and window setup, while surface analysis owns
the geometric quantities derived from that mesh.

# Pipeline assembly

Reading the callouts from top to bottom gives the executable's operational
story. Each stage leaves the data needed by the stages below it in the shared
scope of `main()`.

```cpp {name=pipeline-code}
@<initialize-viewer@>

@<load-mesh@>

@<inspect-mesh@>

@<register-mesh@>

@<compute-cross-field@>
@<add-mesh-quantities@>

@<report-frame-field@>

@<run-interactive-viewer@>
```

# Dependencies

The pipeline combines specialized dependencies from its two contributing
sections with the Geometry Central and Polyscope types shared by both.

```cpp {name=pipeline-includes}
@<surface-analysis-includes@>
@<viewer-includes@>
"geometrycentral/surface/surface_mesh.h"
"geometrycentral/surface/vertex_position_geometry.h"

"polyscope/polyscope.h"
"polyscope/surface_mesh.h"

<iostream>
<memory>
<tuple>
```

# Namespace aliases

These aliases apply to the entire generated `main.cpp` translation unit. They
keep Geometry Central's mathematical and surface-mesh types readable throughout
the pipeline without importing individual library names into the global
namespace.

```cpp {name=pipeline-namespace-aliases}
namespace gc = geometrycentral;
namespace gcs = geometrycentral::surface;
```

# File-local definitions

Each contributing section owns its private diagnostic helpers. This aggregate
gives [main.o.md](main.o.md) one definition callout and keeps the detailed
categories within the files that explain them.

```cpp {name=pipeline-defs}
@<surface-analysis-defs@>
@<viewer-defs@>
```
