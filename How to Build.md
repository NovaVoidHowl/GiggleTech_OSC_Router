# GiggleTech OSC Router - Build Guide

This guide provides step-by-step instructions for building the GiggleTech OSC Router from source on Linux and Windows.

## Prerequisites

### Common Requirements (All Platforms)

- **Rust Toolchain**: Version 1.56.0 or newer

  - Install from [rustup.rs](https://rustup.rs/) (or package manager, linux distro dependent)
  - Verify installation: `rustc --version` and `cargo --version`

- **Git**: For cloning the repository

  - Verify installation: `git --version`

### Platform-Specific Requirements

#### Linux

**Package Dependencies:**

```bash
# Debian/Ubuntu
sudo apt-get update
sudo apt-get install -y build-essential pkg-config libssl-dev

# Fedora/RHEL/CentOS
sudo dnf install gcc openssl-devel pkg-config

# Arch Linux
sudo pacman -S base-devel openssl pkg-config rustup
rustup default stable

```

**Optional (for testing):**

- `ping` utility (usually pre-installed)
- Network access to haptic devices on local network

#### Windows

**Required Tools:**

- **Visual Studio Build Tools** or **Visual Studio 2019/2022**
  - Install "Desktop development with C++" workload
  - Includes MSVC compiler and Windows SDK
  - Download from [visualstudio.microsoft.com](https://visualstudio.microsoft.com/downloads/)

**Optional Components:**

- **Bonjour Service** (for mDNS device discovery via `giggletech.local`)
  - Download from Apple Developer site or install iTunes/iCloud for Windows
- **giggletech_oscq.exe** (for OSCQuery integration)
  - Installed separately to `%LocalAppData%\Giggletech\`

## Build Instructions

### Clone the Repository

```bash
git clone https://github.com/your-org/GiggleTech_OSC_Router.git
cd GiggleTech_OSC_Router
```

### Configuration Setup

Before building, ensure you have a valid `config.yml` file:

1. Review the included `config.yml.example` and copy to `config.yml` customising as needed
2. Update device IP addresses to match your hardware
3. Adjust OSC port settings as needed

**Minimal config.yml:**

```yaml
devices:
  - ip: 192.168.1.69
    proximity_parameter: proximity_01

setup:
  port_rx: 9001
  default_min_speed: 5
  default_max_speed: 25
  default_start_tx: 20
  timeout: 5
  default_use_velocity_control: True
  default_outer_proximity: 0
  default_inner_proximity: 0.7
  default_velocity_scalar: 20
```

### Build Process

#### Option 1: Debug Build (Development)

```bash
# From workspace root
cargo build

# Binary location:
# target/debug/async-osc (Linux)
# target/debug/async-osc.exe (Windows)
```

#### Option 2: Release Build (Production)

```bash
# From workspace root
cargo build --release

# Binary location:
# target/release/async-osc (Linux)
# target/release/async-osc.exe (Windows)
```

**Note:** The package is named `async-osc` despite the folder being `giggletech-router`.

#### Option 3: Build and Run

```bash
# Debug mode
cargo run

# Release mode (recommended for actual use)
cargo run --release
```

### Build Troubleshooting

#### Common Issues

**1. "package `async-osc` cannot be built because it requires rustc 1.56.0 or newer"**

```bash
# Update Rust toolchain
rustup update stable
```

**2. OpenSSL errors (Linux)**

```bash
# Install OpenSSL development headers
sudo apt-get install libssl-dev pkg-config
```

**3. Linker errors (Windows)**

- Ensure Visual Studio Build Tools are installed with C++ workload
- Restart your terminal/IDE after installation

**4. "config.yml not found" at runtime**

- The config file must be in the **current working directory**, not the workspace root
- Copy `config.yml` to the directory where you run the binary

**5. YAML parsing errors**

- Check for missing colons (`:`) after keys
- Ensure proper indentation (spaces, not tabs)
- Validate structure matches the example above

## Running the Application

### Linux

```bash
# From workspace root with config.yml present
./target/release/async-osc

# Or specify working directory
cd /path/to/config
/path/to/GiggleTech_OSC_Router/target/release/async-osc
```

### Windows

```cmd
REM From workspace root with config.yml present
target\release\async-osc.exe

REM Or double-click the executable in Windows Explorer
REM (ensure config.yml is in the same directory)
```

### Runtime Requirements

1. **config.yml** in execution directory
2. Network connectivity to haptic devices
3. OSC client (e.g., VRChat) sending messages to configured port

### Logs and Debugging

- Application logs are written to `giggletech_log.txt` in the execution directory
- Check this file for startup messages, errors, and connectivity information
- Timestamped entries help track application behavior

## Development Workflow

### Running Tests

```bash
# Run all tests
cargo test

# Run tests with output
cargo test -- --nocaptures
```

### Checking Code

```bash
# Check for compilation errors without building
cargo check

# Run clippy for linting
cargo clippy

# Format code
cargo fmt
```

### Clean Build

```bash
# Remove build artifacts
cargo clean

# Then rebuild
cargo build --release
```

## Cross-Compilation

### Building for Windows on Linux

```bash
# Install Windows target
rustup target add x86_64-pc-windows-gnu

# Install mingw-w64
sudo apt-get install mingw-w64

# Build for Windows
cargo build --release --target x86_64-pc-windows-gnu
```

**Note:** Windows-specific features (OSCQuery, Bonjour, ping syntax) may not work in cross-compiled binaries.

### Building for Linux on Windows (WSL)

Use Windows Subsystem for Linux:

```bash
# In WSL terminal
cd /mnt/c/path/to/GiggleTech_OSC_Router
cargo build --release
```

## Dependencies Overview

The project uses these major Rust crates (automatically handled by Cargo):

- **async-std** (1.8.0) - Async runtime
- **tokio** (1.x) - Additional async support
- **rosc** (0.4.2) - OSC protocol implementation
- **serde** / **serde_yaml** - Configuration parsing
- **reqwest** (0.11) - HTTP client for OSCQuery
- **anyhow** (1.0) - Error handling
- **chrono** (0.4) - Timestamps for logging
- **lazy_static** (1.4) - Global state management

See [giggletech-router/Cargo.toml](giggletech-router/Cargo.toml) for the complete dependency list.

## Distribution

### Creating a Release Package

**Linux:**

```bash
cargo build --release
mkdir -p giggletech-osc-router-linux
cp target/release/async-osc giggletech-osc-router-linux/
cp config.yml giggletech-osc-router-linux/
cp README.md giggletech-osc-router-linux/
tar -czf giggletech-osc-router-linux.tar.gz giggletech-osc-router-linux/
```

**Windows:**

```cmd
cargo build --release
mkdir giggletech-osc-router-windows
copy target\release\async-osc.exe giggletech-osc-router-windows\
copy config.yml giggletech-osc-router-windows\
copy README.md giggletech-osc-router-windows\
REM Then zip the folder using Windows Explorer or 7-Zip
```

## Performance Tips

1. **Always use `--release` for production** - Debug builds are 10-100x slower
2. **Motor Speed Scale**: Never exceed 0.66 in code (motor longevity)
3. **Connection pooling**: Managed automatically by the global ConnectionManager
4. **Memory**: Release builds use ~10-20MB RAM with multiple devices

## Getting Help

- Check `giggletech_log.txt` for error messages
- Review [README.md](README.md) for configuration details
- See [.github/copilot-instructions.md](.github/copilot-instructions.md) for architecture details
- Verify device connectivity with ping: `ping 192.168.1.69`

## Additional Resources

- [Rust Book](https://doc.rust-lang.org/book/) - Learning Rust
- [Cargo Book](https://doc.rust-lang.org/cargo/) - Build system documentation
- [OSC Specification](http://opensoundcontrol.org/spec-1_0) - OSC protocol details
