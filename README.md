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
- **Build Type**: Static (minimal configuration)
- **Components**: Qt Core and Qt Network only
- **Disabled Features**: GUI, Widgets, SQL, DBus, OpenSSL, ICU

### Build Time

Building Qt6 from source is time-intensive:
- **First build**: 30-120 minutes depending on hardware
- **Cached builds**: Much faster due to GitHub Actions caching

### Platform Support

Qt6 6.8.2 static builds are created for:
- **Linux**: All Debian distributions (Bullseye, Bookworm, Trixie) and architectures (amd64, armv6, armv7, arm64)
- **macOS**: arm64 and x64 architectures
- **Windows**: arm64 and x64 architectures
