# Surface Analysis and Viewer Quantities

This section derives the geometric information that connects the input mesh to
the later Ricci-flow workflow. A fourfold direction field proposes cone
singularities, while Polyscope quantities expose the field, its indices, and
the mesh's Gaussian curvature for inspection.

# Dependencies :CARD:

Geometry Central computes the direction field and singularity indices. The cone
target interface interprets those indices for Ricci flow.

```cpp {name=surface-analysis-includes}
<cstdint>
@<cone-targets-interface-includes@>
"geometrycentral/surface/direction_fields.h"
```

# File-local definitions :CARD:

The only file-local helper expands the raw frame-field indices into a detailed
diagnostic report. Keeping it behind this aggregate lets `main.o.md` assemble
private definitions without knowing their implementation categories.

```cpp {name=surface-analysis-defs}
@<frame-field-diagnostic-defs@>
```

# Compute a fourfold direction field :CARD:

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

# Attach analysis quantities to Polyscope  :CARD:

Geometry Central expresses the cross field in each face's two-dimensional
tangent coordinates. Polyscope needs the corresponding three-dimensional basis
vectors to draw those directions on the embedded surface, so we request the
face tangent bases and provide both basis vectors with the field.

The vertex index and Gaussian curvature are scalar quantities. Marking the
index as symmetric gives positive and negative singularities a diverging color
map centered at zero.

Polyscope copies these arrays into its own quantity objects. Once registration
is complete, the matching `unrequire...()` calls release Geometry Central's
cached tangent bases and Gaussian curvatures because later pipeline stages do
not read them.

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

geometry->unrequireFaceTangentBasis();
geometry->unrequireVertexGaussianCurvatures();
```

# Report the field and proposed cones :CARD:

The first report exposes the raw Geometry Central indices and their locations.
The second applies the index-to-curvature conversion documented in
[cone-targets.o.md](cone-targets.o.md), showing the exact prescription that
would be handed to Ricci flow before manual editing.

Geometry Central has already copied the topology report's Euler characteristic
using the boundary convention required by Ricci flow. Passing that value to the
diagnostic avoids duplicating topology arithmetic in this section.

```cpp {name=report-frame-field}
print_frame_field_diagnostics(*mesh,
                              *geometry,
                              frameIndex,
                              fieldSymmetry,
                              topologyPreflight.euler_characteristic);
print_frame_field_cone_targets(
    derive_frame_field_cone_targets(*mesh, frameIndex, fieldSymmetry),
    std::cout);
```

# Frame-field diagnostics :CARD:

For each nonzero index, the diagnostic prints both the integer index unit and
the ordinary fractional index obtained by dividing by the field symmetry. The
sum is compared with $n\chi$, the Poincare--Hopf expectation for a closed
$n$-symmetric field. On a surface with boundary, boundary alignment contributes
additional behavior, so this line is a diagnostic rather than a validity test.

```cpp {name=frame-field-diagnostic-defs}
void print_frame_field_diagnostics(gcs::SurfaceMesh& mesh,
                                   const gcs::VertexPositionGeometry& geometry,
                                   const gcs::VertexData<int>& frameIndex,
                                   int fieldSymmetry,
                                   std::int64_t eulerCharacteristic) {
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

  std::cout << "Found " << singularityCount
            << " nonzero frame-field singularity vertices\n"
            << "Frame index sum: " << frameIndexSum << " (expected "
            << fieldSymmetry * eulerCharacteristic << " for a closed "
            << fieldSymmetry << "-field, chi = " << eulerCharacteristic
            << ")\n";
}
```
