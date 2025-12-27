# GiggleTech OSC Router - AI Coding Agent Instructions

## Project Overview

Async Rust OSC (Open Sound Control) router for GiggleTech haptic hardware (GigglePuck/GiggleSpark). Receives OSC messages from VRChat avatars and controls physical haptic devices via UDP. Built on a heavily modified `async-osc` library with workspace architecture.

## Architecture

### Component Structure

- **giggletech-router/**: Main router binary (renamed package: `async-osc`)
  - Core OSC router with device management and signal processing
  - Uses async-std runtime (NOT tokio for main execution, though tokio is a dep)
- **giggletech-web/**: Web interface (basic HTML/JS, appears minimal/future feature)

### Key Data Flow

1. OSC messages received on configurable port (default 9001 or via OSCQuery)
2. Messages parsed by address (proximity parameters, max speed parameters)
3. Per-device proximity signals processed into motor control values
4. Motor commands sent via UDP to device IPs with connection pooling
5. Timeout monitors stop devices if signals cease for N seconds

### Critical Patterns

#### Device Configuration (config.yml)

- YAML-based with `setup` (global defaults) and `devices` (per-device overrides)
- Devices identified by IP address (192.168.x.x)
- Each device has `proximity_parameter` (e.g., `proximity_01`) and optional `max_speed_parameter`
- Two control modes: proximity-based or velocity-based (see `use_velocity_control`)
- Configuration validated on load with detailed error messages showing line numbers (see [yaml_validator.rs](../giggletech-router/src/config/yaml_validator.rs))

#### OSC Message Routing Pattern

In [main.rs](../giggletech-router/src/main.rs#L183-L220):

```rust
// Match OSC address against device parameters
if address == *device.max_speed_parameter {
    // Update max speed limit
} else if address == *device.proximity_parameter {
    // Process proximity signal → motor control
}
```

#### Motor Control Constants

- `MOTOR_SPEED_SCALE = 0.66` in [data_processing.rs](../giggletech-router/src/data_processing.rs#L69) - NEVER exceed (motor longevity)
- Start transmission (`start_tx`) ensures motors don't stall on low initial values
- Velocity mode calculates delta between signals over time for dynamic response

#### Connection Management

- Global `ConnectionManager` with connection pooling ([giggletech_osc.rs](../giggletech-router/src/giggletech_osc.rs#L30-L90))
- Sockets created per-message with timeout handling (not persistent connections)
- Statistics tracking: connection_count, success_count, error_count per device
- Automatic cleanup of stale connections after 300s

#### Timeout Pattern

- Each device has async timeout loop spawned in [main.rs](../giggletech-router/src/main.rs#L162-L165)
- Uses `DEVICE_LAST_SIGNAL_TIME` global HashMap (lazy_static, Arc<Mutex>)
- Sends stop signals (0) if no OSC message received within timeout period
- "Terminator" worker continuously sends 0 when proximity stops ([terminator.rs](../giggletech-router/src/terminator.rs))

## Development Workflows

### Build & Run

```bash
# Build from workspace root
cargo build --release

# Run (requires config.yml in execution directory)
cargo run --release

# Package name is "async-osc", not "giggletech-router"
```

### Configuration Testing

- Uses `config.yml` in current directory (NOT workspace root when running from subdirectory)
- Test configuration with giggletech_vrc_simulator.exe (Windows-only test tool, see [README](../README.md#L82))
- Check logs in `giggletech_log.txt` (timestamped entries)

### OSCQuery Integration

- If `port_rx: OSCQuery` set, spawns `giggletech_oscq.exe` from AppData\\Local\\Giggletech
- Reads HTTP port from `config_oscq.yml`, retrieves UDP port dynamically
- Process auto-restarts if UDP port invalid ([oscq_giggletech.rs](../giggletech-router/src/config/oscq_giggletech.rs))

### Dependency Notes

- Both `async-std` and `tokio` present (use async-std for consistency)
- `rosc` crate provides OSC types (re-exported in [lib.rs](../giggletech-router/src/lib.rs))
- `lazy_static` for global state (DEVICE_LAST_SIGNAL_TIME, DEVICE_LAST_VALUE, CONNECTION_MANAGER)
- Error handling via `anyhow::Result` in modules, `async_osc::Result` in main

## Project-Specific Conventions

### Module Documentation

Every module has extensive block comments explaining purpose, key features, and usage. Follow this pattern when creating new modules.

### Logging Pattern

```rust
fn log_to_file(message: &str) {
    let timestamp = Local::now().format("%Y-%m-%d %H:%M:%S");
    // Append to giggletech_log.txt
}
```

Use for startup, config loading, errors, connectivity tests.

### Proximity Processing Modes

1. **Proximity Mode** (`use_velocity_control: False`): Direct scaling of proximity value
2. **Velocity Mode** (`use_velocity_control: True`): Calculates velocity from proximity delta over time
   - Uses `outer_proximity`, `inner_proximity`, `velocity_scalar` parameters
   - See [handle_proximity_parameter.rs](../giggletech-router/src/handle_proximity_parameter.rs#L47-L86)

### Error Handling

- Panic hook logs to file before crash ([main.rs](../giggletech-router/src/main.rs#L97-L100))
- Console kept open after errors ("Press Enter to exit...")
- Config errors return detailed line-level diagnostics

### State Management

- `AtomicBool running` passed to terminator for worker control
- Per-device state in Arc\<Mutex<HashMap>> for thread-safe access
- Device configs cloned when passing to async tasks (DeviceConfig is Clone)

## External Integration Points

### VRChat Avatar Communication

- Avatars send float values (0.0-1.0) on proximity parameters via OSC
- Address format: `/avatar/parameters/{proximity_parameter}`
- Max speed controlled via separate parameter (e.g., `/avatar/parameters/max_speed_04`)

### Hardware Communication

- UDP packets to `{device_ip}:9001` with motor values (i32)
- No response expected (fire-and-forget)
- Device LED behavior indicates connection status (see README device setup)

### Windows-Specific Features

- Bonjour service required for `giggletech.local` mDNS resolution
- System tray minimization mentioned as future feature ([main.rs](../giggletech-router/src/main.rs#L41-L42))
- Ping test uses `ping -n 1 -w 1000` Windows syntax

## Testing & Debugging

### Device Connectivity Test

Function `test_device_connectivity()` in [main.rs](../giggletech-router/src/main.rs#L235-L267) pings all configured devices. Call before main loop when debugging connectivity.

### Proximity Visualization

`proximity_graph()` creates ASCII representation of signal strength: `------>` (see [data_processing.rs](../giggletech-router/src/data_processing.rs#L50-L55))

### Connection Statistics

Global CONNECTION_MANAGER tracks per-device stats. Use `get_stats()` for debugging connection issues.

## Common Pitfalls

1. **Don't exceed MOTOR_SPEED_SCALE 0.66** - motor longevity critical
2. **Config.yml location matters** - must be in execution directory, not workspace root
3. **Async runtime mixing** - prefer async-std over tokio for new code
4. **Device IP format** - no `http://` prefix, just IP address (e.g., `192.168.1.69`)
5. **Port numbers** - OSC RX port (9001), device TX port (9001), HTTP OSCQuery (variable)
6. **Mutex poisoning** - timeout module handles poisoned mutexes gracefully ([osc_timeout.rs](../giggletech-router/src/osc_timeout.rs#L41-L47))
