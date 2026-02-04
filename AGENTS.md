# AGENTS.md - GoBeep Project Documentation

## Overview

**GoBeep** (beeep) is a cross-platform Go library for sending desktop notifications, alerts, and beeps. It provides a unified API that abstracts away platform-specific differences, allowing developers to create consistent notification experiences across Windows, macOS, Linux, and web environments.

**Repository:** github.com/gen2brain/beeep  
**Go Version:** 1.21.5+  
**License:** See LICENSE file

---

## Project Purpose

GoBeep solves the problem of inconsistent notification APIs across different operating systems. It provides:

- **Desktop Notifications**: Toast-style notifications with title, message, and optional icons
- **Alerts**: Notifications with accompanying sound effects
- **System Beeps**: PC speaker beeps with configurable frequency and duration

The library is particularly useful for:
- Command-line tools that need user attention
- Background services that require status updates
- Applications running on multiple platforms
- WebAssembly applications targeting browsers

---

## Architecture

### Design Philosophy

GoBeep uses Go's build tags to provide platform-specific implementations while maintaining a single, clean API surface. This design pattern ensures:

1. **Zero runtime overhead**: Only the target platform's code is compiled
2. **Type safety**: Compile-time checking of platform compatibility
3. **Clean abstractions**: Developers use the same functions regardless of platform

### File Structure

```
beeep.go                 # Core package documentation and utilities
├── beep_*.go            # Platform-specific beep implementations
├── notify_*.go          # Platform-specific notification implementations
├── alert_*.go           # Platform-specific alert implementations
└── *_test.go            # Test files
```

**Platform-specific files:**
- `*_windows.go` - Windows implementations
- `*_darwin.go` - macOS implementations
- `*_unix.go` - Linux/BSD implementations
- `*_js.go` - WebAssembly/JavaScript implementations
- `*_unsupported.go` - Fallback for unsupported platforms

---

## Core API

### Public Variables

```go
var AppName string = "DefaultAppName"
```
- **Purpose**: Sets the application name for notifications
- **Usage**: Set before calling Notify/Alert to customize the sender name
- **Platform Behavior**: 
  - Windows: Used as AppID in toast notifications
  - Linux: Used with `-a` flag in notify-send
  - macOS: Used as notification group ID

### Functions

#### `Beep(freq float64, duration int) error`
Plays a system beep sound.

**Parameters:**
- `freq`: Frequency in Hz (platform-specific defaults apply)
- `duration`: Duration in milliseconds

**Platform-specific behavior:**
- **Windows**: Uses `kernel32.dll` Beep function via syscall
  - Frequency range: 37-32767 Hz
  - Default: 587 Hz (middle A), 500ms
- **macOS**: Uses `osascript beep` or bell character (ASCII 7)
  - Parameters ignored (system default sound)
- **Linux**: Writes to `/dev/input/by-path/platform-pcspkr-event-spkr`
  - Frequency range: 0-20000 Hz
  - Default: 440 Hz (A4), 200ms
  - Falls back to bell character if device unavailable
- **Web**: Plays embedded WAV file via HTML5 Audio

**Error Handling:** Returns error if beep cannot be played

---

#### `Notify(title, message string, icon any) error`
Sends a desktop notification.

**Parameters:**
- `title`: Notification title
- `message`: Notification body text
- `icon`: Either:
  - `string`: Path to PNG file
  - `[]byte`: PNG image data (embedded)
  - Platform-specific stock icon names

**Platform-specific behavior:**

##### Windows
- **Windows 10/11**: Uses Windows Runtime COM API (go-toast)
  - Supports PNG icons
  - Duration: Short (<10s) or Long (>10s)
  - Silent by default
- **Windows 7**: Uses Win32 API with system tray balloons
  - Requires icon conversion to ICO format
  - Displays for 5 seconds (configurable via timeout variable)

##### macOS
- **Primary**: Uses `terminal-notifier` command-line tool
  - Converts PNG to ICNS format
  - Supports sound with `-sound default` flag
- **Fallback**: Uses AppleScript via `osascript`
  - No icon support
  - No sound support
- **Note**: Both methods respect AppName as group ID

##### Linux/Unix
- **Primary**: D-Bus notification system (godbus/dbus)
  - Full PNG support via image hints
  - Urgency levels: Normal/Critical
  - Sound hints for critical notifications
- **Fallback 1**: `notify-send` command
  - Requires libnotify-bin
  - Supports PNG icons
  - Timeout configurable
- **Fallback 2**: `kdialog` (KDE)
  - Passive popup dialogs
  - Icon support
- **Build tag**: Use `nodbus` to skip D-Bus implementation

##### WebAssembly
- Uses browser's Notification API
- Supports data URIs for icons
- **Firefox**: Works immediately
- **Chrome**: Requires:
  - User gesture (click, keypress)
  - HTTPS (not HTTP)
  - Permission granted

**Error Handling:** Returns error if notification cannot be sent or invalid icon type

---

#### `Alert(title, message string, icon any) error`
Displays a notification with accompanying sound.

**Parameters:** Same as `Notify()`

**Behavior:**
- Calls `Notify()` with urgent/critical flag
- Plays system beep after notification
- Platform-specific urgency:
  - Windows: Default sound enabled
  - macOS: Default sound
  - Linux: Critical urgency + bell sound hint
  - Web: Same as Notify (browser controls sound)

---

## Build Tags

### `nodbus`
Disables D-Bus implementation on Linux/Unix systems.

**Usage:**
```bash
go build -tags nodbus
```

**Benefits:**
- Reduces dependencies (removes godbus/dbus)
- Smaller binary size
- Useful for environments without D-Bus

**Trade-offs:**
- Falls back to `notify-send` or `kdialog`
- May have reduced functionality

---

## Dependencies

### Core Dependencies
```go
require (
    git.sr.ht/~jackmordaunt/go-toast v1.1.2      // Windows toast notifications
    github.com/esiqveland/notify v0.13.3          // Linux D-Bus notifications
    github.com/godbus/dbus/v5 v5.1.0              // D-Bus communication
    github.com/jackmordaunt/icns/v3 v3.0.1        // macOS ICNS encoding
    github.com/sergeymakinen/go-ico v1.0.0-beta.0 // Windows ICO encoding
    github.com/tadvi/systray v0.0.0-20190226123456-11a2b8fa57af // Windows system tray
    golang.org/x/sys v0.30.0                      // System calls
)
```

### Indirect Dependencies
```go
github.com/go-ole/go-ole v1.3.0          // Windows COM
github.com/nfnt/resize v0.0.0-20180221191011-83c6a9932646 // Image resizing
github.com/sergeymakinen/go-bmp v1.0.0  // BMP format support
```

---

## Platform-Specific Details

### Windows

#### Requirements
- Windows 7+ (Windows 10/11 recommended)
- No external dependencies required

#### Implementation Details
- **Registry Detection**: Checks `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion` for version
- **Icon Conversion**: PNG → ICO conversion for Windows 7 balloon notifications
- **System Tray**: Uses temporary system tray icon for balloon notifications
- **COM API**: Uses Windows Runtime COM API for Windows 10/11 toasts

#### Limitations
- Balloon notifications (Windows 7) require system tray
- Icon conversion creates temporary files
- 100ms sleep after toast notification for Windows 10/11

---

### macOS

#### Requirements
- macOS 10.9+
- Optional: `terminal-notifier` for enhanced features
- Built-in: `osascript` always available

#### Implementation Details
- **Icon Conversion**: PNG → ICNS format using `jackmordaunt/icns`
- **Fallback Strategy**: Tries terminal-notifier, then osascript
- **Sound**: Default system sound via terminal-notifier

#### Limitations
- osascript fallback doesn't support icons
- osascript fallback doesn't support sound
- Terminal-notifier must be installed separately

---

### Linux/Unix

#### Requirements
- D-Bus session bus (recommended)
- Optional: `notify-send` (libnotify-bin)
- Optional: `kdialog` (KDE)
- For beeps: `pcspkr` module loaded, user in `input` group

#### Implementation Details
- **D-Bus**: Uses freedesktop.org Notification Specification
  - Supports image hints for inline icons
  - Urgency levels: Normal, Critical
  - Sound hints for critical notifications
- **Fallback**: notify-send with `-u` flag for urgency
- **Beeps**: Direct PC speaker access via `/dev/input/by-path/platform-pcspkr-event-spkr`

#### Permissions
```bash
# Add user to input group for PC speaker access
sudo usermod -a -G input $USER

# Load pcspkr module
sudo modprobe pcspkr

# Enable bell in X11 terminals
xset b on
```

#### Limitations
- PC speaker requires root or group membership
- D-Bus may not be available in all environments
- Fallback commands may not be installed

---

### WebAssembly

#### Requirements
- Modern browser with Notification API support
- HTTPS for Chrome (Firefox works on HTTP)
- User gesture for permission request (Chrome)

#### Implementation Details
- **Permission Handling**: Automatic permission request
- **Icons**: Supports data URIs for embedded images
- **Audio**: Embedded WAV file for beeps

#### Limitations
- Chrome requires user gesture and HTTPS
- No control over notification duration
- Browser controls notification behavior

---

## Usage Examples

### Basic Beep
```go
import "github.com/gen2brain/beeep"

err := beeep.Beep(beeep.DefaultFreq, beeep.DefaultDuration)
if err != nil {
    panic(err)
}
```

### Notification with File Icon
```go
err := beeep.Notify("Title", "Message body", "path/to/icon.png")
if err != nil {
    panic(err)
}
```

### Notification with Embedded Icon
```go
//go:embed testdata/info.png
var icon []byte

err := beeep.Notify("Title", "Message body", icon)
if err != nil {
    panic(err)
}
```

### Alert with Custom App Name
```go
beeep.AppName = "My Application"

err := beeep.Alert("Warning", "Something happened!", "warning.png")
if err != nil {
    panic(err)
}
```

### Cross-Platform Usage
```go
package main

import (
    "embed"
    "log"

    "github.com/gen2brain/beeep"
)

//go:embed icon.png
var icon []byte

func main() {
    beeep.AppName = "MyApp"

    // Send notification
    if err := beeep.Notify("Hello", "World!", icon); err != nil {
        log.Fatal(err)
    }

    // Play beep
    if err := beeep.Beep(beeep.DefaultFreq, beeep.DefaultDuration); err != nil {
        log.Fatal(err)
    }
}
```

---

## Testing

### Test Files
- `beep_test.go` - Beep functionality tests
- `notify_test.go` - Notification tests
- `alert_test.go` - Alert tests
- `example_test.go` - Example usage tests

### Running Tests
```bash
# Run all tests
go test ./...

# Run tests with verbose output
go test -v ./...

# Run tests for specific platform
go test -tags nodbus ./...
```

### CI/CD
The project uses GitHub Actions for continuous integration:
- **Platforms**: Ubuntu, macOS, Windows
- **Go Versions**: 1.24.x
- **Triggers**: Push and pull requests

---

## Development Guidelines

### Adding Platform Support

1. Create platform-specific files using build tags:
   ```go
   //go:build yourplatform && !linux && !windows && !darwin && !js
   
   package beeep
   ```

2. Implement the three core functions:
   - `Beep(freq float64, duration int) error`
   - `Notify(title, message string, icon any) error`
   - `Alert(title, message string, icon any) error`

3. Follow existing patterns for icon handling:
   - Support both `string` (path) and `[]byte` (data)
   - Use `bytesToFilename()` for temporary file creation
   - Clean up temporary files with `defer os.Remove()`

4. Add error handling and fallback mechanisms

5. Update documentation and examples

---

### Code Style

- **Build Tags**: Use explicit negation of other platforms
- **Error Handling**: Always return errors, never panic in library code
- **Icon Handling**: Support both string paths and byte slices
- **Temporary Files**: Always clean up with `defer`
- **Documentation**: Add godoc comments for all exported functions

---

### Icon Conversion Utilities

The library provides helper functions for icon format conversion:

#### `bytesToFilename(data []byte) (string, error)`
Creates a temporary PNG file from byte data.

#### `pngToIco(icon string) (string, error)` (Windows)
Converts PNG to ICO format for Windows balloon notifications.

#### `pngToIcns(icon string) (string, error)` (macOS)
Converts PNG to ICNS format for macOS notifications.

---

## Troubleshooting

### Common Issues

#### Windows
- **Icon not showing**: Ensure PNG file is valid path
- **No sound on Alert**: Check system sound settings
- **Balloon not appearing**: Verify system tray is visible

#### macOS
- **Icon not showing**: Install `terminal-notifier` for full support
- **No sound**: Verify `terminal-notifier` is installed
- **Fallback to osascript**: Expected behavior without terminal-notifier

#### Linux
- **Permission denied**: Add user to `input` group for PC speaker
- **No D-Bus**: Use `nodbus` build tag
- **notify-send not found**: Install `libnotify-bin`
- **Beep not working**: Load `pcspkr` module, check `/dev/input/by-path/platform-pcspkr-event-spkr`

#### WebAssembly
- **Chrome no notification**: Ensure HTTPS and user gesture
- **Permission denied**: Call from button click handler
- **Icon not showing**: Use data URI format

---

## Contributing

### Pull Request Process

1. Fork the repository
2. Create a feature branch
3. Add tests for new functionality
4. Ensure all tests pass on all platforms
5. Update documentation
6. Submit pull request

### Testing Checklist

- [ ] Tests pass on Windows
- [ ] Tests pass on macOS
- [ ] Tests pass on Linux
- [ ] Tests pass with `nodbus` build tag
- [ ] Documentation updated
- [ ] Examples working

---

## Future Enhancements

Potential areas for improvement:

1. **Custom Sounds**: Support for custom sound files
2. **Action Buttons**: Add interactive buttons to notifications
3. **Progress Bars**: Display progress in notifications
4. **Rich Content**: Support HTML/markdown in message bodies
5. **More Icon Formats**: Support JPEG, SVG, WebP
6. **Notification History**: Track sent notifications
7. **Click Handlers**: Callbacks when notifications are clicked
8. **Timeout Control**: Per-notification timeout configuration

---

## License

See [LICENSE](LICENSE) file for details.

---

## Credits

- **Original Author**: gen2brain
- **Contributors**: See GitHub commit history

---

## Related Projects

- [go-toast](https://git.sr.ht/~jackmordaunt/go-toast) - Windows toast notifications
- [notify](https://github.com/esiqveland/notify) - Linux D-Bus notifications
- [icns](https://github.com/jackmordaunt/icns) - macOS ICNS encoding
- [go-ico](https://github.com/sergeymakinen/go-ico) - Windows ICO encoding

---

## Changelog

Key changes from recent commits:
- Fixed nodbus support (issue #68)
- Don't close shared dbus connection (improvement)
- Various platform-specific fixes

For complete history, see git log.
