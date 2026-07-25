# Viewer Lifecycle and Mesh Setup

This section owns the application shell around the geometric analysis. It
creates the long-lived mesh and geometry objects, initializes Polyscope, loads
and inspects the requested mesh, registers it for display, and finally enters
the interactive event loop.

# Dependencies

Mesh loading uses Geometry Central's general and manifold-specific readers.
Polyscope owns the window and registered surface, while the flatten and
preflight interfaces connect those objects to the rest of the application.
Types shared with surface analysis are supplied by the pipeline's common
dependencies.

```cpp {name=viewer-includes}
"geometrycentral/surface/manifold_surface_mesh.h"
"geometrycentral/surface/meshio.h"
```

# File-local definitions

```cpp {name=viewer-defs}
@<mesh-stats-defs@>
```

# Initialize the viewer

The mesh and its geometry share the lifetime of `main()`. Geometry Central
returns them as owning pointers, allowing the application to select a concrete
mesh type at load time while the remaining pipeline uses the common
`SurfaceMesh` interface.

Polyscope must be initialized before any surface is registered.

```cpp {name=initialize-viewer}
std::unique_ptr<gcs::SurfaceMesh> mesh;
std::unique_ptr<gcs::VertexPositionGeometry> geometry;

polyscope::options::programName = "Mesh to Spline";
polyscope::init();
```

# Load the mesh

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

# Inspect the loaded mesh

Loading establishes a valid Geometry Central representation; it does not imply
that the mesh satisfies every Ricci-flow assumption. The topology and geometry
preflights therefore produce explicit reports before more expensive analysis
begins. The flatten action later enforces these reports when solver readiness
becomes mandatory.

```cpp {name=inspect-mesh}
print_mesh_stats(*mesh);

const RicciTopologyPreflight topologyPreflight = analyze_ricci_topology(*mesh);
print_ricci_topology_preflight(topologyPreflight, std::cout);

const RicciGeometryPreflight geometryPreflight =
    analyze_ricci_geometry(*mesh, *geometry);
print_ricci_geometry_preflight(geometryPreflight, std::cout);
```

# Register the display surface

Polyscope consumes vertex positions and a face-vertex list. The returned
surface handle is retained because surface analysis attaches quantities to this
same displayed mesh.

```cpp {name=register-mesh}
auto* psMesh = polyscope::registerSurfaceMesh(
    "mesh", geometry->vertexPositions, mesh->getFaceVertexList());
```

# Enter the interactive viewer

The callback captures the long-lived mesh, geometry, and frame-field data. It
owns the flatten controls and launches solver work in response to UI actions.
`polyscope::show()` then runs the event loop until the window closes.

```cpp {name=run-interactive-viewer}
polyscope::state::userCallback = make_flatten_callback(
    options.mesh, *mesh, *geometry, frameIndex, fieldSymmetry);

polyscope::show();
```

# Mesh summary helper

This compact summary records the element counts associated with the in-memory
Geometry Central mesh. The manifold marker is especially useful when the
general reader was requested.

```cpp {name=mesh-stats-defs}
void print_mesh_stats(gcs::SurfaceMesh& mesh) {
  std::cout << "Loaded mesh: " << mesh.nVertices() << " vertices, "
            << mesh.nEdges() << " edges, " << mesh.nFaces() << " faces";
  if (!mesh.isManifold()) {
    std::cout << " (nonmanifold)";
  }
  std::cout << '\n';
}
```
