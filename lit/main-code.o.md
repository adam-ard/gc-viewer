# Read and populate mesh variables

We can either use manifold specific mesh classes or more general ones. For most of the operations we do, we require that the surface be manifold, so that is the default ([cmdline.o.md](cmdline.o.md)). But occassionally it is nice to load a non-manifold model, just to visualize it.

```cpp {name=read-mesh}
if (options.require_manifold)
{
    std::unique_ptr<gcs::ManifoldSurfaceMesh> manifold_mesh;
    std::tie(manifold_mesh, geometry) = gcs::readManifoldSurfaceMesh(options.mesh);
    mesh = std::move(manifold_mesh);
}
else
{
    std::tie(mesh, geometry) = gcs::readSurfaceMesh(options.mesh);
}

print_mesh_stats(*mesh);

const RicciTopologyPreflight topologyPreflight =
    analyze_ricci_topology(*mesh);
print_ricci_topology_preflight(topologyPreflight, std::cout);

const RicciGeometryPreflight geometryPreflight =
    analyze_ricci_geometry(*mesh, *geometry);
print_ricci_geometry_preflight(geometryPreflight, std::cout);
```

# Compute Cross Field

Choose the boundary-aware solver for open meshes, then compute the singularity
index induced by the resulting fourfold field.

```cpp {name=compute-cross-field}
constexpr int fieldSymmetry = 4;
gcs::FaceData<gc::Vector2> field = mesh->hasBoundary()
    ? gcs::computeSmoothestBoundaryAlignedFaceDirectionField(
          *geometry, fieldSymmetry)
    : gcs::computeSmoothestFaceDirectionField(*geometry, fieldSymmetry);

gcs::VertexData<int> frameIndex =
    gcs::computeVertexIndex(*geometry, field, fieldSymmetry);
```

# Add Mesh Quantities

Attach the computed field, its singularity indices, and Gaussian curvature to
the Polyscope mesh.

```cpp {name=add-mesh-quantities}
geometry->requireFaceTangentBasis();
geometry->requireVertexGaussianCurvatures();

gcs::FaceData<gc::Vector3> basisX(*mesh);
gcs::FaceData<gc::Vector3> basisY(*mesh);
for (gcs::Face f : mesh->faces())
{
    basisX[f] = geometry->faceTangentBasis[f][0];
    basisY[f] = geometry->faceTangentBasis[f][1];
}

psMesh->addFaceTangentVectorQuantity(
    "cross field", field, basisX, basisY, fieldSymmetry);

psMesh->addVertexScalarQuantity("vertex singularity index", frameIndex,
                                polyscope::DataType::SYMMETRIC);

psMesh->addVertexScalarQuantity("Gaussian Curvature",
                                geometry->vertexGaussianCurvatures);
```

# Frame-field Diagnostics

```cpp {name=definitions}
void print_frame_field_diagnostics(
    gcs::SurfaceMesh& mesh,
    const gcs::VertexPositionGeometry& geometry,
    const gcs::VertexData<int>& frameIndex,
    int fieldSymmetry)
{
    int frameIndexSum = 0;
    int singularityCount = 0;

    std::cout << "Frame-field singularity vertices:\n";
    for (gcs::Vertex vertex : mesh.vertices())
    {
        const int index = frameIndex[vertex];
        frameIndexSum += index;
        if (index == 0)
        {
            continue;
        }

        const gc::Vector3& position = geometry.vertexPositions[vertex];
        ++singularityCount;
        std::cout << "  vertex " << vertex.getIndex() << ": index " << index
                  << " (fractional "
                  << static_cast<double>(index) / fieldSymmetry
                  << "), position (" << position[0] << ", "
                  << position[1] << ", " << position[2] << ")\n";
    }

    const int eulerCharacteristic =
        static_cast<int>(mesh.nVertices()) -
        static_cast<int>(mesh.nEdges()) +
        static_cast<int>(mesh.nFaces());
    std::cout << "Found " << singularityCount
              << " nonzero frame-field singularity vertices\n"
              << "Frame index sum: " << frameIndexSum
              << " (expected " << fieldSymmetry * eulerCharacteristic
              << " for a closed " << fieldSymmetry
              << "-field, chi = " << eulerCharacteristic << ")\n";
}
```



# Mesh Stats Function

```cpp {name=definitions}
void print_mesh_stats(gcs::SurfaceMesh& mesh)
{
    std::cout << "Loaded mesh: " << mesh.nVertices() << " vertices, "
              << mesh.nEdges() << " edges, " << mesh.nFaces() << " faces";
    if (!mesh.isManifold())
    {
        std::cout << " (nonmanifold)";
    }
    std::cout << '\n';
}
```

# Main Code

```cpp {name=main-code}
std::unique_ptr<gcs::SurfaceMesh> mesh;
std::unique_ptr<gcs::VertexPositionGeometry> geometry;

polyscope::options::programName = "Mesh to Spline";
polyscope::init();

@<read-mesh@>

auto* psMesh = polyscope::registerSurfaceMesh("mesh", 
                                              geometry->vertexPositions, 
											  mesh->getFaceVertexList());

@<compute-cross-field@>
@<add-mesh-quantities@>
print_frame_field_diagnostics(
    *mesh, *geometry, frameIndex, fieldSymmetry);
print_frame_field_cone_targets(
    derive_frame_field_cone_targets(
        *mesh, frameIndex, fieldSymmetry),
    std::cout);

polyscope::state::userCallback =
    make_flatten_callback(
        options.mesh, *mesh, *geometry, frameIndex, fieldSymmetry);

polyscope::show();
```
