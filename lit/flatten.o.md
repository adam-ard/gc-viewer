# Flatten UI

The flatten module owns the interactive Ricci-flow workflow. Its public header
exposes one callback factory; process management, validation, output placement,
and UI state remain private to the implementation.

# Public Interface

```cpp {name=flatten-header tangle=src/flatten.h}
#pragma once

#include <functional>
#include <string>

namespace geometrycentral::surface {
class SurfaceMesh;
class VertexPositionGeometry;
}

std::function<void()> make_flatten_callback(
    std::string input_path,
    geometrycentral::surface::SurfaceMesh& mesh,
    geometrycentral::surface::VertexPositionGeometry& geometry);
```

# Implementation Includes

```cpp {name=flatten-includes}
"flatten.h"
"mesh_preflight.h"

"geometrycentral/surface/meshio.h"
"geometrycentral/surface/surface_mesh.h"
"geometrycentral/surface/vertex_position_geometry.h"

"polyscope/polyscope.h"
"polyscope/surface_mesh.h"
"polyscope/view.h"

<algorithm>
<chrono>
<cstdlib>
<filesystem>
<future>
<limits>
<memory>
<stdexcept>
<utility>
```

# Ricci Command

```cpp {name=flatten-definitions}
std::string shell_quote(const std::string& value)
{
    std::string quoted = "'";
    for (char character : value)
    {
        if (character == '\'')
        {
            quoted += "'\\''";
        }
        else
        {
            quoted += character;
        }
    }
    quoted += '\'';
    return quoted;
}

struct RicciConfig
{
    std::filesystem::path input_path;
    std::filesystem::path target_path;
    std::filesystem::path output_path;

    std::string command() const
    {
        return shell_quote(RICCI_FLOW_CLI_PATH) + " " +
               shell_quote(input_path.string()) +
               " --flatten-boundary --cone-targets " +
               shell_quote(target_path.string()) +
               " --target-mode additive --target-continuation --output " +
               shell_quote(output_path.string());
    }
};

RicciConfig make_validation_ricci_config(
    const std::filesystem::path& input_path)
{
    if (input_path.filename() != "stanford_bunny_with_hole.obj")
    {
        throw std::runtime_error(
            "The validation cone targets require "
            "stanford_bunny_with_hole.obj");
    }

    const std::filesystem::path output_directory =
        std::filesystem::temp_directory_path() / "geometry-central-viewer";
    std::filesystem::create_directories(output_directory);

    RicciConfig config{
        input_path,
        "assets/targets_bunny_pi_over_4.txt",
        output_directory /
            (input_path.stem().string() + ".ricci-flat.obj")};

    if (!std::filesystem::is_regular_file(config.target_path))
    {
        throw std::runtime_error(
            "Could not find validation targets: " +
            config.target_path.string());
    }

    return config;
}

struct RicciJob
{
    RicciConfig config;
    std::future<int> result;

    void start(RicciConfig new_config)
    {
        config = std::move(new_config);
        const std::string command = config.command();
        result = std::async(std::launch::async, [command]() {
            return std::system(command.c_str());
        });
    }

    bool running() const
    {
        return result.valid();
    }

    bool ready() const
    {
        return running() &&
               result.wait_for(std::chrono::seconds(0)) ==
                   std::future_status::ready;
    }

    int finish()
    {
        return result.get();
    }
};
```

# Mesh Placement

```cpp {name=flatten-definitions}
struct MeshBounds
{
    gc::Vector3 min = gc::Vector3::infinity();
    gc::Vector3 max = -gc::Vector3::infinity();

    gc::Vector3 center() const
    {
        return 0.5 * (min + max);
    }

    double extent() const
    {
        const gc::Vector3 size = max - min;
        return std::max({size[0], size[1], size[2], 1e-6});
    }
};

MeshBounds mesh_bounds(gcs::SurfaceMesh& mesh,
                       const gcs::VertexPositionGeometry& geometry)
{
    MeshBounds bounds;
    for (gcs::Vertex vertex : mesh.vertices())
    {
        bounds.min = gc::componentwiseMin(
            bounds.min, geometry.vertexPositions[vertex]);
        bounds.max = gc::componentwiseMax(
            bounds.max, geometry.vertexPositions[vertex]);
    }
    return bounds;
}

void place_flattened_mesh_beside_original(
    gcs::SurfaceMesh& original_mesh,
    const gcs::VertexPositionGeometry& original_geometry,
    gcs::SurfaceMesh& flattened_mesh,
    gcs::VertexPositionGeometry& flattened_geometry)
{
    const MeshBounds original_bounds =
        mesh_bounds(original_mesh, original_geometry);
    const MeshBounds flat_bounds =
        mesh_bounds(flattened_mesh, flattened_geometry);
    const double original_extent = original_bounds.extent();
    const double scale = 0.5 * original_extent / flat_bounds.extent();

    const gc::Vector3 original_center = original_bounds.center();
    const gc::Vector3 flat_center = flat_bounds.center();
    const gc::Vector3 target_center{
        original_bounds.max[0] + 0.4 * original_extent,
        original_center[1],
        original_center[2]};

    for (gcs::Vertex vertex : flattened_mesh.vertices())
    {
        flattened_geometry.vertexPositions[vertex] =
            scale * (flattened_geometry.vertexPositions[vertex] -
                     flat_center) +
            target_center;
    }
}
```

# Input Validation

The topology preflight is shared with the viewer's load-time report. Repeating
the inexpensive analysis when the button is pressed keeps the flattening
boundary honest if later viewer tools are allowed to modify mesh connectivity.

```cpp {name=flatten-definitions}
void validate_ricci_input(
    gcs::SurfaceMesh& mesh,
    gcs::VertexPositionGeometry& geometry)
{
    require_ricci_topology(analyze_ricci_topology(mesh));
    require_ricci_geometry(analyze_ricci_geometry(mesh, geometry));
}

struct FlattenState
{
    std::string status;
    RicciJob job;
};
```

# Callback Body

```cpp {name=flatten-callback-body}
ImGui::BeginDisabled(state->job.running());
const bool flatten_requested = ImGui::Button("Flatten");
ImGui::EndDisabled();

if (flatten_requested)
{
    try
    {
        validate_ricci_input(mesh, geometry);

        state->status = "Running Ricci flow...";
        state->job.start(make_validation_ricci_config(input_path));
    }
    catch (const std::exception& error)
    {
        state->status = std::string("Flatten failed: ") + error.what();
    }
}

if (state->job.ready())
{
    try
    {
        const int exit_status = state->job.finish();
        if (exit_status != 0)
        {
            throw std::runtime_error(
                "Ricci flow failed with status " +
                std::to_string(exit_status) +
                "; targets: " + state->job.config.target_path.string());
        }

        auto [flattened_mesh, flattened_geometry] =
            gcs::readSurfaceMesh(state->job.config.output_path.string());
        place_flattened_mesh_beside_original(
            mesh, geometry, *flattened_mesh, *flattened_geometry);

        polyscope::registerSurfaceMesh(
            "flattened mesh", flattened_geometry->vertexPositions,
            flattened_mesh->getFaceVertexList());
        polyscope::view::resetCameraToHomeView();

        state->status =
            "Loaded Ricci mesh using four pi/4 cone targets\nTargets: " +
            state->job.config.target_path.string() +
            "\nOutput: " + state->job.config.output_path.string();
    }
    catch (const std::exception& error)
    {
        state->status = std::string("Flatten failed: ") + error.what();
    }
}

if (!state->status.empty())
{
    ImGui::TextWrapped("%s", state->status.c_str());
}
```

# Implementation File

```cpp {name=flatten-source tangle=src/flatten.cpp}
#include @<flatten-includes@>

namespace gc = geometrycentral;
namespace gcs = geometrycentral::surface;

namespace {
    @<flatten-definitions@>
}

std::function<void()> make_flatten_callback(
    std::string input_path,
    gcs::SurfaceMesh& mesh,
    gcs::VertexPositionGeometry& geometry)
{
    auto state = std::make_shared<FlattenState>();
    return [state, input_path = std::move(input_path), &mesh, &geometry]() {
        @<flatten-callback-body@>
    };
}
```
