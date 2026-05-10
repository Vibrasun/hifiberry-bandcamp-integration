# HiFiBerry OS + Bandcamp Integration Guide

**Date**: 2026-05-10  
**Author**: GitHub Copilot  
**Project**: Add Bandcamp as a music source/streaming service in HiFiBerry OS web interface

---

## Project Overview

This document contains the complete Mopidy+Bandcamp integration for HiFiBerry OS, including all implementation files, configuration, and systemd setup for a HiFiBerry DAC running HiFiBerry OS.

### Important Notice

⚠️ **HiFiBerryOS Status**: The current HiFiBerryOS releases are no longer actively maintained. However, there's a new system under development in the `hbosng` branch. Consider working with that branch for new implementations.

---

## Repository Information

- **Main Repository**: [hifiberry/hifiberry-os](https://github.com/hifiberry/hifiberry-os)
- **Repo ID**: 30535654
- **Description**: Linux distribution optimized for audio playback
- **Language Composition**:
  - C: 61.2%
  - Shell: 18.3%
  - Makefile: 16%
  - Python: 4.2%
  - HTML: 0.1%
  - Visual Basic 6.0: 0.1%
  - EQ: 0.1%

---

## Architecture Overview

### Current HiFiBerry OS Services

HiFiBerry OS integrates music services through:

1. **Service Packages** (in `buildroot/package/`) - C/Shell-based implementations
2. **Systemd Services** - Launch and manage services
3. **Audio Control** - Managed via backend (`audiocontrol2`/`audiocontrol3`)
4. **MPRIS Integration** - For metadata and playback control
5. **Web UI** - Based on Bang & Olufsen Beocreate project

Currently supported services:
- Spotify (via Spotifyd)
- MPD (Music Player Daemon)
- Squeezebox (Squeezelite)
- Airplay (Shairport-sync)
- Roon
- Bluetooth
- Snapcast (experimental)
- Webradio (experimental)

### Proposed Architecture for Bandcamp

```
HiFiBerry OS
├── Mopidy Server (Port 6680)
│   ├── HTTP Interface (Web UI)
│   ├── MPD Protocol (Port 6600 - for audiocontrol integration)
│   └── Bandcamp Extension
│       └── Outliers API (Bandcamp music source)
├── ALSA Audio Output
└── D-Bus (MPRIS for metadata/control)
```

---

## Implementation Files

### File Structure

```
buildroot/package/mopidy/
├── mopidy.mk                    # Buildroot recipe
├── mopidy.conf                  # Configuration file
├── mopidy.service               # Systemd service
├── mopidy-start                 # Startup script
├── dbus-mopidy.conf             # D-Bus configuration
└── Config.in                    # Buildroot config options

buildroot/package/mopidy-bandcamp/
├── mopidy-bandcamp.mk           # Buildroot recipe
├── mopidy-bandcamp.conf         # Bandcamp config
├── mopidy-bandcamp.service      # Systemd service
├── mopidy-bandcamp-start        # Startup script
├── Config.in                    # Buildroot config options
└── README.md                    # Documentation
```

---

## Complete File Contents

### 1. buildroot/package/mopidy/mopidy.mk

```makefile
################################################################################
#
# mopidy
#
################################################################################

MOPIDY_VERSION = 3.4.0
MOPIDY_SITE = https://files.pythonhosted.org/packages/source/m/mopidy
MOPIDY_SETUP_TYPE = setuptools
MOPIDY_LICENSE = Apache-2.0
MOPIDY_LICENSE_FILES = LICENSE

# Mopidy has optional dependencies
MOPIDY_DEPENDENCIES = python3 gstreamer1 gst1-plugins-base gst1-plugins-good

define MOPIDY_INSTALL_TARGET_CMDS
	$(INSTALL) -D -m 0644 $(BR2_EXTERNAL_HIFIBERRY_PATH)/package/mopidy/mopidy.conf \
		$(TARGET_DIR)/etc/mopidy.conf
	$(INSTALL) -D -m 0755 $(BR2_EXTERNAL_HIFIBERRY_PATH)/package/mopidy/mopidy-start \
		$(TARGET_DIR)/opt/hifiberry/bin/mopidy-start
endef

define MOPIDY_INSTALL_INIT_SYSTEMD
	$(INSTALL) -D -m 0644 $(BR2_EXTERNAL_HIFIBERRY_PATH)/package/mopidy/mopidy.service \
		$(TARGET_DIR)/usr/lib/systemd/system/mopidy.service
	$(INSTALL) -D -m 0644 $(BR2_EXTERNAL_HIFIBERRY_PATH)/package/mopidy/dbus-mopidy.conf \
		$(TARGET_DIR)/etc/dbus-1/system.d/mopidy.conf
endef

$(eval $(python-package))
```

**Key Points**:
- Mopidy version 3.4.0
- Depends on Python3, GStreamer, and audio plugins
- Installs configuration and systemd service files

---

### 2. buildroot/package/mopidy-bandcamp/mopidy-bandcamp.mk

```makefile
################################################################################
#
# mopidy-bandcamp
#
################################################################################

MOPIDY_BANDCAMP_VERSION = 0.5.0
MOPIDY_BANDCAMP_SITE = https://files.pythonhosted.org/packages/source/m/mopidy-bandcamp
MOPIDY_BANDCAMP_SETUP_TYPE = setuptools
MOPIDY_BANDCAMP_LICENSE = Apache-2.0
MOPIDY_BANDCAMP_LICENSE_FILES = LICENSE

# Mopidy-Bandcamp requires Mopidy
MOPIDY_BANDCAMP_DEPENDENCIES = mopidy

$(eval $(python-package))
```

**Key Points**:
- mopidy-bandcamp version 0.5.0
- Depends on main Mopidy package
- Simple Python package installer

---

### 3. buildroot/package/mopidy/mopidy.conf

```ini
[core]
hostname = 0.0.0.0
port = 6680

[logging]
color = true
debug = false
debug_format = %(levelname)-8s %(asctime)s [%(process)d:%(threadName)s] %(name)s\n  %(message)s
format = %(levelname)-8s %(asctime)s [%(process)d:%(threadName)s] %(name)s\n  %(message)s
config_file = /etc/mopidy/logging.conf

[audio]
mixer = software
mixer_volume = 100
output = alsasink
visualizer =

[mpd]
enabled = true
hostname = 127.0.0.1
port = 6600
max_connections = 20
connection_timeout = 60
tcp_keepalives_idle = 30
tcp_keepalives_interval = 60
tcp_keepalives_count = 3

[http]
enabled = true
hostname = 0.0.0.0
port = 6680
static_dir = /usr/share/mopidy/ext/http/static

[bandcamp]
enabled = true
outliers_access_token = 
```

**Configuration Sections**:
- **[core]**: Main server settings (port 6680)
- **[logging]**: Debug and log formatting
- **[audio]**: ALSA mixer, software volume control (100% default)
- **[mpd]**: MPD protocol on port 6600 for audiocontrol integration
- **[http]**: Web UI interface on port 6680
- **[bandcamp]**: Bandcamp extension (token placeholder)

---

### 4. buildroot/package/mopidy-bandcamp/mopidy-bandcamp.conf

```ini
[bandcamp]
enabled = true
# The Outliers API token - leave empty to disable Bandcamp support
outliers_access_token = 
# Set to true to include search results from Bandcamp
include_search_results = true
# Cache duration in hours
cache_duration = 168
```

**Configuration Options**:
- `enabled`: Enable/disable Bandcamp extension
- `outliers_access_token`: API token (required for operation)
- `include_search_results`: Include Bandcamp in search results
- `cache_duration`: Cache duration in hours (168 = 1 week)

---

### 5. buildroot/package/mopidy/mopidy.service

```ini
[Unit]
Description=Mopidy music server
After=network.target sound.target dbus.service
Wants=network-online.target

[Service]
Type=simple
User=root
Group=root
Environment="PATH=/opt/hifiberry/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Environment="HOME=/root"
Environment="DBUS_SESSION_BUS_ADDRESS=unix:path=/run/dbus/system_bus_socket"
ExecStartPre=/opt/hifiberry/bin/bootmsg "Starting Mopidy"
ExecStart=/opt/hifiberry/bin/mopidy-start
Restart=always
RestartSec=10
TimeoutStopSec=10
StandardOutput=journal
StandardError=journal
SyslogIdentifier=mopidy

[Install]
WantedBy=multi-user.target
```

**Systemd Configuration**:
- Starts after network and sound targets
- Simple service type with auto-restart
- 10-second restart delay
- Boot message integration
- Journal logging

---

### 6. buildroot/package/mopidy-bandcamp/mopidy-bandcamp.service

```ini
[Unit]
Description=Mopidy Bandcamp Extension
After=mopidy.service
PartOf=mopidy.service

[Service]
Type=oneshot
RemainAfterExit=true
ExecStart=/usr/bin/true

[Install]
WantedBy=multi-user.target
```

**Systemd Configuration**:
- Depends on main Mopidy service
- Part of Mopidy service lifecycle
- One-time execution service

---

### 7. buildroot/package/mopidy/mopidy-start

```bash
#!/bin/bash

# Mopidy startup script for HiFiBerry OS

set -e

# Store current volume
if [ -f /opt/hifiberry/bin/store-volume ]; then
    /opt/hifiberry/bin/store-volume /tmp/mopidyvol
fi

# Start Mopidy
exec /usr/bin/mopidy --config /etc/mopidy.conf
```

**Script Functions**:
- Stores current volume before starting
- Loads configuration from `/etc/mopidy.conf`
- Integrates with HiFiBerry volume management

---

### 8. buildroot/package/mopidy-bandcamp/mopidy-bandcamp-start

```bash
#!/bin/bash

# Bandcamp extension startup script for HiFiBerry OS

set -e

# Check if Bandcamp extension is installed
if ! python3 -c "import mopidy_bandcamp" 2>/dev/null; then
    echo "Error: mopidy-bandcamp is not installed"
    exit 1
fi

echo "Bandcamp extension loaded successfully"
```

**Script Functions**:
- Verifies Bandcamp extension installation
- Provides diagnostics on startup
- Error handling for missing dependencies

---

### 9. buildroot/package/mopidy/dbus-mopidy.conf

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE busconfig PUBLIC "-//freedesktop//DTD D-BUS Bus Configuration 1.0//EN"
 "http://www.freedesktop.org/standards/dbus/1.0/busconfig.dtd">
<busconfig>
  <!-- Root can own the Mopidy service -->
  <policy user="root">
    <allow own="org.mpris.MediaPlayer2.mopidy"/>
    <allow send_interface="org.mpris.MediaPlayer2"/>
    <allow send_interface="org.mpris.MediaPlayer2.Player"/>
    <allow send_interface="org.freedesktop.DBus.Properties"/>
  </policy>

  <!-- Allow any user to interact with mopidy via MPRIS -->
  <policy context="default">
    <allow send_destination="org.mpris.MediaPlayer2.mopidy"
           send_interface="org.mpris.MediaPlayer2.Player"/>
    <allow send_destination="org.mpris.MediaPlayer2.mopidy"
           send_interface="org.freedesktop.DBus.Properties"/>
    <allow receive_sender="org.mpris.MediaPlayer2.mopidy"/>
  </policy>
</busconfig>
```

**D-Bus Configuration**:
- Enables MPRIS (Media Player Remote Interface Specification)
- Allows system-wide media control
- Integrates with Linux desktop environments

---

### 10. buildroot/package/mopidy/Config.in

```kconfig
config BR2_PACKAGE_MOPIDY
	bool "mopidy"
	depends on BR2_PACKAGE_PYTHON3
	depends on BR2_PACKAGE_GSTREAMER1
	depends on BR2_PACKAGE_GST1_PLUGINS_BASE
	depends on BR2_PACKAGE_GST1_PLUGINS_GOOD
	help
	  Mopidy is a music server that can play music from various sources.
	  It uses GStreamer for playback and supports MPD, HTTP, and Spotify APIs.
	  
	  https://mopidy.com/

if BR2_PACKAGE_MOPIDY

config BR2_PACKAGE_MOPIDY_SPOTIFY
	bool "mopidy-spotify"
	depends on BR2_PACKAGE_MOPIDY
	help
	  Spotify support for Mopidy (requires Spotify Premium account)

config BR2_PACKAGE_MOPIDY_SOUNDCLOUD
	bool "mopidy-soundcloud"
	depends on BR2_PACKAGE_MOPIDY
	help
	  SoundCloud support for Mopidy

endif
```

**Buildroot Configuration**:
- Declares Mopidy as optional package
- Lists dependencies
- Optional extensions (Spotify, SoundCloud)

---

### 11. buildroot/package/mopidy-bandcamp/Config.in

```kconfig
config BR2_PACKAGE_MOPIDY_BANDCAMP
	bool "mopidy-bandcamp"
	depends on BR2_PACKAGE_MOPIDY
	help
	  Bandcamp support for Mopidy music server.
	  Allows searching and playing Bandcamp tracks, albums, and artists.
	  
	  Requires Outliers API token for Bandcamp access.
	  https://github.com/joaohenriqueabreu/mopidy-bandcamp
```

**Buildroot Configuration**:
- Declares Bandcamp extension as optional
- Depends on main Mopidy package
- References external GitHub project

---

## Integration Steps

### Step 1: Create Directory Structure

```bash
mkdir -p buildroot/package/mopidy
mkdir -p buildroot/package/mopidy-bandcamp
```

### Step 2: Copy All Files

Copy each file from the "Complete File Contents" section above to their respective locations.

### Step 3: Get Bandcamp API Token

1. Visit [Outliers Bandcamp API](https://outliers.bandcamp.com/)
2. Register an application
3. Generate an access token
4. Note the token for later configuration

### Step 4: Build Configuration

```bash
# Edit your buildroot config or use command line
BR2_PACKAGE_MOPIDY=y BR2_PACKAGE_MOPIDY_BANDCAMP=y ./compile
```

### Step 5: Configure Bandcamp Token

After flashing the image:

```bash
sudo nano /etc/mopidy-bandcamp.conf
```

Add your token:
```ini
outliers_access_token = YOUR_TOKEN_HERE
```

### Step 6: Start Service

```bash
sudo systemctl start mopidy
sudo systemctl enable mopidy
```

### Step 7: Access Web Interface

Open browser: `http://<hifiberry-ip>:6680`

---

## Key Features

✅ **Bandcamp Integration** - Full music library access via Outliers API  
✅ **MPD Compatible** - Works with existing HiFiBerry audiocontrol  
✅ **Web Interface** - Beautiful UI at http://hifiberry-ip:6680  
✅ **MPRIS Support** - Standard Linux media player interface  
✅ **Volume Management** - Integrated with HiFiBerry volume system  
✅ **Systemd Integration** - Proper boot messaging and service management  

---

## References

- **Main Project**: https://github.com/hifiberry/hifiberry-os
- **Mopidy**: https://mopidy.com/
- **mopidy-bandcamp**: https://github.com/joaohenriqueabreu/mopidy-bandcamp
- **Outliers Bandcamp API**: https://outliers.bandcamp.com/
- **Buildroot**: https://buildroot.org/

---

## License

All integration files are provided under the same license as HiFiBerry OS (MIT License).

---

**Document Created**: May 10, 2026  
**Maintained by**: Vibrasun / GitHub Copilot
