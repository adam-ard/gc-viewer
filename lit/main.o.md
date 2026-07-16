# Includes

```cpp {name=includes}
"flatten.h"

"geometrycentral/surface/manifold_surface_mesh.h"
"geometrycentral/surface/meshio.h"
"geometrycentral/surface/surface_mesh.h"
"geometrycentral/surface/vertex_position_geometry.h"
"geometrycentral/surface/direction_fields.h"

"polyscope/polyscope.h"
"polyscope/surface_mesh.h"

<cstdlib>
<exception>
<iostream>
<memory>
<string>
<tuple>
```

# Main File

```cpp {name=main tangle=src/main.cpp}
#include @<includes@>

namespace gc = geometrycentral;
namespace gcs = geometrycentral::surface;

namespace {
    @<cmdline-defs@>
    @<definitions@>
}

int main(int argc, char** argv)
{
	@<handle-cmdline@>

    try
    {
        @<main-code@>
	}
    catch (const std::exception& error)
    {
        std::cerr << "Error: " << error.what() << '\n';
        return EXIT_FAILURE;
    }

    return EXIT_SUCCESS;
}
```
