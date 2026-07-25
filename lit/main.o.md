# Executable Assembly

This file is the top-level outline of the viewer executable. It assembles
feature-specific definitions and dependencies from the literate sections that
explain them; the only implementation logic kept here is the process boundary:
parse the command line, run the viewer, and translate exceptions into a
nonzero exit status.

# Includes

Each contributing section owns the headers required by its code. Repeated
headers are intentional: local dependency ownership is more useful here than
deduplicating a harmless preprocessor operation.

```cpp {name=main-includes}
@<cmdline-includes@>
@<pipeline-includes@>

<cstdlib>
<exception>
<iostream>
```

# Main File

Expected command-line mistakes are still reported locally by the parser.
Placing that work inside the outer `try` also ensures that any unexpected
parser failure remains inside the executable's single exception boundary.

The helper definitions assembled below are implementation details of the
executable. The anonymous namespace gives them internal linkage, preventing
their names from colliding with definitions in other translation units.
`main()` remains outside because C++ requires it in the global namespace.

```cpp {name=main tangle=src/main.cpp}
#include @<main-includes@>

@<pipeline-namespace-aliases@>

namespace {
@<cmdline-defs@>
@<pipeline-defs@>
}  // namespace

int main(int argc, char** argv) {
  try {
    @<handle-cmdline@>
    @<pipeline-code@>
  } catch (const std::exception& error) {
    std::cerr << "Error: " << error.what() << '\n';
    return EXIT_FAILURE;
  }

  return EXIT_SUCCESS;
}
```
