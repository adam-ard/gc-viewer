# Command-line options

The viewer accepts one required mesh path. It loads that mesh through Geometry
Central's manifold-specific interface by default, because the downstream
geometry algorithms assume manifold connectivity. The `--nonmanifold` option
relaxes that requirement for cases where merely inspecting a more general mesh
is useful.

# Dependencies

This section owns the standard-library facilities used by the command-line
model, parser, and diagnostics.

```cpp {name=cmdline-includes}
<cstdlib>
<iostream>
<string>
```

# Definition aggregate

The executable needs the option model, usage printer, and parser together.
Collecting them behind one callout keeps those implementation details out of
the top-level outline in [main.o.md](main.o.md).

```cpp {name=cmdline-defs}
@<cmdline-option-model@>
@<cmdline-usage-def@>
@<cmdline-parser-def@>
```

# Option model

`Options` is the command-line boundary between parsing and the viewer
pipeline. The mesh pointer refers to storage in `argv`, which remains valid for
the lifetime of `main()`, so this small model does not need to copy or own the
path.

```cpp {name=cmdline-option-model}
struct Options {
  const char* mesh = nullptr;
  bool require_manifold = true;
};
```

# Usage text

Angle brackets in the synopsis mark the mesh as required. Square brackets mark
the options as optional.

```cpp {name=cmdline-usage-def}
void print_usage(const char* program) {
  std::cerr << "Usage:\n  " << program << " [options] <mesh>\n\n"
            << "Options:\n"
            << "  --nonmanifold    Ignore manifold requirement when loading "
               "the mesh\n"
            << "  -h, --help       Show this help text\n";
}
```

# Parse the arguments

The parser examines arguments from left to right. Recognized options update the
model; the first other argument becomes the mesh path. A second such argument
is an error because the viewer operates on one mesh at a time. Help and
expected input errors are resolved locally before the geometry pipeline starts.

```cpp {name=cmdline-parser-def}
Options parse_options(int argc, char** argv) {
  Options options;

  for (int i = 1; i < argc; ++i) {
    const std::string arg = argv[i];

    if (arg == "--nonmanifold") {
      options.require_manifold = false;
    } else if (arg == "-h" || arg == "--help") {
      print_usage(argv[0]);
      std::exit(EXIT_SUCCESS);
    } else if (!options.mesh) {
      options.mesh = argv[i];
    } else {
      std::cerr << "Unexpected argument: " << arg << '\n';
      print_usage(argv[0]);
      std::exit(EXIT_FAILURE);
    }
  }

  return options;
}
```

# Connect parsing to `main()`

This chunk expands inside `main()`. It establishes the required-mesh invariant
before the viewer pipeline begins, allowing later sections to use
`options.mesh` without repeatedly checking it.

```cpp {name=handle-cmdline}
const Options options = parse_options(argc, argv);
if (!options.mesh) {
  print_usage(argv[0]);
  return EXIT_FAILURE;
}
```
