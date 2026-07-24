# Flatten UI

The flatten module owns the interactive Ricci-flow workflow. Its public header
exposes one callback factory; process management, validation, output placement,
and UI state remain private to the implementation.

# Public Interface

```cpp {name=flatten-interface-includes}
"flatten.h"
```

```cpp {name=flatten-header tangle=src/flatten.h}
#pragma once

#include "geometrycentral/surface/surface_mesh.h"

#include <functional>
#include <string>

namespace geometrycentral::surface {
class VertexPositionGeometry;
}

std::function<void()> make_flatten_callback(
    std::string input_path,
    geometrycentral::surface::SurfaceMesh& mesh,
    geometrycentral::surface::VertexPositionGeometry& geometry,
    geometrycentral::surface::VertexData<int>& frame_indices,
    int field_symmetry);
```

# Implementation Includes

```cpp {name=flatten-implementation-includes}
"cone_targets.h"
"flatten.h"
"mesh_preflight.h"

"geometrycentral/surface/meshio.h"
"geometrycentral/surface/surface_mesh.h"
"geometrycentral/surface/vertex_position_geometry.h"

"polyscope/polyscope.h"
"polyscope/pick.h"
"polyscope/point_cloud.h"
"polyscope/point_cloud_color_quantity.h"
"polyscope/surface_mesh.h"
"polyscope/view.h"

<algorithm>
<chrono>
<cstdlib>
<filesystem>
<fstream>
<future>
<limits>
<memory>
<sstream>
<stdexcept>
<utility>
<vector>
```

# Ricci Command

The command-line bridge uses an OBJ written from Geometry Central's in-memory
mesh rather than reopening the user's original file. This normalization makes
the target IDs reliable: Geometry Central may strip unused input vertices, and
its dense enumeration is exactly the enumeration written to the temporary OBJ.
When the Ricci solver becomes a library this pair of temporary files can be
replaced by in-memory data without changing the cone-selection model.

```cpp {name=flatten-definitions}
std::string shell_quote(const std::string& value) {
  std::string quoted = "'";
  for (char character : value) {
    if (character == '\'') {
      quoted += "'\\''";
    } else {
      quoted += character;
    }
  }
  quoted += '\'';
  return quoted;
}

struct RicciConfig {
  std::filesystem::path input_path;
  std::filesystem::path target_path;
  std::filesystem::path output_path;
  FrameFieldConeTargets cone_targets;

  std::string command() const {
    return shell_quote(RICCI_FLOW_CLI_PATH) + " " +
           shell_quote(input_path.string()) +
           " --flatten-boundary --cone-targets " +
           shell_quote(target_path.string()) +
           " --target-mode additive --target-continuation"
           " --solver-diagnostics --output " +
           shell_quote(output_path.string());
  }
};

RicciConfig make_ricci_config(const std::filesystem::path& original_input_path,
                              gcs::SurfaceMesh& mesh,
                              gcs::VertexPositionGeometry& geometry,
                              const FrameFieldConeTargets& cone_targets) {
  const std::filesystem::path output_directory =
      std::filesystem::temp_directory_path() / "geometry-central-viewer";
  std::filesystem::create_directories(output_directory);

  std::string stem = original_input_path.stem().string();
  if (stem.empty()) {
    stem = "surface";
  }

  RicciConfig config{output_directory / (stem + ".frame-field-input.obj"),
                     output_directory / (stem + ".frame-field-targets.txt"),
                     output_directory / (stem + ".ricci-flat.obj"),
                     cone_targets};

  gcs::writeSurfaceMesh(mesh, geometry, config.input_path.string(), "obj");

  std::ofstream target_output(config.target_path);
  if (!target_output) {
    throw std::runtime_error("Could not create frame-field targets: " +
                             config.target_path.string());
  }
  write_obj_cone_targets(config.cone_targets, target_output);
  if (!target_output) {
    throw std::runtime_error("Failed while writing frame-field targets: " +
                             config.target_path.string());
  }

  return config;
}

struct RicciJob {
  RicciConfig config;
  std::future<int> result;

  void start(RicciConfig new_config) {
    config = std::move(new_config);
    const std::string command = config.command();
    result = std::async(std::launch::async,
                        [command]() { return std::system(command.c_str()); });
  }

  bool running() const { return result.valid(); }

  bool ready() const {
    return running() && result.wait_for(std::chrono::seconds(0)) ==
                            std::future_status::ready;
  }

  int finish() { return result.get(); }
};
```

# Mesh Placement

```cpp {name=flatten-definitions}
struct MeshBounds {
  gc::Vector3 min = gc::Vector3::infinity();
  gc::Vector3 max = -gc::Vector3::infinity();

  gc::Vector3 center() const { return 0.5 * (min + max); }

  double extent() const {
    const gc::Vector3 size = max - min;
    return std::max({size[0], size[1], size[2], 1e-6});
  }
};

MeshBounds mesh_bounds(gcs::SurfaceMesh& mesh,
                       const gcs::VertexPositionGeometry& geometry) {
  MeshBounds bounds;
  for (gcs::Vertex vertex : mesh.vertices()) {
    bounds.min =
        gc::componentwiseMin(bounds.min, geometry.vertexPositions[vertex]);
    bounds.max =
        gc::componentwiseMax(bounds.max, geometry.vertexPositions[vertex]);
  }
  return bounds;
}

void place_flattened_mesh_beside_original(
    gcs::SurfaceMesh& original_mesh,
    const gcs::VertexPositionGeometry& original_geometry,
    gcs::SurfaceMesh& flattened_mesh,
    gcs::VertexPositionGeometry& flattened_geometry) {
  const MeshBounds original_bounds =
      mesh_bounds(original_mesh, original_geometry);
  const MeshBounds flat_bounds =
      mesh_bounds(flattened_mesh, flattened_geometry);
  const double original_extent = original_bounds.extent();
  const double scale = 0.5 * original_extent / flat_bounds.extent();

  const gc::Vector3 original_center = original_bounds.center();
  const gc::Vector3 flat_center = flat_bounds.center();
  const gc::Vector3 target_center{
      original_bounds.max[0] + 0.4 * original_extent, original_center[1],
      original_center[2]};

  for (gcs::Vertex vertex : flattened_mesh.vertices()) {
    flattened_geometry.vertexPositions[vertex] =
        scale * (flattened_geometry.vertexPositions[vertex] - flat_center) +
        target_center;
  }
}
```

# Input Validation

The topology preflight is shared with the viewer's load-time report. Repeating
the inexpensive analysis when the button is pressed keeps the flattening
boundary honest if later viewer tools are allowed to modify mesh connectivity.

```cpp {name=flatten-definitions}
void validate_ricci_input(gcs::SurfaceMesh& mesh,
                          gcs::VertexPositionGeometry& geometry) {
  require_ricci_topology(analyze_ricci_topology(mesh));
  require_ricci_geometry(analyze_ricci_geometry(mesh, geometry));
}

struct FlattenState {
  std::string status;
  RicciJob job;
  EditableConeTargets editor;
  polyscope::PointCloud* candidate_cloud = nullptr;
  polyscope::PointCloudColorQuantity* candidate_colors = nullptr;
  std::size_t focused_candidate = 0;
  std::size_t colored_candidate = std::numeric_limits<std::size_t>::max();
  bool candidate_colors_dirty = true;
};
```

# Cone Editor

The frame field remains visible as the computed proposal, while this editor
stores an independent Ricci prescription. Every nonzero frame-field
singularity becomes a persistent candidate marker. Orange markers carry
positive prescribed index, blue markers carry negative index, gray markers are
disabled, and the candidate currently selected in Polyscope is yellow.

Clicking a marker only changes which row is focused. The checkbox and integer
input perform the mathematical edit; this separation prevents an exploratory
camera click from silently changing a target. Editing is disabled while a
solver process is running, and the process receives an immutable
`active_targets()` snapshot.

```cpp {name=flatten-definitions}
std::vector<glm::vec3> candidate_colors(const EditableConeTargets& editor,
                                        std::size_t focused_candidate) {
  std::vector<glm::vec3> colors;
  colors.reserve(editor.candidates.size());
  for (std::size_t i = 0; i < editor.candidates.size(); ++i) {
    const EditableConeTarget& candidate = editor.candidates[i];
    if (i == focused_candidate) {
      colors.emplace_back(1.0F, 0.85F, 0.1F);
    } else if (!candidate.selected || candidate.prescribed_index == 0) {
      colors.emplace_back(0.45F, 0.45F, 0.45F);
    } else if (candidate.prescribed_index > 0) {
      colors.emplace_back(0.95F, 0.3F, 0.08F);
    } else {
      colors.emplace_back(0.12F, 0.4F, 0.95F);
    }
  }
  return colors;
}

void refresh_candidate_colors(FlattenState& state) {
  if (!state.candidate_cloud || !state.candidate_colors ||
      state.editor.candidates.empty()) {
    return;
  }
  if (!state.candidate_colors_dirty &&
      state.colored_candidate == state.focused_candidate) {
    return;
  }

  state.candidate_colors->updateData(
      candidate_colors(state.editor, state.focused_candidate));
  state.colored_candidate = state.focused_candidate;
  state.candidate_colors_dirty = false;
}

void follow_polyscope_candidate_selection(FlattenState& state) {
  if (!state.candidate_cloud || !polyscope::haveSelection()) {
    return;
  }

  const polyscope::PickResult selection = polyscope::getSelection();
  if (selection.structure != state.candidate_cloud) {
    return;
  }

  const polyscope::PointCloudPickResult point =
      state.candidate_cloud->interpretPickResult(selection);
  if (point.index >= 0 &&
      static_cast<std::size_t>(point.index) < state.editor.candidates.size()) {
    state.focused_candidate = static_cast<std::size_t>(point.index);
  }
}

void draw_cone_editor(FlattenState& state) {
  if (!ImGui::CollapsingHeader("Cone singularities",
                               ImGuiTreeNodeFlags_DefaultOpen)) {
    return;
  }

  const FrameFieldConeTargets active = state.editor.active_targets();
  ImGui::Text("Active cones: %zu / %zu", active.targets.size(),
              state.editor.candidates.size());
  ImGui::Text("Index sum: %lld / %lld",
              static_cast<long long>(active.selected_index_sum),
              static_cast<long long>(active.required_index_sum));
  ImGui::Text("Boundary residual: %.6g radians", active.curvature_residual);
  if (!active.satisfies_gauss_bonnet()) {
    ImGui::TextWrapped(
        "Additive mode assigns this residual to unselected "
        "boundary vertices.");
  }

  ImGui::BeginDisabled(state.job.running());
  if (ImGui::Button("Reset to frame field")) {
    state.editor.reset_to_frame_field();
    state.candidate_colors_dirty = true;
  }
  ImGui::SameLine();
  if (ImGui::Button("Deselect all")) {
    state.editor.deselect_all();
    state.candidate_colors_dirty = true;
  }

  ImGui::TextWrapped(
      "Click a marker to focus it. Active controls whether the cone "
      "is sent to Ricci flow; one index unit equals 2pi/%d.",
      state.editor.field_symmetry);

  ImGui::BeginChild("cone candidate list", ImVec2(0.0F, 260.0F), true);
  for (std::size_t i = 0; i < state.editor.candidates.size(); ++i) {
    EditableConeTarget& candidate = state.editor.candidates[i];
    ImGui::PushID(static_cast<int>(i));

    bool selected = candidate.selected;
    if (ImGui::Checkbox("##active", &selected)) {
      candidate.selected = selected;
      state.candidate_colors_dirty = true;
    }
    ImGui::SameLine();
    ImGui::SetNextItemWidth(100.0F);
    int prescribed_index = candidate.prescribed_index;
    if (ImGui::InputInt("##index", &prescribed_index, 1, 4)) {
      candidate.prescribed_index = prescribed_index;
      candidate.selected = prescribed_index != 0;
      state.candidate_colors_dirty = true;
    }
    ImGui::SameLine();

    std::ostringstream label;
    label << "vertex " << candidate.vertex_index << "  (frame "
          << candidate.original_frame_index << ")";
    if (ImGui::Selectable(label.str().c_str(), state.focused_candidate == i)) {
      state.focused_candidate = i;
    }

    ImGui::PopID();
  }
  ImGui::EndChild();
  ImGui::EndDisabled();
}
```

# Callback Body

```cpp {name=flatten-callback-body}
follow_polyscope_candidate_selection(*state);
refresh_candidate_colors(*state);
draw_cone_editor(*state);

ImGui::BeginDisabled(state->job.running());
const bool flatten_requested = ImGui::Button("Flatten");
ImGui::EndDisabled();

if (flatten_requested) {
  try {
    validate_ricci_input(mesh, geometry);

    const FrameFieldConeTargets active_targets = state->editor.active_targets();
    RicciConfig config =
        make_ricci_config(input_path, mesh, geometry, active_targets);
    std::ostringstream status;
    status << "Running Ricci flow with " << config.cone_targets.targets.size()
           << " active cones"
           << "\nIndex sum: " << config.cone_targets.selected_index_sum << " / "
           << config.cone_targets.required_index_sum
           << "\nBoundary curvature residual: "
           << config.cone_targets.curvature_residual;
    state->status = status.str();
    state->job.start(std::move(config));
  } catch (const std::exception& error) {
    state->status = std::string("Flatten failed: ") + error.what();
  }
}

if (state->job.ready()) {
  try {
    const int exit_status = state->job.finish();
    if (exit_status != 0) {
      throw std::runtime_error(
          "Ricci flow failed with status " + std::to_string(exit_status) +
          "; targets: " + state->job.config.target_path.string());
    }

    auto [flattened_mesh, flattened_geometry] =
        gcs::readSurfaceMesh(state->job.config.output_path.string());
    place_flattened_mesh_beside_original(mesh, geometry, *flattened_mesh,
                                         *flattened_geometry);

    polyscope::registerSurfaceMesh("flattened mesh",
                                   flattened_geometry->vertexPositions,
                                   flattened_mesh->getFaceVertexList());
    polyscope::view::resetCameraToHomeView();

    std::ostringstream status;
    status << "Loaded Ricci mesh using "
           << state->job.config.cone_targets.targets.size() << " active cones"
           << "\nIndex sum: "
           << state->job.config.cone_targets.selected_index_sum << " / "
           << state->job.config.cone_targets.required_index_sum
           << "\nBoundary curvature residual: "
           << state->job.config.cone_targets.curvature_residual
           << "\nTargets: " << state->job.config.target_path.string()
           << "\nOutput: " << state->job.config.output_path.string();
    state->status = status.str();
  } catch (const std::exception& error) {
    state->status = std::string("Flatten failed: ") + error.what();
  }
}

if (!state->status.empty()) {
  ImGui::TextWrapped("%s", state->status.c_str());
}
```

# Implementation File

```cpp {name=flatten-source tangle=src/flatten.cpp}
#include @<flatten-implementation-includes@>

namespace gc = geometrycentral;
namespace gcs = geometrycentral::surface;

namespace {
@<flatten-definitions@>
}

std::function<void()> make_flatten_callback(
    std::string input_path,
    gcs::SurfaceMesh& mesh,
    gcs::VertexPositionGeometry& geometry,
    gcs::VertexData<int>& frame_indices,
    int field_symmetry) {
  auto state = std::make_shared<FlattenState>();
  state->editor =
      derive_editable_cone_targets(mesh, frame_indices, field_symmetry);

  std::vector<gc::Vector3> candidate_positions;
  candidate_positions.reserve(state->editor.candidates.size());
  const gcs::VertexData<std::size_t> dense_indices = mesh.getVertexIndices();
  std::vector<gcs::Vertex> vertices_by_index(mesh.nVertices());
  for (gcs::Vertex vertex : mesh.vertices()) {
    vertices_by_index[dense_indices[vertex]] = vertex;
  }
  for (const EditableConeTarget& candidate : state->editor.candidates) {
    candidate_positions.push_back(
        geometry.vertexPositions[vertices_by_index[candidate.vertex_index]]);
  }

  if (!candidate_positions.empty()) {
    state->candidate_cloud =
        polyscope::registerPointCloud("cone candidates", candidate_positions);
    state->candidate_cloud->setPointRadius(0.012);
    state->candidate_colors = state->candidate_cloud->addColorQuantity(
        "prescribed cone state",
        candidate_colors(state->editor, state->focused_candidate));
    state->candidate_colors->setEnabled(true);
    state->candidate_colors_dirty = false;
    state->colored_candidate = state->focused_candidate;
  }

  return [state, input_path = std::move(input_path), &mesh, &geometry]() {
    @<flatten-callback-body@>
  };
}
```
