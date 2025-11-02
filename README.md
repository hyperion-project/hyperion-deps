# hyperion-deps

This repository is responsible for creating Hyperion.NG dependencies for Linux, macOS and Windows.
This allows us to speed up the creation process of our build artifacts.

## Qt6 Static Build

As of the latest update, this repository builds Qt6 (version 6.8.2) statically from source for all platforms.
This ensures:
- Consistent Qt version across all platforms
- Static linking for reduced runtime dependencies
- Portable binaries

### Build Configuration

- **Qt Version**: 6.8.2
- **Build Type**: Static
- **Modules Built**:
  - **qtbase**: Core, Gui, Network, Sql, Widgets
  - **qtserialport**: SerialPort
  - **qtwebsockets**: WebSockets
- **Disabled Features**: DBus (optional), OpenSSL, ICU

### Build Time

Building Qt6 from source is time-intensive:
- **First build**: 45-180 minutes depending on hardware (increased due to additional modules)
- **Cached builds**: Much faster due to GitHub Actions caching

### Platform Support

Qt6 6.8.2 static builds are created for:
- **Linux**: All Debian distributions (Bullseye, Bookworm, Trixie) and architectures (amd64, armv6, armv7, arm64)
  - Note: Debian Bullseye armv6 uses Qt 5.15.2 due to Docker image availability
- **macOS**: arm64 and x64 architectures
- **Windows**: arm64 and x64 architectures
