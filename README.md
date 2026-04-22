# Razer Blade 16 Audio Fix

Complete audio fix for Razer Blade 16 (2023/2024) on Linux. Fixes speakers, automatic headphone switching (3.5mm jack, USB & BT device support), audio hissing, and game audio static.

## Problems Solved

| Issue | Symptom | Fix |
|-------|---------|-----|
| Silent speakers | Internal speakers produce no sound | ALC298 codec initialization via hda-verb |
| No auto-switching | Plugging headphones doesn't switch audio | Virtual sink + switching daemon |
| Audio hissing | Periodic hiss/crackle on 3.5mm output | Disable HDA power management |
| Paused playback | Media pauses when switching devices | Seamless stream redirection |
| Game audio static | Crackling/static in games at 44100Hz | Multi-rate PipeWire + high-quality resampling |

## Supported Hardware

- **Razer Blade 16 (2023)** - RZ09-0483
- **Razer Blade 16 (2024)** - RZ09-0510

Other Razer laptops with ALC298 codec may also work.

## Requirements

- Linux with PipeWire and WirePlumber
- systemd
- `alsa-tools` package (installed automatically)
- sof-firmware

Tested on Arch Linux, should work on Fedora, Ubuntu 24.04+, and other modern distros.

# Modifications from original:
Original code [litesung/razer-blade-16-audio-fix](https://github.com/Litesung/razer-blade-16-audio-fix)

Tested and working on Omarchy 3.5.0 (Arch Linux).

Removed a redundant function call `SPEAKER_SCRIPT` on line 88, causing the script to crash.

And replaced:

```bash
su - "$REAL_USER" -c "systemctl
```

with:

```bash
sudo -u "$REAL_USER" XDG_RUNTIME_DIR=$XDG_RUNTIME_DIR systemctl
```

`RB16_2024_enable_internal_speaker.sh` is a standalone script for quick fix for silent speakers issue (used in testing).
`razer-blade-audio-daemon` is a standalone script for audio device switching (used in testing).

## Quick Install

```bash
# Download and run
curl -fsSL https://raw.githubusercontent.com/litesung/razer-blade-16-audio-fix/main/install.sh | sudo bash
```

Or clone and run:

```bash
git clone https://github.com/litesung/razer-blade-16-audio-fix.git
cd razer-blade-16-audio-fix
sudo ./install.sh
```

## What Gets Installed

| Component | Location | Purpose |
|-----------|----------|---------|
| Speaker fix script | `/usr/local/bin/razer-blade-speaker-fix.sh` | Initializes ALC298 codec |
| Boot service | `/etc/systemd/system/razer-blade-speaker-fix.service` | Runs fix at boot |
| Resume service | `/etc/systemd/system/razer-blade-speaker-fix-resume.service` | Runs fix after suspend |
| Virtual sink | `~/.config/pipewire/pipewire.conf.d/10-virtual-output.conf` | PipeWire Main-Output sink |
| Clock rate config | `~/.config/pipewire/pipewire.conf.d/10-clock-rate.conf` | 44100Hz/48000Hz support + resampling quality |
| Audio daemon | `~/.local/bin/razer-blade-audio-daemon` | Auto-switches devices |
| Daemon service | `~/.config/systemd/user/` | User service for daemon |
| WirePlumber config | `~/.config/wireplumber/wireplumber.conf.d/` | Routing settings |
| Udev rules | `/etc/udev/rules.d/99-razer-blade-audio.rules` | Disable power management |
| Modprobe config | `/etc/modprobe.d/razer-blade-audio.conf` | Disable power saving |
| Sudoers rule | `/etc/sudoers.d/razer-blade-audio` | Allow daemon hda-verb access |

## How It Works

### Speaker Fix

The ALC298 codec in Razer Blade 16 requires ~2000 register writes to initialize properly. The Linux driver doesn't do this automatically, so speakers are silent out of the box. This fix sends the necessary `hda-verb` commands at boot and after resume.

### Hybrid Audio Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              Hybrid Audio Architecture                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐            │
│  │ Firefox │  │ Spotify │  │  Game   │  │  App N  │            │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘            │
│       │            │            │            │                  │
│       └────────────┴─────┬──────┴────────────┘                  │
│                          │                                      │
│              ┌───────────┴───────────┐                          │
│              │   Default Sink        │                          │
│              │  (set by daemon)      │                          │
│              └───────────┬───────────┘                          │
│                          │                                      │
│         ┌────────────────┴────────────────┐                     │
│         │                                 │                     │
│         ▼                                 ▼                     │
│  ┌─────────────────────┐          ┌─────────────────┐           │
│  │    Main-Output      │          │   Bluetooth     │           │
│  │   (Virtual Sink)    │          │   (Direct)      │           │
│  │                     │          │                 │           │
│  │  For: Speakers &    │          │  Apps connect   │           │
│  │       3.5mm Jack    │          │  directly to    │           │
│  └──────────┬──────────┘          │  BT sink        │           │
│             │                     └─────────────────┘           │
│       pw-link                                                   │
│             │                                                   │
│      ┌──────┴──────┐                                            │
│      ▼             ▼                                            │
│  ┌────────┐  ┌──────────┐                                       │
│  │Speaker │  │Headphones│  Hardware sinks at 100%               │
│  │ (100%) │  │  (100%)  │  Volume controlled via Main-Output    │
│  └────────┘  └──────────┘                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why hybrid?** Bluetooth works fine natively. The virtual sink solves the HDA profile switching problem for speakers/headphones only.

### Seamless Device Switching

```
┌─────────────────────────────────────────────────────────────────┐
│              Seamless Device Switching Flow                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CASE 1: Speaker ↔ Headphones (via Virtual Sink)                │
│  ───────────────────────────────────────────────                │
│                                                                 │
│  App ──► Main-Output ─────────────────────────► (continuous)    │
│              │                                                  │
│              │ pw-link (reconnects instantly)                   │
│              ▼                                                  │
│         ┌─────────┐        ┌─────────┐                          │
│         │ Speaker │ ─────► │Headphone│                          │
│         └─────────┘        └─────────┘                          │
│              │                  │                               │
│         [unlink]           [link]                               │
│                                                                 │
│  App never knows device changed - stream stays on Main-Output   │
│                                                                 │
│  ═══════════════════════════════════════════════════════════    │
│                                                                 │
│  CASE 2: Any ↔ Bluetooth (Direct Switch)                        │
│  ───────────────────────────────────────                        │
│                                                                 │
│  App ──► Main-Output                App ──► Bluetooth           │
│              │          pactl           │                       │
│              │      move-sink-input     │                       │
│              └──────────────────────────┘                       │
│                                                                 │
│  Daemon moves active streams seamlessly - no rebuffering        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Auto-Switching Daemon

```
┌─────────────────────────────────────────────────────────────────┐
│              Daemon - "Last Connected Wins"                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    ┌────────────────┐                           │
│                    │  Poll every 1s │ ◄─────────────────────┐   │
│                    └───────┬────────┘                       │   │
│                            │                                │   │
│             ┌──────────────┴──────────────┐                 │   │
│             ▼                             ▼                 │   │
│  ┌────────────────────┐       ┌────────────────────┐        │   │
│  │ hda-verb: check    │       │ pactl: check for   │        │   │
│  │ jack pin state     │       │ bluez_output sink  │        │   │
│  └──────────┬─────────┘       └──────────┬─────────┘        │   │
│             │                            │                  │   │
│      ┌──────┴──────┐              ┌──────┴──────┐           │   │
│      │             │              │             │           │   │
│   CHANGED      unchanged      CHANGED      unchanged        │   │
│      │             │              │             │           │   │
│      ▼             │              ▼             │           │   │
│  ┌────────┐        │         ┌────────┐        │           │   │
│  │ Switch │        │         │ Switch │        │           │   │
│  └────┬───┘        │         └────┬───┘        │           │   │
│       │            │              │            │            │   │
│       └────────────┴──────────────┴────────────┘            │   │
│                            │                                │   │
│                            └────────────────────────────────┘   │
│                                                                 │
│  ═══════════════════════════════════════════════════════════    │
│                                                                 │
│  JACK PLUGGED IN:                                               │
│    1. Set Main-Output as default                                │
│    2. Move streams to Main-Output                               │
│    3. Switch HDA profile to Headphones                          │
│    4. pw-link Main-Output → Headphone sink                      │
│    5. Set hardware volume to 100%                               │
│                                                                 │
│  JACK UNPLUGGED (was using headphones):                         │
│    → Fall back: BT connected? → BT, else → Speakers             │
│                                                                 │
│  BLUETOOTH CONNECTED:                                           │
│    1. Set BT sink as default (DIRECT - no virtual sink)         │
│    2. Move streams to BT sink                                   │
│                                                                 │
│  BLUETOOTH DISCONNECTED (was using BT):                         │
│    → Fall back: Jack plugged? → Headphones, else → Speakers     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Behavior**: Last connected device always wins (not priority-based)

## Verify Installation

```bash
# Check services
systemctl status razer-blade-speaker-fix.service
systemctl --user status razer-blade-audio-daemon

# Check virtual sink
pactl get-default-sink  # Should show "Main-Output"

# Check speaker fix log
cat /tmp/razer-blade-speaker-fix.log

# Check daemon logs
journalctl --user -u razer-blade-audio-daemon -f
```

## Uninstall

```bash
sudo ./install.sh --uninstall
```

Or if you used curl:

```bash
curl -fsSL https://raw.githubusercontent.com/litesung/razer-blade-16-audio-fix/main/install.sh | sudo bash -s -- --uninstall
```

## Troubleshooting

### Speakers still silent

```bash
# Run fix manually
sudo /usr/local/bin/razer-blade-speaker-fix.sh

# Check log
cat /tmp/razer-blade-speaker-fix.log
```

### Auto-switching not working

```bash
# Check daemon status
systemctl --user status razer-blade-audio-daemon

# Restart daemon
systemctl --user restart razer-blade-audio-daemon

# Check jack detection manually
sudo hda-verb /dev/snd/hwC1D0 0x21 GET_PIN_SENSE 0
# 0x80000000 = plugged, 0x0 = unplugged
```

### Wrong audio card number

The script auto-detects the card number, but if you have multiple audio devices:

```bash
# List audio devices
aplay -l

# Check which card has ALC298
cat /proc/asound/cards
```

## Credits

- Speaker fix based on research from [Arch Linux Forums](https://bbs.archlinux.org/viewtopic.php?id=285121)
- HDA verb sequences derived from Windows driver analysis

## License

MIT License - see [LICENSE](LICENSE)
