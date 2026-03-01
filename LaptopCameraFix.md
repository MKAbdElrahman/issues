# Intel IPU6 Laptop Camera Fix (Ubuntu 24.04 + Kernel 6.17)

## Problem

On Dell laptops (and similar) with Intel Meteor Lake CPUs, the built-in MIPI camera (OV02E10 sensor) has three issues out of the box:

1. **Camera not visible to apps** — Browsers, video calls, and GNOME apps can't find the camera.
2. **Dozens of ghost "ipu6" video sources** — PipeWire exposes all 48 raw IPU6 ISYS capture nodes, cluttering device lists.
3. **Camera LED always on** — The default `ipu6-camera-relay` service streams from the sensor 24/7, even when no app is using the camera.

## Root Causes

| Issue | Cause |
|-------|-------|
| Camera invisible | WirePlumber probes the v4l2loopback device before the GStreamer writer connects, caching it as "Video Output" instead of "Video Capture" |
| Ghost sources | PipeWire enumerates every `/dev/video1`–`/dev/video48` (raw IPU6 ISYS nodes) as separate video sources |
| LED always on | `ipu6-camera-relay.service` runs `gst-launch-1.0 icamerasrc` non-stop; the proper on-demand `v4l2-relayd` service was failing due to a wrong format config and insufficient v4l2loopback buffers |

## Fix (Step by Step)

### Step 1: Fix v4l2-relayd config (wrong video format)

The default config uses `YUY2`, but `icamerasrc` only supports `NV12`.

```bash
sudo sed -i 's/^FORMAT=YUY2$/FORMAT=NV12/' /etc/default/v4l2-relayd
```

Verify:
```bash
cat /etc/default/v4l2-relayd
# Should show FORMAT=NV12
```

### Step 2: Fix v4l2loopback buffer allocation

The default `max_buffers=2` is too low and causes `v4l2sink` to fail. Update the modprobe config:

```bash
sudo bash -c 'echo "options v4l2loopback devices=1 exclusive_caps=1 max_buffers=8 card_label=\"Intel MIPI Camera\"" > /etc/modprobe.d/v4l2-relayd.conf'
```

Reload the module (requires nothing to be using `/dev/video0`):

```bash
sudo rmmod v4l2loopback && sudo modprobe v4l2loopback
```

> **Note:** If `rmmod` says "Module is in use", close any app using the camera (browser, etc.) and try again. Check with `sudo fuser /dev/video0`.

### Step 3: Switch from always-on relay to on-demand relay

Stop and disable the always-on service (this is what keeps the LED on 24/7):

```bash
sudo systemctl stop ipu6-camera-relay.service
sudo systemctl disable ipu6-camera-relay.service
```

Enable and start the on-demand `v4l2-relayd` service:

```bash
sudo systemctl reset-failed v4l2-relayd@default.service
sudo systemctl enable v4l2-relayd@default.service
sudo systemctl start v4l2-relayd@default.service
```

Verify it's running:

```bash
systemctl status v4l2-relayd@default.service
# Should show "active (running)"
```

### Step 4: Hide ghost IPU6 sources from PipeWire

Create a WirePlumber rule to disable all raw IPU6 devices:

```bash
mkdir -p ~/.config/wireplumber/main.lua.d
```

```bash
cat > ~/.config/wireplumber/main.lua.d/51-hide-ipu6-raw-devices.lua << 'EOF'
-- Hide all raw Intel IPU6 ISYS capture devices from PipeWire.
-- Only the v4l2loopback "Intel MIPI Camera" (/dev/video0) should be visible.

v4l2_monitor.rules = v4l2_monitor.rules or {}

table.insert(v4l2_monitor.rules, {
  matches = {
    {
      { "device.name", "matches", "v4l2_device.pci-0000_00_05.0*" },
    },
  },
  apply_properties = {
    ["device.disabled"] = true,
  },
})

table.insert(v4l2_monitor.rules, {
  matches = {
    {
      { "node.name", "matches", "v4l2_input.pci-0000_00_05.0*" },
    },
  },
  apply_properties = {
    ["node.disabled"] = true,
  },
})
EOF
```

### Step 5: Restart WirePlumber

```bash
systemctl --user restart wireplumber
```

### Step 6: Verify everything works

**Camera should be the only video source:**
```bash
pw-dump 2>/dev/null | python3 -c "
import json, sys
for obj in json.load(sys.stdin):
    props = obj.get('info',{}).get('props',{})
    mc = props.get('media.class','')
    if 'Video' in mc:
        nd = props.get('node.description', props.get('device.description',''))
        print(f'  {obj[\"id\"]:>4} | {mc:20} | {nd}')
"
```

Expected output (only 2 entries):
```
    60 | Video/Device         | Intel MIPI Camera
    58 | Video/Source         | Intel MIPI Camera (V4L2)
```

**Camera LED should be OFF** (no app is using it).

**Test capture:**
```bash
ffplay -f v4l2 -video_size 1280x720 -i /dev/video0
# LED turns ON, you see video. Close ffplay → LED turns OFF.
```

## Summary of Changed Files

| File | Change |
|------|--------|
| `/etc/default/v4l2-relayd` | `FORMAT=YUY2` → `FORMAT=NV12` |
| `/etc/modprobe.d/v4l2-relayd.conf` | Added `max_buffers=8` |
| `~/.config/wireplumber/main.lua.d/51-hide-ipu6-raw-devices.lua` | New file — hides raw IPU6 nodes |
| `ipu6-camera-relay.service` | Disabled (was always-on) |
| `v4l2-relayd@default.service` | Enabled (on-demand) |

## Hardware Info

- **Sensor:** OV02E10 (OVTI02E1) via MIPI CSI-2
- **ISP:** Intel IPU6 (PCI 0000:00:05.0)
- **Privacy controller:** Intel IVSC (Visual Sensing Controller)
- **Relay:** v4l2-relayd → v4l2loopback (`/dev/video0`)
- **Tested on:** Ubuntu 24.04.4 LTS, kernel 6.17.0-14-generic
