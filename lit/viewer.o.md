# Viewer Lifecycle and Mesh Setup

This section owns the application shell around the geometric analysis. It
creates the long-lived mesh and geometry objects, initializes Polyscope, loads
and inspects the requested mesh, registers it for display, and finally enters
the interactive event loop.

# Dependencies :CARD:

Mesh loading uses Geometry Central's general and manifold-specific readers.
Polyscope owns the window and registered surface, while the flatten and
preflight interfaces connect those objects to the rest of the application.

```cpp {name=viewer-includes}
@<flatten-interface-includes@>
@<mesh-preflight-interface-includes@>
"geometrycentral/surface/manifold_surface_mesh.h"
"geometrycentral/surface/meshio.h"
"geometrycentral/surface/surface_mesh.h"
"geometrycentral/surface/vertex_position_geometry.h"

"polyscope/polyscope.h"
"polyscope/surface_mesh.h"

<iostream>
<memory>
<tuple>
<utility>
```

# File-local definitions :CARD:

The viewer contributes one private helper for the concise mesh summary printed
immediately after loading.

```cpp {name=viewer-defs}
@<mesh-summary-defs@>
```

# Create the long-lived viewer state :CARD:

The mesh and its geometry share the lifetime of `main()`. Geometry Central
returns them as owning pointers, allowing the application to select a concrete
mesh type at load time while the remaining pipeline uses the common
`SurfaceMesh` interface.

Declaration order is significant: local objects are destroyed in reverse
order, so declaring `geometry` after `mesh` guarantees that the geometry object
is destroyed before the mesh it references.

```cpp {name=initialize-viewer-state}
std::unique_ptr<gcs::SurfaceMesh> mesh;
std::unique_ptr<gcs::VertexPositionGeometry> geometry;
```

# Load the mesh :CARD:

Most downstream algorithms require manifold connectivity, so the
manifold-specific Geometry Central reader is the default. The general reader
supports the explicit command-line opt-out used when a mesh only needs to be
inspected.

```cpp {name=load-mesh}
if (options.require_manifold) {
  std::unique_ptr<gcs::ManifoldSurfaceMesh> manifold_mesh;
  std::tie(manifold_mesh, geometry) =
      gcs::readManifoldSurfaceMesh(options.mesh);
  mesh = std::move(manifold_mesh);
} else {
  std::tie(mesh, geometry) = gcs::readSurfaceMesh(options.mesh);
}
```

# Inspect the loaded mesh :CARD:

Loading establishes a valid Geometry Central representation; it does not imply
that the mesh satisfies every Ricci-flow assumption. The topology and geometry
preflights therefore produce explicit reports before more expensive analysis
begins. The flatten action later recomputes and enforces the same conditions
when solver readiness becomes mandatory.

```cpp {name=inspect-mesh}
print_mesh_stats(*mesh);

const RicciTopologyPreflight topologyPreflight = analyze_ricci_topology(*mesh);
print_ricci_topology_preflight(topologyPreflight, std::cout);

const RicciGeometryPreflight geometryPreflight =
    analyze_ricci_geometry(*mesh, *geometry);
print_ricci_geometry_preflight(geometryPreflight, std::cout);
```

# Initialize Polyscope :CARD:

Graphics resources are created only after the mesh has loaded successfully and
its preflight reports have been printed. Polyscope must still be initialized
before any display surface is registered.

```cpp {name=initialize-polyscope}
polyscope::options::programName = "Mesh to Spline";
polyscope::init();
```

# Register the display surface :CARD:

Polyscope consumes vertex positions and a face-vertex list. The returned
pointer is non-owning: Polyscope's global registry owns the display surface.
The pipeline retains the pointer because surface analysis attaches quantities
to this same registered mesh.

```cpp {name=register-mesh}
auto* psMesh = polyscope::registerSurfaceMesh(
    "mesh", geometry->vertexPositions, mesh->getFaceVertexList());
```

# Configure flattening  :CARD:

The callback factory converts the frame indices into editable cone state before
returning. The callback itself retains that state and references the long-lived
mesh and geometry; it does not retain the temporary frame-index object. This
allows all manifold analysis variables to remain local to the conditional
pipeline branch.

```cpp {name=configure-flattening}
polyscope::state::userCallback = make_flatten_callback(
    options.mesh, *mesh, *geometry, frameIndex, fieldSymmetry);
```

# Enter the interactive viewer :CARD:

In manifold mode the registered callback adds the analysis and flattening
controls. Inspection-only mode leaves the callback unset and therefore shows
only Polyscope's standard mesh interface. `polyscope::show()` runs the event
loop until the window closes.

```cpp {name=show-viewer}
polyscope::show();
```

# Mesh summary helper :CARD:

This compact summary records the element counts associated with the in-memory
Geometry Central mesh. The manifold marker is especially useful when the
general reader was requested.

```cpp {name=mesh-summary-defs}
void print_mesh_stats(gcs::SurfaceMesh& mesh) {
  std::cout << "Loaded mesh: " << mesh.nVertices() << " vertices, "
            << mesh.nEdges() << " edges, " << mesh.nFaces() << " faces";
  if (!mesh.isManifold()) {
    std::cout << " (nonmanifold)";
  }
  std::cout << '\n';
}
```
