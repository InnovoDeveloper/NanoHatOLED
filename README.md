# NanoHat OLED - Debian Trixie/Bookworm Edition

Modern, systemd-compatible NanoHat OLED driver for FriendlyELEC boards running Debian 12/13.

## Features

- **Modern GCC Support**: Builds cleanly on Debian 12 (Bookworm) / Debian 13 (Trixie) with GCC 12+
- **Pillow 10+ Compatible**: Works with latest Python Pillow library
- **Systemd Integration**: Proper service management with clean startup/shutdown
- **Auto-detection**: Automatically detects Python 3 interpreter version
- **Fixed Process Management**: No zombie processes or service timeout issues

## What's Included

- **C Daemon** (`NanoHatOLED`): Manages OLED display and hardware buttons (K1, K2, K3)
- **BakeBit Integration**: Uses original FriendlyELEC BakeBit drivers with compatibility fixes
- **Installation Scripts**:
  - `install-local.sh` - Local installation (recommended for development/testing)
  - `install-compat.sh` - Legacy compatible installation
  - `install.sh` - Standard installation
- **Systemd Service**: Managed service with proper signal handling and cleanup

## Requirements

- Debian 12 (Bookworm) or Debian 13 (Trixie)
- FriendlyELEC NanoPi board with NanoHat OLED accessory
- I²C bus enabled (`/dev/i2c-0` or `/dev/i2c-1`)
- Root/sudo access
- Internet connection (for first-time package installation)

## Installation

### First-Time Setup (with package installation)

```bash
# Clone repository (with submodules)
cd ~
git clone --recursive https://github.com/InnovoDeveloper/NanoHatOLED.git
cd NanoHatOLED

# Run installation (installs packages, compiles daemon, sets up service)
sudo ./install-local.sh

# Start the service
sudo systemctl start nanohat-oled.service

# Check status
systemctl status nanohat-oled.service
```

### Quick Reinstall (packages already installed)

If you need to rebuild/reinstall without reinstalling system packages:

```bash
cd ~/NanoHatOLED

# Stop service and clean up
sudo systemctl stop nanohat-oled.service
sudo systemctl reset-failed nanohat-oled.service
sudo killall -9 NanoHatOLED python3 2>/dev/null || true

# Reinstall (packages already installed, so faster)
sudo ./install-local.sh

# Start service
sudo systemctl start nanohat-oled.service
```

> **Note**: Package installation steps are commented out in `install-local.sh` by default for faster testing. Uncomment them if this is a fresh system.

## Service Management

```bash
# Start service
sudo systemctl start nanohat-oled.service

# Stop service
sudo systemctl stop nanohat-oled.service

# Restart service
sudo systemctl restart nanohat-oled.service

# Check status
systemctl status nanohat-oled.service

# Enable on boot
sudo systemctl enable nanohat-oled.service

# Disable on boot
sudo systemctl disable nanohat-oled.service

# View live logs
journalctl -fu nanohat-oled.service

# View daemon debug log
tail -f /tmp/nanohat-oled.log

# View Python script log
tail -f /tmp/nanoled-python.log
```

## Hardware Buttons

The NanoHat OLED has three buttons:

- **K1** (GPIO 0): Default - Previous page in display rotation
- **K2** (GPIO 2): Default - Next page in display rotation  
- **K3** (GPIO 3): Default - Toggle through display modes

Button mappings can be customized in `BakeBit/Software/Python/bakebit_nanohat_oled.py`.

## File Structure

```
NanoHatOLED/
├── install-local.sh          # Main installation script (recommended)
├── install-compat.sh          # Legacy compatible installer
├── install.sh                 # Standard installer
├── NanoHatOLED                # Compiled daemon binary (after install)
├── README.md                  # This file
├── Source/
│   ├── main.c                 # Main daemon code
│   ├── daemonize.c           # Daemonization utilities
│   └── daemonize.h           # Header file with configuration
├── BakeBit/
│   ├── WiringNP/             # GPIO library (patched for GCC 12+)
│   └── Software/Python/
│       ├── bakebit_nanohat_oled.py    # Main OLED display script
│       └── bakebit.py                  # BakeBit library
└── test_*.c                   # Test utilities
```

## Troubleshooting

### Service Won't Start / Timeout Errors

```bash
# Clean up completely
sudo systemctl stop nanohat-oled.service
sudo systemctl reset-failed nanohat-oled.service
sudo killall -9 NanoHatOLED python3
sudo pkill -9 -f "bakebit_nanohat_oled.py"
sudo rm -f /var/run/nanohat-oled.pid

# Reinstall
sudo ./install-local.sh
sudo systemctl start nanohat-oled.service
```

### Multiple Zombie Processes

The service uses `Type=simple` with `KillMode=control-group` to prevent zombie processes. If you still see them:

```bash
# Force cleanup
sudo systemctl kill nanohat-oled.service
sudo systemctl reset-failed nanohat-oled.service
sudo killall -9 NanoHatOLED python3
```

### I²C Not Working

```bash
# Check I²C devices
ls -l /dev/i2c-*

# Test I²C detection
i2cdetect -y 0

# Enable I²C if needed (add to /boot/config.txt or equivalent)
# Then reboot
```

### Display Not Updating

```bash
# Check Python script is running
ps aux | grep bakebit_nanohat_oled.py

# View Python errors
tail -f /tmp/nanoled-python.log

# Test display manually
cd ~/NanoHatOLED/BakeBit/Software/Python
sudo python3 bakebit_nanohat_oled.py
```

### Compilation Errors

If you get GCC errors about implicit declarations or void return types:

```bash
# The installer automatically patches WiringNP
# But if needed, clean and rebuild:
cd ~/NanoHatOLED/BakeBit/WiringNP
sudo ./build clean
cd ~/NanoHatOLED
sudo ./install-local.sh
```

## What Changed From Original

This fork includes fixes for modern Debian systems:

1. **GCC 12+ Compatibility**: Fixed void function return statements in WiringNP
2. **Pillow 10+ Support**: Updated image handling for newer Pillow API
3. **Systemd Type=simple**: Changed from `Type=forking` to prevent PID tracking issues
4. **Signal Handling**: Proper SIGTERM handling for clean shutdown
5. **Process Cleanup**: Added cleanup handlers to kill child processes on exit
6. **No Daemonization**: Removed double-fork when running under systemd
7. **Python Auto-detection**: Automatically finds and uses correct Python 3 version

## Configuration

Edit `Source/daemonize.h` to customize:

```c
#define DEBUG           1                      // Enable debug logging
#define LOG_FILE_NAME   "/tmp/nanohat-oled.log"  // Debug log location
#define LOCKFILE        "/var/run/nanohat-oled.pid"  // PID file
#define PYTHON3_INTERP  "python3.13"           // Python interpreter name
#define PYTHON3_SCRIPT  "bakebit_nanohat_oled.py"  // Display script
```

After changes, recompile:

```bash
cd ~/NanoHatOLED
sudo ./install-local.sh
```

## Development

```bash
# Build just the daemon
gcc Source/daemonize.c Source/main.c -lrt -lpthread -o NanoHatOLED

# Test daemon manually (run as root)
sudo ./NanoHatOLED

# Check logs
tail -f /tmp/nanohat-oled.log
tail -f /tmp/nanoled-python.log
```

## Credits

- Original NanoHat OLED: [FriendlyELEC](http://wiki.friendlyarm.com/wiki/index.php/NanoHat_OLED)
- BakeBit Library: FriendlyELEC
- WiringNP: FriendlyELEC
- Debian Trixie/Bookworm fixes: Community contributors

## License

MIT License - See individual source files for details.
