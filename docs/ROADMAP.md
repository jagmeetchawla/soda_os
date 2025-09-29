# SodaOS Development Roadmap

## Phase 1: Foundation (Current)
- [x] Project setup and repository initialization
- [ ] Define core architecture and technical specifications
- [ ] Set up development environment on Ubuntu 24.04
- [ ] Create initial build system using live-build

## Phase 2: Base System
- [ ] Ubuntu 24.04 LTS base with Btrfs filesystem
- [ ] Vanilla GNOME desktop environment
- [ ] Basic package selection and customizations
- [ ] Initial ISO build and testing

## Phase 3: Developer Features
- [ ] Pre-installed development tools
- [ ] NVIDIA driver integration
- [ ] Container runtime configuration
- [ ] VS Code and development extensions

## Phase 4: Polish & Testing
- [ ] Hardware compatibility testing
- [ ] Performance optimizations
- [ ] Documentation and user guides
- [ ] Initial public release preparation

## Phase 5: Community & Maintenance
- [ ] Community feedback integration
- [ ] Update mechanisms
- [ ] Long-term maintenance planning
- [ ] Repository and package management

## Technical Decisions

### Base System
- **Ubuntu 24.04 LTS**: Stability and long-term support
- **Btrfs**: Modern filesystem with snapshots
- **Vanilla GNOME**: Clean desktop experience
- **Flatpak**: Primary application distribution method

### Build System
- **live-build**: Primary ISO creation tool
- **Debootstrap**: Base system creation
- **Custom scripts**: Additional customizations

### Hardware Support
- **NVIDIA drivers**: Pre-installed and configured
- **Modern hardware**: Focus on current developer workstations
- **Power management**: Laptop and desktop optimization