# Setup commandline options handling

```cpp {name=cmdline-defs}
struct Options
{
    const char* mesh = nullptr;
    bool require_manifold = true;
};

void print_usage(const char* program)
{
    std::cerr << "Usage:\n  " << program << " [options] [mesh]\n\n"
              << "Options:\n"
              << "  --nonmanifold    Ignore manifold requirement when loading "
                 "the mesh\n"
              << "  -h, --help    Show this help text\n";
}

Options parse_options(int argc, char** argv)
{
    Options options;

    for (int i = 1; i < argc; ++i)
    {
        const std::string arg = argv[i];

        if (arg == "--nonmanifold")
        {
            options.require_manifold = false;
        }
        else if (arg == "-h" || arg == "--help")
        {
            print_usage(argv[0]);
            std::exit(EXIT_SUCCESS);
        }
        else if (!options.mesh)
        {
            options.mesh = argv[i];
        }
        else
        {
            std::cerr << "Unexpected argument: " << arg << '\n';
            print_usage(argv[0]);
            std::exit(EXIT_FAILURE);
        }
    }

    return options;
}
```


# Handle-cmdline

```cpp {name=handle-cmdline}
const Options options = parse_options(argc, argv);
if (!options.mesh)
{
    print_usage(argv[0]);
    return EXIT_FAILURE;
}
```

