# SodaOS

A developer-focused Linux distribution that combines the best aspects of Pop!_OS, Ubuntu, and Fedora.

## Vision

SodaOS aims to provide developers with a polished, productive Linux environment that features:

📋 **[View the detailed technical plan →](PLAN.md)**

- **Clean GNOME Desktop**: Vanilla GNOME experience without heavy customizations
- **Btrfs Filesystem**: Modern filesystem with snapshots and better reliability (like Fedora)
- **Developer-First Philosophy**: Pre-configured development tools and optimized workflows (like Pop!_OS)
- **Ubuntu Foundation**: Built on Ubuntu 24.04 LTS for stability and compatibility
- **Hardware Excellence**: Superior hardware support, especially for NVIDIA GPUs

## Key Features

### Filesystem & System
- **Btrfs by default** with automatic snapshots via Timeshift
- **Subvolume structure** similar to Fedora (@, @home, @var, etc.)
- **Flatpak-first** application distribution
- **Optimized boot times** and system performance

### Desktop Experience
- **Vanilla GNOME Shell** - clean, unmodified upstream experience
- **Improved font rendering** and HiDPI support
- **Developer-friendly shortcuts** and workflows
- **Clean theme** with professional aesthetics

### Developer Tools
- **Pre-installed development stack** (Git, build tools, IDEs)
- **Container runtime** ready (Docker/Podman)
- **Multiple language support** (Python, Node.js, Rust, Go, etc.)
- **VS Code integration** and extensions

### Hardware Support
- **NVIDIA drivers** pre-installed and configured
- **Gaming optimizations** for development workstations
- **Power management** optimized for laptops and desktops
- **Modern hardware** support out-of-the-box

## Development Status

🚧 **Early Development Phase** 🚧

This project is in the initial planning and development stage. We are currently:

- [ ] Setting up build infrastructure
- [ ] Designing the base system architecture
- [ ] Creating installation and configuration scripts
- [ ] Planning the initial release roadmap

## Project Structure

```
soda_os/
├── build/              # Build scripts and configuration
├── config/             # System configuration files
├── packages/           # Custom packages and repositories
├── scripts/            # Installation and setup scripts
├── docs/               # Documentation
└── testing/            # Testing and validation tools
```

## Contributing

This project is in early development. Contributions, ideas, and feedback are welcome!

## License

TBD - Will be determined as the project develops

## Contact

Project maintained by Jagmeet Chawla

---

*SodaOS - Refreshingly simple Linux for developers*