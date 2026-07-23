# From Frame-Field Indices to Ricci Cone Targets

The fourfold direction field gives more than a visual suggestion of where a
quadrilateral layout might have extraordinary vertices. Its integer index is
also the natural unit for prescribing curvature to Ricci flow.

## Index and curvature

An \(n\)-symmetric field stores directions modulo a rotation of \(2\pi/n\).
Geometry Central reports an integer field index \(I(p)\), while the ordinary
topological index is \(I(p)/n\). A cone representing that singularity therefore
has target curvature

\[
    \bar K(p)=\frac{2\pi}{n}I(p).
\]

For the fourfold field used by a quadrilateral layout this becomes

\[
    \bar K(p)=\frac{\pi}{2}I(p).
\]

Thus an index \(+1\) suggests a \(+\pi/2\) cone and an index \(-1\) suggests a
\(-\pi/2\) cone. Zero-index vertices remain regular and need not appear in the
target file.

## Gauss--Bonnet residual

The target curvatures of a surface must sum to \(2\pi\chi\). In index units,

\[
    \sum_p I(p)=n\chi.
\]

For closed fields this is the familiar Poincare--Hopf relation. On an open mesh,
the boundary-aligned field and Geometry Central's per-vertex index rounding do
not necessarily make the selected nonzero indices sum to \(n\chi\). We retain
the solver's *additive* target policy during this intermediate stage: exact
quarter-turn targets are assigned at frame singularities, and the remaining
curvature is distributed over unselected boundary vertices. The report exposes
that residual explicitly. A later manual editor should aim to make it zero.

## Public representation

The dense vertex index is important. Geometry Central may remove unused input
vertices while loading, and mesh mutation can make raw handle indices sparse.
`getVertexIndices()` supplies the canonical dense enumeration used when the
normalized solver input is written.

```cpp {name=cone-targets-header tangle=src/cone_targets.h}
#pragma once

#include "geometrycentral/surface/surface_mesh.h"

#include <cstddef>
#include <cstdint>
#include <iosfwd>
#include <vector>

struct FrameFieldConeTarget
{
    std::size_t vertex_index = 0;
    int frame_index = 0;
    double curvature_radians = 0.0;
};

struct FrameFieldConeTargets
{
    int field_symmetry = 0;
    std::int64_t euler_characteristic = 0;
    std::int64_t selected_index_sum = 0;
    std::int64_t required_index_sum = 0;
    double selected_curvature_sum = 0.0;
    double required_curvature_sum = 0.0;
    double curvature_residual = 0.0;
    std::vector<FrameFieldConeTarget> targets;

    bool satisfies_gauss_bonnet() const;
};

FrameFieldConeTargets derive_frame_field_cone_targets(
    geometrycentral::surface::SurfaceMesh& mesh,
    const geometrycentral::surface::VertexData<int>& frame_indices,
    int field_symmetry);

void write_obj_cone_targets(
    const FrameFieldConeTargets& targets,
    std::ostream& output);

void print_frame_field_cone_targets(
    const FrameFieldConeTargets& targets,
    std::ostream& output);
```

## Derivation and serialization

The target file uses OBJ's one-based vertex convention because the temporary
solver input is always normalized to OBJ. Comments preserve the originating
frame index for inspection without changing the Ricci parser's two-column
format.

```cpp {name=cone-targets-source tangle=src/cone_targets.cpp}
#include "cone_targets.h"

#include <cmath>
#include <iomanip>
#include <ostream>
#include <stdexcept>

namespace gcs = geometrycentral::surface;

namespace {
constexpr double pi = 3.141592653589793238462643383279502884;
}

bool FrameFieldConeTargets::satisfies_gauss_bonnet() const
{
    return selected_index_sum == required_index_sum;
}

FrameFieldConeTargets derive_frame_field_cone_targets(
    gcs::SurfaceMesh& mesh,
    const gcs::VertexData<int>& frame_indices,
    int field_symmetry)
{
    if (frame_indices.getMesh() != &mesh)
    {
        throw std::invalid_argument(
            "frame indices and cone targets must use the same mesh");
    }
    if (field_symmetry <= 0)
    {
        throw std::invalid_argument(
            "field symmetry must be a positive integer");
    }

    FrameFieldConeTargets result;
    result.field_symmetry = field_symmetry;
    result.euler_characteristic =
        static_cast<std::int64_t>(mesh.nVertices()) -
        static_cast<std::int64_t>(mesh.nEdges()) +
        static_cast<std::int64_t>(mesh.nFaces());
    result.required_index_sum =
        static_cast<std::int64_t>(field_symmetry) *
        result.euler_characteristic;

    const double curvature_per_index =
        2.0 * pi / static_cast<double>(field_symmetry);
    const gcs::VertexData<std::size_t> dense_indices =
        mesh.getVertexIndices();

    for (gcs::Vertex vertex : mesh.vertices())
    {
        const int index = frame_indices[vertex];
        result.selected_index_sum += index;
        if (index == 0)
        {
            continue;
        }

        const double curvature =
            curvature_per_index * static_cast<double>(index);
        result.targets.push_back(
            {dense_indices[vertex], index, curvature});
        result.selected_curvature_sum += curvature;
    }

    result.required_curvature_sum =
        2.0 * pi * static_cast<double>(result.euler_characteristic);
    result.curvature_residual =
        result.required_curvature_sum -
        result.selected_curvature_sum;
    return result;
}

void write_obj_cone_targets(
    const FrameFieldConeTargets& targets,
    std::ostream& output)
{
    output << "# Generated from Geometry Central frame-field singularities\n"
           << "# Format: OBJ vertex_id (1-based) curvature_radians\n"
           << "# field symmetry: " << targets.field_symmetry << '\n'
           << "# selected index sum: " << targets.selected_index_sum << '\n'
           << "# required index sum: " << targets.required_index_sum << '\n'
           << std::setprecision(17);

    for (const FrameFieldConeTarget& target : targets.targets)
    {
        output << target.vertex_index + 1 << ' '
               << target.curvature_radians
               << " # frame index " << target.frame_index << '\n';
    }
}

void print_frame_field_cone_targets(
    const FrameFieldConeTargets& targets,
    std::ostream& output)
{
    output << "Frame-field Ricci targets:\n"
           << "  selected cones: " << targets.targets.size() << '\n'
           << "  selected index sum: "
           << targets.selected_index_sum << '\n'
           << "  required index sum: "
           << targets.required_index_sum << '\n'
           << "  Gauss-Bonnet curvature residual: "
           << targets.curvature_residual << " radians\n";

    if (!targets.satisfies_gauss_bonnet())
    {
        output << "  note: additive mode will distribute this residual "
                  "over unselected boundary vertices\n";
    }
}
```

## Executable examples

The first example intentionally leaves one index unit of residual. This models
the transition state before manual editing: the selected cones remain exact
quarter turns, while the report truthfully records what the boundary must
absorb.

```cpp {name=cone-targets-test tangle=tests/cone_targets_test.cpp}
#include "cone_targets.h"

#include "geometrycentral/surface/manifold_surface_mesh.h"

#include <cmath>
#include <cstdlib>
#include <iostream>
#include <sstream>
#include <string>

namespace gcs = geometrycentral::surface;

namespace {
constexpr double pi = 3.141592653589793238462643383279502884;

void expect(bool condition, const std::string& explanation)
{
    if (!condition)
    {
        std::cerr << "Cone-target example failed: "
                  << explanation << '\n';
        std::exit(EXIT_FAILURE);
    }
}
}

int main()
{
    gcs::ManifoldSurfaceMesh disk({{0, 1, 2}});
    gcs::VertexData<int> indices(disk, 1);

    const FrameFieldConeTargets residual_targets =
        derive_frame_field_cone_targets(disk, indices, 4);
    expect(residual_targets.targets.size() == 3,
           "all three nonzero frame indices should become cones");
    expect(residual_targets.selected_index_sum == 3,
           "the selected index sum should be three");
    expect(residual_targets.required_index_sum == 4,
           "a disk requires four index units for a 4-field");
    expect(std::abs(residual_targets.curvature_residual - pi / 2.0) <
               1e-12,
           "one missing index unit should leave pi/2 residual");

    std::ostringstream serialized;
    write_obj_cone_targets(residual_targets, serialized);
    expect(serialized.str().find("1 1.570796") != std::string::npos,
           "OBJ targets should be one-based and use pi/2 per index");
    expect(serialized.str().find("# frame index 1") != std::string::npos,
           "the serialized comment should retain the frame index");

    indices[disk.vertex(0)] = 4;
    indices[disk.vertex(1)] = 0;
    indices[disk.vertex(2)] = 0;
    const FrameFieldConeTargets complete_targets =
        derive_frame_field_cone_targets(disk, indices, 4);
    expect(complete_targets.satisfies_gauss_bonnet(),
           "an index sum of four should satisfy disk Gauss-Bonnet");
    expect(complete_targets.targets.size() == 1,
           "zero-index vertices should not be serialized");
    expect(std::abs(complete_targets.targets[0].curvature_radians -
                    2.0 * pi) < 1e-12,
           "index four should map to curvature 2pi");

    return EXIT_SUCCESS;
}
```
