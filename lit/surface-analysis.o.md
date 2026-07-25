# Surface Analysis and Viewer Quantities

This section derives the geometric information that connects the input mesh to
the later Ricci-flow workflow. A fourfold direction field proposes cone
singularities, while Polyscope quantities expose the field, its indices, and
the mesh's Gaussian curvature for inspection.

# Dependencies

Geometry Central computes the direction field and singularity indices. The cone
target interface interprets those indices for Ricci flow. Shared mesh,
geometry, visualization, and stream types come from the pipeline dependencies.

```cpp {name=surface-analysis-includes}
"geometrycentral/surface/direction_fields.h"
```

# File-local definitions

```cpp {name=surface-analysis-defs}
@<ff-defs@>
```

# Compute a fourfold direction field

An $n$-symmetric direction field represents directions modulo rotations of
$2\pi/n$. Setting $n=4$ produces a cross field suited to quadrilateral layout.
Open meshes use Geometry Central's boundary-aligned solver so the field follows
the surface boundary; closed meshes use the unconstrained smoothest-field
solver.

`computeVertexIndex()` converts the field's circulation around each vertex into
integer index units. Nonzero values identify the singularity candidates later
translated into Ricci cone targets.

```cpp {name=compute-cross-field}
constexpr int fieldSymmetry = 4;
gcs::FaceData<gc::Vector2> field =
    mesh->hasBoundary()
        ? gcs::computeSmoothestBoundaryAlignedFaceDirectionField(*geometry,
                                                                 fieldSymmetry)
        : gcs::computeSmoothestFaceDirectionField(*geometry, fieldSymmetry);

gcs::VertexData<int> frameIndex =
    gcs::computeVertexIndex(*geometry, field, fieldSymmetry);
```

# Attach analysis quantities to Polyscope

Geometry Central expresses the cross field in each face's two-dimensional
tangent coordinates. Polyscope needs the corresponding three-dimensional basis
vectors to draw those directions on the embedded surface, so we request the
face tangent bases and provide both basis vectors with the field.

The vertex index and Gaussian curvature are scalar quantities. Marking the
index as symmetric gives positive and negative singularities a diverging color
map centered at zero.

```cpp {name=add-mesh-quantities}
geometry->requireFaceTangentBasis();
geometry->requireVertexGaussianCurvatures();

gcs::FaceData<gc::Vector3> basisX(*mesh);
gcs::FaceData<gc::Vector3> basisY(*mesh);
for (gcs::Face f : mesh->faces()) {
  basisX[f] = geometry->faceTangentBasis[f][0];
  basisY[f] = geometry->faceTangentBasis[f][1];
}

psMesh->addFaceTangentVectorQuantity(
    "cross field", field, basisX, basisY, fieldSymmetry);

psMesh->addVertexScalarQuantity("vertex singularity index",
                                frameIndex,
                                polyscope::DataType::SYMMETRIC);

psMesh->addVertexScalarQuantity("Gaussian Curvature",
                                geometry->vertexGaussianCurvatures);
```

# Report the field and proposed cones

The first report exposes the raw Geometry Central indices and their locations.
The second applies the index-to-curvature conversion documented in
[cone-targets.o.md](cone-targets.o.md), showing the exact prescription that
would be handed to Ricci flow before manual editing.

```cpp {name=report-frame-field}
print_frame_field_diagnostics(*mesh, *geometry, frameIndex, fieldSymmetry);
print_frame_field_cone_targets(
    derive_frame_field_cone_targets(*mesh, frameIndex, fieldSymmetry),
    std::cout);
```

# Frame-field diagnostics

For each nonzero index, the diagnostic prints both the integer index unit and
the ordinary fractional index obtained by dividing by the field symmetry. The
sum is compared with $n\chi$, the Poincare--Hopf expectation for a closed
$n$-symmetric field. On a surface with boundary, boundary alignment contributes
additional behavior, so this line is a diagnostic rather than a validity test.

```cpp {name=ff-defs}
void print_frame_field_diagnostics(gcs::SurfaceMesh& mesh,
                                   const gcs::VertexPositionGeometry& geometry,
                                   const gcs::VertexData<int>& frameIndex,
                                   int fieldSymmetry) {
  int frameIndexSum = 0;
  int singularityCount = 0;

  std::cout << "Frame-field singularity vertices:\n";
  for (gcs::Vertex vertex : mesh.vertices()) {
    const int index = frameIndex[vertex];
    frameIndexSum += index;
    if (index == 0) {
      continue;
    }

    const gc::Vector3& position = geometry.vertexPositions[vertex];
    ++singularityCount;
    std::cout << "  vertex " << vertex.getIndex() << ": index " << index
              << " (fractional " << static_cast<double>(index) / fieldSymmetry
              << "), position (" << position[0] << ", " << position[1] << ", "
              << position[2] << ")\n";
  }

  const int eulerCharacteristic = static_cast<int>(mesh.nVertices()) -
                                  static_cast<int>(mesh.nEdges()) +
                                  static_cast<int>(mesh.nFaces());
  std::cout << "Found " << singularityCount
            << " nonzero frame-field singularity vertices\n"
            << "Frame index sum: " << frameIndexSum << " (expected "
            << fieldSymmetry * eulerCharacteristic << " for a closed "
            << fieldSymmetry << "-field, chi = " << eulerCharacteristic
            << ")\n";
}
```
