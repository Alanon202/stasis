<p align="center">
  <img src="assets/stasis.png" alt="Stasis Logo" width="200"/>
</p>

<h1 align="center">Stasis</h1>

<p align="center">
  <i>A modern Wayland idle manager designed for simplicity and effectiveness.</i>
</p>

<p align="center">
  <a href="#features">Features</a> •
  <a href="#installation">Installation</a> •
  <a href="#configuration">Configuration</a> •
  <a href="#usage">Usage</a> •
  <a href="#contributing">Contributing</a>
</p>

## Features

- **🧠 Smart idle detection** with configurable timeouts
- **🎵 Media-aware idle handling** - automatically detects media playback
- **🚫 Application-specific inhibitors** - prevent idle when specific apps are running
- **⏸️ Idle inhibitor respect** - honors Wayland idle inhibitor protocols
- **⚙️ Flexible action system** - supports named action blocks and custom commands
- **🔍 Regex pattern matching** - powerful app filtering with regular expressions
- **📝 Clean configuration** - uses the intuitive [RUNE](https://github.com/saltnpepper97/rune-cfg) configuration language
- **🔧 CLI options** - verbose logging, version info, and live config reloading
- **⚡ Live reload** - update configuration without restarting the daemon

## Compositor Support

Stasis uses each compositor's native IPC protocol for app inhibiting functionality.

| Compositor | app_inhibit Support | Status |
|------------|-------------------|--------|
| Niri | ✅ | Tested & Working |
| River | ✅ | Implemented (see below) |
| Hyprland | ✅ | Implemented |
| Others | ❌ | Send a PR! |

### River Support Notes

- **Limited window enumeration:** Unlike Niri or Hyprland, River does not provide a full IPC interface to list all windows. Stasis cannot reliably enumerate every active application via River.
- **Fallback mechanism:** When using River, Stasis falls back to process-based detection (sysinfo) for app inhibition.
- **Regex and app names may differ:** Because process-based detection relies on executable names and paths, some regex patterns or app IDs from Niri/Hyprland may not match exactly. Users may need to adjust inhibit_apps patterns for River.
- **Logging:** Stasis will log which apps were detected for inhibition, helping users refine their patterns.

> **Tip:** For best results with River, include both exact executable names and regex patterns for applications you want to inhibit.

**Want to add support for your compositor?** We welcome pull requests! Stasis integrates with each compositor's native IPC protocol, so adding support typically involves implementing the specific IPC calls for window/app detection.

## Installation

### Arch Linux (AUR)

Available on the AUR as `stasis`.

Using `yay`:
```bash
yay -S stasis
```

Using `paru`:
```bash
paru -S stasis
```

### From Source

```bash
git clone https://github.com/saltnpepper97/stasis
cd stasis
cargo build --release --locked
sudo install -Dm755 target/release/stasis /usr/local/bin/stasis
```

Or install to your local bin directory:
```bash
install -Dm755 target/release/stasis ~/.local/bin/stasis
```

## Usage

### Command Line Options

```bash
# Run normally
stasis

# Specify custom config path
stasis -c "/path/to/file"

# Show version information
stasis --version

# Enable verbose logging
stasis --verbose
```

### Subcommands

```bash
# Reload stasis config
stasis reload-config

# Stop currently running instance
stasis stop

# Pause timers
stasis pause

# Resume timers
stasis resume

# Trigger all idle timers
stasis trigger-idle

# Trigger pre-suspend-command
statis trigger-pre_suspend
```

### Running Manually
```bash
stasis
```

### Live Configuration Reload
While Stasis is running, you can reload the configuration without restarting:
```bash
stasis reload-config
```

### Systemd Service (Recommended)

Create `~/.config/systemd/user/stasis.service`:

```ini
[Unit]
Description=Stasis Wayland Idle Manager
After=graphical-session.target

[Service]
ExecStart=%h/.local/bin/stasis
# Use /usr/local/bin/stasis if installed system-wide
Restart=always
RestartSec=5

[Install]
WantedBy=default.target
```

Enable and start the service:
```bash
systemctl --user enable stasis.service
systemctl --user start stasis.service
```

Check service status:
```bash
systemctl --user status stasis.service
```

## Configuration

Stasis uses a configuration file written in **[RUNE](https://github.com/saltnpepper97/rune-cfg)**, a simple but effective configuration language designed for clarity and ease of use.

### Configuration File Location

Create your config file at:
```
~/.config/stasis/stasis.rune
```

The default config can be found at:
```
/usr/share/doc/stasis/stasis.rune
```

To get started quickly, copy the default config:
```bash
mkdir -p ~/.config/stasis
cp /usr/share/doc/stasis/stasis.rune ~/.config/stasis/stasis.rune
```

### Example Configuration

Create your configuration at `~/.config/stasis/stasis.rune`:

```rune
@author "Dustin Pilgrim"
@description "Stasis configuration file"

app_default_timeout 300

idle:
  resume_command "systemctl resume-sessions"
  pre_suspend_command "notify-send 'System suspending in 5 seconds' && sleep 5"
  monitor_media true
  respect_idle_inhibitors true

  inhibit_apps [
    "vlc"
    "Spotify"
    "mpv"
    r".*\.exe"
    r"steam_app_.*"
    r"firefox.*"
  ]

  # Desktop-only idle actions
  lock_screen:
    timeout = 300
    command "brightnessctl set 10% && sleep 0.2 && swaylock"
  end

  suspend:
    timeout 1800
    command "systemctl suspend"
  end

  dpms:
    timeout 330
    command "niri msg action power-off-monitors"
  end

  # Laptop-only AC actions
  on_ac:

    lock_screen:
      command "brightnessctl set 20% && sleep 0.2 && swaylock"
      timeout 300
    end

    dpms:
      command "niri msg action power-off-monitors"
      timeout 300
    end

    brightness:
      command "brightnessctl set 100%"
      timeout 0
    end

    suspend:
      command "systemctl suspend"
      timeout 1800
    end

    custom:
      command "notify-send 'AC mode custom action'"
      timeout 300
    end
  end

  # Laptop-only battery actions
  on_battery:

    lock_screen:
      command "brightnessctl set 10% && sleep 0.2 && swaylock"
      timeout 300
    end

    dpms:
      command "niri msg action power-off-monitors"
      timeout 300
    end

    brightness:
      command "brightnessctl set 30%"
      timeout 0
    end

    suspend:
      command "systemctl suspend"
      timeout 1200
    end

    custom:
      command "notify-send 'Battery mode custom action'"
      timeout 300
    end
  end
end
```

> **📖 For complete configuration details, see:** `man 5 stasis`

### Configuration Options

#### Global Settings
- `app_default_timeout` - Default timeout in seconds for actions

#### Idle Block Configuration
- `resume_command` - Command to run when resuming from idle
- `pre_suspend_command` - Command to run before system suspend operations
- `monitor_media` - Automatically detect media playbook to prevent idle
- `respect_idle_inhibitors` - Honor Wayland idle inhibitor protocols  
- `inhibit_apps` - Array of application names/patterns to prevent idle

#### Named Action Blocks

**⚠️ Important Configuration Notes as of v0.1.2:**

1. **Variable naming flexibility:** You can use either dashes (`-`) or underscores (`_`) in variable names, but **be consistent within each individual variable name**:
   ```rune
   # ✅ Good - consistent within each variable name
   app_default_timeout 300
   monitor-media true
   lock_screen:
     # ...
   
   # ✅ Also good - using underscores consistently per variable
   app_default_timeout 300
   monitor_media true
   lock_screen:
     # ...
   
   # ❌ Bad - mixing within the same variable name
   app_default-timeout 300  # Don't mix _ and - in same variable!
   monitor_media-setting true  # Don't mix _ and - in same variable!
   ```

2. **Named action blocks are fixed:** As of version 0.1.2, the named action blocks listed below are **set in stone**. You must use these exact names for built-in functionality - they cannot be customized or renamed.

##### Built-in Action Blocks (Fixed Names as of v0.1.2)

**Lock Screen** (`lock_screen` or `lock-screen`)
- `timeout` - Time in seconds before locking (can reference variables)
- `command` - Command to execute for screen locking

**Suspend** (`suspend`)
- `timeout` - Time in seconds before suspending
- `command` - Command to execute for system suspend

**DPMS** (`dpms`)
- `timeout` - Time in seconds before turning off displays
- `command` - Command to execute for display power management

**Brightness** (`brightness`)
- `timeout` - Time in seconds before dimming screen
- `command` - Command to execute for brightness control

##### Custom Action Blocks

You can define any number of custom action blocks with unique names (avoid using the reserved names above):

```rune
idle:
  hibernate:
    timeout 3600  # 1 hour
    command "systemctl hibernate"
  end
  
  shutdown:
    timeout 7200  # 2 hours
    command "systemctl poweroff"
  end
  
  notify_user:
    timeout 600   # 10 minutes
    command "notify-send 'Long idle detected'"
  end
  
  backup_data:
    timeout 1200  # 20 minutes
    command "/home/user/scripts/backup.sh"
  end
end
```

### App Inhibitor Patterns

The `inhibit_apps` array supports both literal app names and regex patterns using raw string syntax:

```rune
inhibit_apps [
  "spotify"              # Exact match
  "mpv"                  # Exact match
  r".*\.exe"             # Regex: any .exe file
  r"steam_app_\d+.*"     # Regex: Steam applications with digits
  r"firefox.*"           # Regex: Firefox and variants
  r"^chrome.*"           # Regex: Chrome browsers (starts with 'chrome')
  r".*[Vv]ideo.*"        # Regex: apps containing 'Video' or 'video'
]
```

**Regex Pattern Guidelines:**
- Use raw string syntax: `r"pattern"` for all regex patterns
- Escape special characters properly: `\.` for literal dots, `\d+` for digits
- Use `.*` for wildcard matching
- Use `^` for start-of-string and `$` for end-of-string anchors
- Test your patterns with verbose logging to ensure they match correctly

### Pre-Suspend Commands

The `pre_suspend_command` option allows you to run a command before any suspend operation occurs. This is useful for:
- Saving work or state before suspend
- Showing notifications to the user
- Gracefully closing applications
- Adding delays for user interaction

```rune
idle:
  pre_suspend_command "notify-send 'Suspending in 10 seconds...' && sleep 10"
  
  suspend:
    timeout 1800
    command "systemctl suspend"
  end
end
```

## About RUNE

Stasis uses **[RUNE](https://github.com/saltnpepper97/rune-cfg)**, a configuration language designed to be simple but effective. RUNE features:

- **Clean, readable syntax** - Easy to write and understand
- **Variable support** - Define and reference variables
- **Nested configuration blocks** - Organize complex configurations
- **Array and string literals** - Flexible data types
- **Raw string support** - Use `r"pattern"` syntax for regex patterns
- **Comments** - Document your config with `#`
- **Type safety and validation** - Catch errors early
- **Metadata** - Use `@` symbol to denote metadata

## Troubleshooting

### Common Issues

**Stasis not detecting apps:**
- Ensure your compositor is supported (see [Compositor Support](#compositor-support))
- Check that the app names in `inhibit_apps` match the actual application names
- Use `stasis --verbose` to see detailed logging of app detection

**Regex patterns not matching:**
- Ensure you're using raw string syntax: `r"pattern"`
- Test patterns with verbose logging to see what apps are detected
- Remember that River uses process-based detection which may have different app names

**Service not starting:**
- Verify the `ExecStart` path in your systemd service file
- Check service logs: `journalctl --user -u stasis.service`

**Configuration not reloading:**
- Use `stasis reload-config` to send reload signal to running daemon
- Check configuration syntax if reload fails

**Configuration errors:**
- Validate your RUNE syntax - ensure consistent use of dashes or underscores
- Verify you're using the correct built-in action block names (they are fixed as of v0.1.2)
- Check the manual: `man 5 stasis`
- Use verbose logging to identify configuration issues
 
## Contributing

We welcome contributions! Here's how you can help:

- 🐛 **Report bugs** by opening an issue
- 💡 **Request features** with detailed use cases  
- 🔧 **Submit pull requests** for bug fixes or new features
- 📦 **Package for your distro** and let us know so we can link to it
- 📖 **Improve documentation** or add examples

### Adding Compositor Support

Want to add support for your favorite compositor? We'd love your help! Adding support typically involves:

1. Implementing the compositor's native IPC protocol
2. Adding window/app detection functionality
3. Testing with real applications

Check out the existing implementations in the source code for reference.

## License

[MIT License](LICENSE)

---

*keeping your Wayland session in perfect balance between active and idle.*
