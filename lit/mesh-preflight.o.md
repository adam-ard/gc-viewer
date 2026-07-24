# Ricci-Flow Topology Preflight

The paper's first step asks for a *feature-aware, computation-safe
triangulation*.  "Computation-safe" is broader than "manifold": it includes
geometric quality and faithful representation of the source CAD model.  We
therefore build the preflight in layers.  This first layer answers only the
topological question:

> Does the mesh belong to the class of surfaces that the current boundary
> Ricci-flow and cut-graph implementation is designed to process?

Keeping that question separate from geometric quality is useful pedagogically
and architecturally.  A topologically valid mesh can still contain a nearly
zero-area triangle, while a geometrically beautiful mesh can still contain a
nonmanifold edge.  The two failures need different explanations and different
repairs.

## What the manifold loader establishes

Geometry Central's `ManifoldSurfaceMesh` constructor establishes local
combinatorial facts: consistently oriented face adjacency, at most two faces
per edge, and one connected fan of faces around each vertex.  These are *local*
conditions.  They do not imply that the whole mesh is connected, that it has a
boundary, or that it has genus zero.

The current Ricci cut code accepts genus-zero surfaces with one or more boundary
components.  A disk has one boundary loop; an annulus has two.  Both have genus
zero, so both are valid inputs even though only the disk is already simply
connected.

## Euler characteristic as a topology checksum

For a finite surface mesh,

$$
    \chi = V-E+F.
$$

For an orientable surface with $c$ connected components, $b$ boundary
loops, and total genus $g$,

$$
    \chi = 2c-2g-b.
$$

Consequently,

$$
    2g = 2c-b-\chi.
$$

This relation does two jobs for us.  It identifies positive-genus inputs that
need handle and tunnel cuts not yet implemented, and it acts as a consistency
check: the right-hand side must be a nonnegative even integer for an orientable
manifold.

### Why not call `ManifoldSurfaceMesh::genus()`?

We should use Geometry Central as the authority whenever its semantics match
the algorithm.  Accordingly, the implementation below delegates manifoldness,
orientation, triangularity, connected-component traversal, boundary-loop
enumeration, and all element counts to Geometry Central.

The pinned Geometry Central revision uses a different Euler-characteristic
convention in `ManifoldSurfaceMesh::eulerCharacteristic()`: it evaluates
$V-E+F+b$, as though every boundary loop had been capped by a face.  The
Ricci-flow Gauss--Bonnet condition and the classification formula above require
the uncapped surface value $V-E+F$.  Moreover, the corresponding library
`genus()` helper is not suitable for an open, multiply connected surface; for
example, it reports a negative value for an annulus.

We therefore use Geometry Central to obtain $V$, $E$, $F$, $b$, and
$c$, but perform the final two lines of integer arithmetic ourselves.  This
is a semantic adapter, not a second mesh-topology implementation.  If a future
Geometry Central revision adds an explicitly boundary-aware genus routine, this
is the one calculation that should be replaced.

## Public report

A preflight returns data rather than merely throwing an exception.  The viewer
can teach the user *why* a mesh was rejected, and future UI work can display
every failed condition at once.

```cpp {name=mesh-preflight-interface-includes}
"mesh_preflight.h"
```

```cpp {name=mesh-preflight-header tangle=src/mesh_preflight.h}
#pragma once

#include <cstddef>
#include <cstdint>
#include <iosfwd>
#include <optional>
#include <string>
#include <vector>

namespace geometrycentral::surface {
class SurfaceMesh;
class VertexPositionGeometry;
}

struct RicciTopologyPreflight
{
    std::size_t connected_components = 0;
    std::size_t boundary_loops = 0;
    std::int64_t euler_characteristic = 0;
    std::optional<std::int64_t> total_genus;

    bool manifold = false;
    bool oriented = false;
    bool triangular = false;
    std::vector<std::string> failures;

    bool ready() const;
};

RicciTopologyPreflight analyze_ricci_topology(
    geometrycentral::surface::SurfaceMesh& mesh);

void print_ricci_topology_preflight(
    const RicciTopologyPreflight& report,
    std::ostream& output);

void require_ricci_topology(const RicciTopologyPreflight& report);

struct RicciGeometryThresholds
{
    double minimum_usable_quality = 1e-12;
    double warning_quality = 0.2;
};

struct RicciGeometryPreflight
{
    std::size_t nonfinite_vertices = 0;
    std::size_t invalid_edges = 0;
    std::size_t invalid_faces = 0;
    std::size_t invalid_corners = 0;
    std::size_t poor_quality_faces = 0;

    double minimum_edge_length = 0.0;
    double maximum_edge_length = 0.0;
    double minimum_face_area = 0.0;
    double minimum_triangle_quality = 0.0;
    double minimum_corner_angle_degrees = 0.0;

    std::vector<std::string> failures;
    std::vector<std::string> warnings;

    bool ready() const;
};

RicciGeometryPreflight analyze_ricci_geometry(
    geometrycentral::surface::SurfaceMesh& mesh,
    geometrycentral::surface::VertexPositionGeometry& geometry,
    const RicciGeometryThresholds& thresholds = {});

void print_ricci_geometry_preflight(
    const RicciGeometryPreflight& report,
    std::ostream& output);

void require_ricci_geometry(const RicciGeometryPreflight& report);
```

The implementation is split into topology and geometry fragments below. A
single tangle block assembles them into the translation unit.

```cpp {name=mesh-preflight-source tangle=src/mesh_preflight.cpp}
@<mesh-preflight-topology-source@>
@<mesh-preflight-geometry-source@>
```

## Computing the invariants

Notice that multiple boundary loops are not an error.  The cut graph is
responsible for joining them later.  Connectedness and genus are stricter:
the present implementation starts its planar face traversal from one seed face
and has no handle/tunnel cuts, so it requires exactly one genus-zero component.

```cpp {name=mesh-preflight-topology-source}
#include "mesh_preflight.h"

#include "geometrycentral/surface/surface_mesh.h"

#include <ostream>
#include <sstream>
#include <stdexcept>

namespace gcs = geometrycentral::surface;

bool RicciTopologyPreflight::ready() const
{
    return failures.empty();
}

RicciTopologyPreflight analyze_ricci_topology(gcs::SurfaceMesh& mesh)
{
    RicciTopologyPreflight report;
    report.connected_components = mesh.nConnectedComponents();
    report.boundary_loops = mesh.nBoundaryLoops();
    // Geometry Central's manifold helper adds one face per boundary loop.
    // Ricci Gauss--Bonnet instead needs the uncapped surface value V-E+F.
    report.euler_characteristic =
        static_cast<std::int64_t>(mesh.nVertices()) -
        static_cast<std::int64_t>(mesh.nEdges()) +
        static_cast<std::int64_t>(mesh.nFaces());
    report.manifold = mesh.isManifold();
    report.oriented = mesh.isOriented();
    report.triangular = mesh.isTriangular();

    if (!report.manifold)
    {
        report.failures.emplace_back(
            "the mesh is not a combinatorial 2-manifold");
    }
    if (!report.oriented)
    {
        report.failures.emplace_back(
            "the mesh does not have consistent face orientation");
    }
    if (!report.triangular)
    {
        report.failures.emplace_back(
            "Ricci flow currently requires every face to be a triangle");
    }
    if (report.connected_components != 1)
    {
        report.failures.emplace_back(
            "Ricci flattening requires exactly one connected component");
    }
    if (report.boundary_loops == 0)
    {
        report.failures.emplace_back(
            "boundary flattening requires at least one boundary loop");
    }

    if (report.manifold && report.oriented)
    {
        const std::int64_t twice_genus =
            2 * static_cast<std::int64_t>(report.connected_components) -
            static_cast<std::int64_t>(report.boundary_loops) -
            report.euler_characteristic;

        if (twice_genus < 0 || twice_genus % 2 != 0)
        {
            report.failures.emplace_back(
                "mesh counts are inconsistent with an orientable surface");
        }
        else
        {
            report.total_genus = twice_genus / 2;
            if (*report.total_genus != 0)
            {
                report.failures.emplace_back(
                    "the current cut graph supports genus-zero surfaces only");
            }
        }
    }

    return report;
}

void print_ricci_topology_preflight(
    const RicciTopologyPreflight& report,
    std::ostream& output)
{
    output << "Ricci topology preflight: "
           << (report.ready() ? "ready" : "not ready") << '\n'
           << "  connected components: " << report.connected_components << '\n'
           << "  boundary loops: " << report.boundary_loops << '\n'
           << "  Euler characteristic: " << report.euler_characteristic << '\n'
           << "  total genus: ";
    if (report.total_genus)
    {
        output << *report.total_genus;
    }
    else
    {
        output << "undetermined";
    }
    output << '\n';

    for (const std::string& failure : report.failures)
    {
        output << "  failed: " << failure << '\n';
    }
}

void require_ricci_topology(const RicciTopologyPreflight& report)
{
    if (report.ready())
    {
        return;
    }

    std::ostringstream message;
    message << "Ricci topology preflight failed";
    for (const std::string& failure : report.failures)
    {
        message << "\n- " << failure;
    }
    throw std::runtime_error(message.str());
}
```

This report deliberately makes no claims about triangle area, aspect ratio,
self-intersection, or feature preservation.  Those belong to the geometric
layer of the Step 1 preflight.

# Geometric Preflight

Topology describes which vertices and faces are connected. Geometry assigns a
point in $\mathbb{R}^3$ to each vertex and therefore gives every edge a
length, every triangle an area, and every corner an angle. The Ricci solver
needs these quantities to define its initial metric.

## A scale-independent quality measure

Absolute area alone is not a useful quality criterion: an otherwise identical
mesh expressed in millimeters has areas one million times larger than when it
is expressed in meters. For a triangle with area $A$ and edge lengths
$a,b,c$, we use

$$
    q = \frac{4\sqrt{3}A}{a^2+b^2+c^2}.
$$

This value is 1 for an equilateral triangle, approaches 0 as a triangle
collapses, and does not change under uniform scaling. A value below
`minimum_usable_quality` is an error because the triangle is numerically
degenerate. A value below `warning_quality` is a warning: Ricci flow may still
work, but the mesh does not satisfy the paper's recommendation for good aspect
ratios.

Geometry Central computes the metric quantities. The application only applies
the policy thresholds.

```cpp {name=mesh-preflight-geometry-source}
#include "geometrycentral/surface/vertex_position_geometry.h"

#include <algorithm>
#include <cmath>
#include <limits>

namespace {
constexpr double pi = 3.141592653589793238462643383279502884;
constexpr double radians_to_degrees = 180.0 / pi;

std::string count_message(std::size_t count, const std::string& description)
{
    return std::to_string(count) + " " + description;
}
}

bool RicciGeometryPreflight::ready() const
{
    return failures.empty();
}

RicciGeometryPreflight analyze_ricci_geometry(
    gcs::SurfaceMesh& mesh,
    gcs::VertexPositionGeometry& geometry,
    const RicciGeometryThresholds& thresholds)
{
    if (!std::isfinite(thresholds.minimum_usable_quality) ||
        !std::isfinite(thresholds.warning_quality) ||
        thresholds.minimum_usable_quality <= 0.0 ||
        thresholds.warning_quality < thresholds.minimum_usable_quality ||
        thresholds.warning_quality > 1.0)
    {
        throw std::invalid_argument(
            "triangle-quality thresholds must satisfy "
            "0 < minimum usable <= warning <= 1");
    }

    RicciGeometryPreflight report;
    report.minimum_edge_length = std::numeric_limits<double>::infinity();
    report.minimum_face_area = std::numeric_limits<double>::infinity();
    report.minimum_triangle_quality =
        std::numeric_limits<double>::infinity();
    report.minimum_corner_angle_degrees =
        std::numeric_limits<double>::infinity();

    for (gcs::Vertex vertex : mesh.vertices())
    {
        if (!geometry.vertexPositions[vertex].isFinite())
        {
            ++report.nonfinite_vertices;
        }
    }

    if (report.nonfinite_vertices != 0)
    {
        report.failures.push_back(count_message(
            report.nonfinite_vertices,
            "vertices have non-finite coordinates"));
        return report;
    }

    if (!mesh.isTriangular())
    {
        report.failures.emplace_back(
            "geometric triangle checks require a triangular mesh");
        return report;
    }

    geometry.requireEdgeLengths();
    geometry.requireFaceAreas();
    geometry.requireCornerAngles();

    for (gcs::Edge edge : mesh.edges())
    {
        const double length = geometry.edgeLengths[edge];
        if (!std::isfinite(length) || length <= 0.0)
        {
            ++report.invalid_edges;
            continue;
        }
        report.minimum_edge_length =
            std::min(report.minimum_edge_length, length);
        report.maximum_edge_length =
            std::max(report.maximum_edge_length, length);
    }

    for (gcs::Face face : mesh.faces())
    {
        const double area = geometry.faceAreas[face];
        double squared_length_sum = 0.0;
        for (gcs::Edge edge : face.adjacentEdges())
        {
            const double length = geometry.edgeLengths[edge];
            squared_length_sum += length * length;
        }

        if (!std::isfinite(area) || area <= 0.0 ||
            !std::isfinite(squared_length_sum) ||
            squared_length_sum <= 0.0)
        {
            ++report.invalid_faces;
            continue;
        }

        report.minimum_face_area =
            std::min(report.minimum_face_area, area);
        const double quality =
            4.0 * std::sqrt(3.0) * area / squared_length_sum;
        report.minimum_triangle_quality =
            std::min(report.minimum_triangle_quality, quality);

        if (!std::isfinite(quality) ||
            quality <= thresholds.minimum_usable_quality)
        {
            ++report.invalid_faces;
        }
        else if (quality < thresholds.warning_quality)
        {
            ++report.poor_quality_faces;
        }

        for (gcs::Corner corner : face.adjacentCorners())
        {
            const double angle = geometry.cornerAngles[corner];
            if (!std::isfinite(angle) || angle <= 0.0 || angle >= pi)
            {
                ++report.invalid_corners;
                continue;
            }
            report.minimum_corner_angle_degrees = std::min(
                report.minimum_corner_angle_degrees,
                angle * radians_to_degrees);
        }
    }

    geometry.unrequireCornerAngles();
    geometry.unrequireFaceAreas();
    geometry.unrequireEdgeLengths();

    if (report.invalid_edges != 0)
    {
        report.failures.push_back(count_message(
            report.invalid_edges,
            "edges have zero or non-finite length"));
    }
    if (report.invalid_faces != 0)
    {
        report.failures.push_back(count_message(
            report.invalid_faces,
            "triangles are geometrically degenerate"));
    }
    if (report.invalid_corners != 0)
    {
        report.failures.push_back(count_message(
            report.invalid_corners,
            "triangle corners have invalid angles"));
    }
    if (report.poor_quality_faces != 0)
    {
        report.warnings.push_back(count_message(
            report.poor_quality_faces,
            "triangles have quality below the recommended threshold"));
    }

    return report;
}

void print_ricci_geometry_preflight(
    const RicciGeometryPreflight& report,
    std::ostream& output)
{
    output << "Ricci geometry preflight: "
           << (report.ready() ? "ready" : "not ready") << '\n';

    if (std::isfinite(report.minimum_edge_length))
    {
        output << "  edge length range: [" << report.minimum_edge_length
               << ", " << report.maximum_edge_length << "]\n";
    }
    if (std::isfinite(report.minimum_face_area))
    {
        output << "  minimum triangle area: "
               << report.minimum_face_area << '\n';
    }
    if (std::isfinite(report.minimum_triangle_quality))
    {
        output << "  minimum triangle quality: "
               << report.minimum_triangle_quality
               << " (1 is equilateral)\n";
    }
    if (std::isfinite(report.minimum_corner_angle_degrees))
    {
        output << "  minimum corner angle: "
               << report.minimum_corner_angle_degrees << " degrees\n";
    }

    for (const std::string& failure : report.failures)
    {
        output << "  failed: " << failure << '\n';
    }
    for (const std::string& warning : report.warnings)
    {
        output << "  warning: " << warning << '\n';
    }
}

void require_ricci_geometry(const RicciGeometryPreflight& report)
{
    if (report.ready())
    {
        return;
    }

    std::ostringstream message;
    message << "Ricci geometry preflight failed";
    for (const std::string& failure : report.failures)
    {
        message << "\n- " << failure;
    }
    throw std::runtime_error(message.str());
}
```

## Executable examples

Small synthetic meshes make the topology equations concrete and protect the
preflight from regression.  The examples include a disk and an annulus to
emphasize that "more than one boundary" does not mean "positive genus."

```cpp {name=mesh-preflight-test tangle=tests/mesh_preflight_test.cpp}
#include "mesh_preflight.h"

#include "geometrycentral/surface/manifold_surface_mesh.h"
#include "geometrycentral/surface/vertex_position_geometry.h"

#include <cstdlib>
#include <cmath>
#include <iostream>
#include <limits>
#include <string>
#include <vector>

#include <Eigen/Core>

namespace gcs = geometrycentral::surface;

namespace {
void expect(bool condition, const std::string& explanation)
{
    if (!condition)
    {
        std::cerr << "Preflight example failed: " << explanation << '\n';
        std::exit(EXIT_FAILURE);
    }
}

gcs::ManifoldSurfaceMesh make_mesh(
    const std::vector<std::vector<std::size_t>>& faces)
{
    return gcs::ManifoldSurfaceMesh(faces);
}
}

int main()
{
    auto disk = make_mesh({{0, 1, 2}});
    const RicciTopologyPreflight disk_report =
        analyze_ricci_topology(disk);
    expect(disk_report.ready(), "a triangular disk should be accepted");
    expect(disk_report.euler_characteristic == 1, "a disk has chi = 1");
    expect(disk_report.boundary_loops == 1, "a disk has one boundary");
    expect(disk_report.total_genus == 0, "a disk has genus zero");

    auto annulus = make_mesh({
        {0, 1, 5}, {0, 5, 4},
        {1, 2, 6}, {1, 6, 5},
        {2, 3, 7}, {2, 7, 6},
        {3, 0, 4}, {3, 4, 7},
    });
    const RicciTopologyPreflight annulus_report =
        analyze_ricci_topology(annulus);
    expect(annulus_report.ready(), "a triangular annulus should be accepted");
    expect(annulus_report.euler_characteristic == 0, "an annulus has chi = 0");
    expect(annulus_report.boundary_loops == 2,
           "an annulus has two boundary loops");
    expect(annulus_report.total_genus == 0, "an annulus has genus zero");

    auto disconnected = make_mesh({{0, 1, 2}, {3, 4, 5}});
    expect(!analyze_ricci_topology(disconnected).ready(),
           "two disconnected disks should be rejected");

    auto closed_tetrahedron = make_mesh({
        {0, 2, 1},
        {0, 1, 3},
        {1, 2, 3},
        {2, 0, 3},
    });
    expect(!analyze_ricci_topology(closed_tetrahedron).ready(),
           "the boundary flattener should reject a closed sphere");

    auto quadrilateral = make_mesh({{0, 1, 2, 3}});
    expect(!analyze_ricci_topology(quadrilateral).ready(),
           "the current Ricci solver should reject polygonal faces");

    Eigen::Matrix<double, 3, 3> equilateral_positions;
    equilateral_positions <<
        0.0, 0.0, 0.0,
        1.0, 0.0, 0.0,
        0.5, std::sqrt(3.0) / 2.0, 0.0;
    gcs::VertexPositionGeometry equilateral_geometry(
        disk, equilateral_positions);
    const RicciGeometryPreflight equilateral_report =
        analyze_ricci_geometry(disk, equilateral_geometry);
    expect(equilateral_report.ready(),
           "an equilateral triangle should be geometrically valid");
    expect(std::abs(equilateral_report.minimum_triangle_quality - 1.0) <
               1e-12,
           "an equilateral triangle should have quality one");
    expect(equilateral_report.warnings.empty(),
           "an equilateral triangle should have no quality warning");

    Eigen::Matrix<double, 3, 3> scaled_positions =
        1000.0 * equilateral_positions;
    gcs::VertexPositionGeometry scaled_geometry(disk, scaled_positions);
    const RicciGeometryPreflight scaled_report =
        analyze_ricci_geometry(disk, scaled_geometry);
    expect(std::abs(scaled_report.minimum_triangle_quality -
                    equilateral_report.minimum_triangle_quality) < 1e-12,
           "uniform scaling should not change triangle quality");

    Eigen::Matrix<double, 3, 3> skinny_positions;
    skinny_positions <<
        0.0, 0.0, 0.0,
        1.0, 0.0, 0.0,
        1e-8, 1e-8, 0.0;
    gcs::VertexPositionGeometry skinny_geometry(disk, skinny_positions);
    const RicciGeometryPreflight skinny_report =
        analyze_ricci_geometry(disk, skinny_geometry);
    expect(skinny_report.ready(),
           "a skinny but nondegenerate triangle should remain usable");
    expect(!skinny_report.warnings.empty(),
           "a skinny triangle should produce a quality warning");

    Eigen::Matrix<double, 3, 3> collinear_positions;
    collinear_positions <<
        0.0, 0.0, 0.0,
        1.0, 0.0, 0.0,
        2.0, 0.0, 0.0;
    gcs::VertexPositionGeometry collinear_geometry(
        disk, collinear_positions);
    expect(!analyze_ricci_geometry(disk, collinear_geometry).ready(),
           "a zero-area triangle should be rejected");

    Eigen::Matrix<double, 3, 3> nonfinite_positions =
        equilateral_positions;
    nonfinite_positions(2, 1) =
        std::numeric_limits<double>::quiet_NaN();
    gcs::VertexPositionGeometry nonfinite_geometry(
        disk, nonfinite_positions);
    expect(!analyze_ricci_geometry(disk, nonfinite_geometry).ready(),
           "a non-finite vertex coordinate should be rejected");

    return EXIT_SUCCESS;
}
```
